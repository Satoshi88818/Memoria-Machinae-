Memoria Machinae v1.3

Production-grade episodic memory microservice for embodied and language agents.

Memoria Machinae is a biologically-inspired, graph-backed memory architecture designed to give AI agents — from large language models to physical robots — a persistent, structured, and queryable sense of experience. It is not a vector database wrapper. It is a full memory stack: ingestion pipeline, neuromorphic salience detection, causal graph construction, schema learning, consolidation scheduling, ethics monitoring, and a REST API — all wired together as a microservice.

Table of Contents

What Makes This Unique

Memory System Architecture

v1.3 Features

Memory Types & Lifecycle

Salience & the SNN Pipeline

Consolidation & the Dream Phase

Schema Induction

Multi-Agent / Social Memory

Ethics & Sentience Monitoring

Configuration Reference

Hardware Guidance

Project Structure

Requirements

What Makes This Unique

Most memory systems for AI agents are retrieval systems — they store embeddings and return the top-k nearest neighbours. Memoria Machinae goes significantly further:

CapabilityTypical Vector DBMemoria MachinaeSemantic retrieval✅✅Memory decay over time❌✅ (type-aware rates)Causal edge graph❌✅Schema / procedural learning❌✅ (hierarchical)Precondition extraction❌✅Neuromorphic salience detection❌✅ (SNN)Predictive coding integration❌✅ (online MLP)Active dream/consolidation phase❌✅Multi-agent social memory❌✅Ethics / sentience safeguards❌✅Prometheus observability❌✅Encrypted audit ledger❌✅ 

The system is built around the insight that useful memory is not flat retrieval — it is structured experience: episodic records that decay, cause-and-effect relationships that strengthen, procedural schemas that generalise, and a dream-like consolidation process that prevents important procedural knowledge from fading.

Memory System Architecture

┌──────────────────────────────────────┐ │ REST API / Service │ │ (acquire · recall · feedback · stats)│ └────────────────┬─────────────────────┘ │ ┌────────────────▼─────────────────────┐ │ Ingestion Pipeline │ │ EventFilter → Embedding (SigLIP + │ │ SentenceTransformer) → SNN Salience │ │ → PredictiveCoding → SpikePro file │ └────────────────┬─────────────────────┘ │ ┌─────────────────────────▼──────────────────────────┐ │ Memory Graph │ │ EpisodicNode (episodic · semantic · procedural) │ │ ├── co-occurrence edges (entity overlap) │ │ ├── causal edges (explicit action → outcome) │ │ ├── schema_of edges (node → schema node) │ │ └── composed_of edges (schema → meta_schema) │ └─────────────────────────┬──────────────────────────┘ │ ┌────────────────────────────────▼────────────────────────────────┐ │ Background Stack │ │ Decay Pass → Consolidation (merge · prune · schema) │ │ → Precondition Learning → Active Dream Phase │ └─────────────────────────────────────────────────────────────────┘ 

EpisodicNode

The atomic unit of memory. Each node carries:

text — raw textual description of the experience

embedding_bytes — 512-dim float32 multimodal embedding (stored as bytes, deserialized on access)

entities — named entities extracted by spaCy (en_core_web_trf) with a robotics-domain vocabulary

strength — a float in [0, 2] that decays over time and is boosted on recall

spike_profile — optional neuromorphic salience metadata from the SNN path

memory_type — episodic | semantic | procedural

source_agent — originating agent ID (v1.3)

observation_type — self | observed | communicated (v1.3)

preconditions — entity-frequency dict extracted from causal predecessors (v1.3)

recency_score() — exponential half-life of 7 days

v1.3 Features

1. Hierarchical Schemas

Schema induction now operates in two phases. Flat induction (powered by PatternMiner) produces level-1 procedural schema nodes from repeated action patterns. A second hierarchical composition pass then groups level-1 schemas that share a common action sequence prefix and promotes groups of ≥ schema_compose_min_count (default: 2) into level-2 meta_schema nodes, linked via composed_of edges.

This mirrors how human memory works: individual procedures (e.g. "pick up part", "inspect part") can be abstracted into a higher-order routine ("handle part"). Meta-schema nodes are immune to pruning.

New config keys: schema_hierarchy_levels, schema_compose_min_count
New metric: schema_composed_total

2. Precondition Learning

schema/precondition.py introduces PreconditionLearner, which runs as a post-induction pass during consolidation. For each procedural schema node it:

Walks backward through causal predecessor chains up to precondition_hop_depth (default: 2) hops.

Counts entity co-occurrence frequencies across all predecessor chains.

Retains entities appearing in ≥ precondition_min_freq (default: 60%) of chains as preconditions.

Preconditions are stored directly on EpisodicNode.preconditions as an entity→frequency dict and are surfaced on recall results and via MemoryGraph.get_preconditions(memory_id). This lets downstream planners query "what context was typically present before this action succeeded?"

New config keys: precondition_hop_depth, precondition_min_freq
New metric: precondition_extracted_total

3. Predictive Coding Integration

models/predictive.py adds PredictiveCodingModel: a two-layer online MLP (default: 512→256→512) that learns to predict the next embedding given the current one. It trains with a single Adam gradient step per call, making it lightweight enough to run on the ingestion hot path.

Prediction error as salience: The L2 distance (RMSE) between predicted and actual embeddings is fed into the salience computation as a third signal (weighted by predictive_error_weight, default: 0.3). High prediction error means the current perception was surprising — a natural marker of high salience. The model is per-subject and lazy-loaded from a singleton registry.

salience_composite = ( spike_score * (1 - predictive_error_weight) + prediction_error_normalised * predictive_error_weight ) 

New config keys: use_predictive_coding, predictive_hidden_dim, predictive_error_weight
New metric: prediction_error histogram

4. Multi-Agent / Social Memory

Every EpisodicNode now carries:

source_agent (str, default 'self') — the agent that generated or reported the memory.

observation_type ('self' | 'observed' | 'communicated') — how the memory was acquired: 

self — the agent experienced it directly.

observed — the agent witnessed another agent's action.

communicated — another agent reported the memory verbally/via message.

These fields are propagated through the full pipeline (ingestion → graph → REST endpoints → feedback). MemoryGraph gains get_by_agent() and get_by_observation_type() query methods. RecallEngine.recall() gains an optional source_agent filter.

Social memories (non-self observation types) are tracked separately via the social_memory_ingested_total counter, enabling audit of cross-agent information flow.

New REST fields: source_agent, observation_type on /acquire and /recall

5. Active Recall / Dream Phase

The nightly consolidation loop (stack._dream_phase()) has been upgraded from a passive edge-strengthening pass to an active recall phase.

Selection criteria: Nodes in the bottom strength quartile that have ≥ dream_min_causal_successors (default: 2) outgoing causal edges. These are structurally important memories (many downstream consequences) that are memory-weak (decaying toward the pruning floor). Without intervention they would silently disappear, severing causal chains that procedural schemas depend on.

Active recall: For each selected node, RecallEngine.dream_recall() issues a real recall query using the node's text. If the node appears in the top-k results, its strength is boosted by dream_hit_strength_boost (default: 0.05) and its outgoing causal edges are strengthened. This replicates the biological function of sleep-phase memory replay: rehearsing important but weakly encoded experiences to prevent forgetting.

New config keys: dream_active_recall, dream_min_causal_successors, dream_hit_strength_boost
New metric: dream_replays_total

6. Better Pattern Mining

schema/pattern_miner.py replaces the naive entity-label grouping used in v1.2 with two complementary algorithms that run in parallel before schema promotion:

Graph community detection uses networkx.algorithms.community.greedy_modularity_communities (an approximation of Louvain) on the undirected co-occurrence/causal subgraph to identify structurally dense node clusters. Dense clusters represent recurring contexts — good candidates for schema promotion.

Sequential pattern mining scans temporally ordered node action-entity sequences for frequent singletons and bigrams (PrefixSpan-style), controlled by pattern_min_support (default: 0.3) and pattern_max_length (default: 4). This captures ordered action sequences, not just co-occurrence.

Both algorithm outputs are merged by pattern key (deduplicating by memory_id) before schema promotion. Disable with USE_PATTERN_MINER=false to fall back to v1.2 behaviour.

New config keys: use_pattern_miner, pattern_min_support, pattern_max_length

Memory Types & Lifecycle

Three memory types are supported, each with distinct decay rates and consolidation eligibility:

TypeDecay RateConsolidation EligibleDescriptionepisodic0.001 / pass✅Individual experiences, time-tagged eventssemantic0.0002 / pass❌Factual knowledge, stable world stateprocedural0.00005 / pass❌Action schemas, routines — decay-resistant 

Procedural and semantic memories decay ~5–20× slower than episodic memories by default. Schema nodes (which are procedural) are additionally immune to the pruning pass entirely.

Strength floor: min_strength_floor (default: 0.05). Nodes falling below this — if consolidation-eligible — are pruned on the next consolidation pass.

Salience & the SNN Pipeline

The ingestion pipeline supports two salience detection paths, switchable via USE_SNN:

SNN Path (default, USE_SNN=true)

A Spiking Neural Network (models/spike_transformer.py) processes embeddings across snn_steps (default: 16) timesteps through snn_depth (default: 6) layers with leaky integrate-and-fire dynamics (membrane_decay=0.9, resting_potential=-0.07, spike_threshold=0.5). The resulting SpikeProfile captures:

overall_density — mean spike rate across all layers and timesteps

conservative_score — max of per-layer and per-timestep peak densities (used as the primary salience signal)

per_layer_density — per-layer breakdown

peak_timestep, peak_layer — where activity was highest

high_salience — bool flag if conservative_score ≥ throttle threshold

Lightweight Path (USE_SNN=false)

A novelty-window detector compares the current embedding cosine similarity against the last salience_novelty_window (default: 50) embeddings. Events with similarity below salience_novelty_threshold (default: 0.4) are flagged as salient.

Predictive Coding Signal (v1.3)

When use_predictive_coding=true, the RMSE from the forward model is added as a third salience signal weighted by predictive_error_weight (default: 0.3).

Consolidation & the Dream Phase

The background scheduler runs two types of passes:

Decay Pass (every decay_interval_seconds, default: 300s): Applies type-aware strength decay to all non-deleted nodes.

Consolidation Pass (every consolidation_interval_seconds, default: 3600s):

Merge pass — nodes with embedding cosine similarity ≥ consolidation_merge_threshold (default: 0.92) are merged: embeddings averaged, entity sets unioned, strength taken as the max plus a small boost, and graph edges re-pointed to the surviving node.

Prune pass — consolidation-eligible nodes below the strength floor are deleted.

Schema induction — PatternMiner runs graph community detection and sequential pattern mining; patterns meeting schema_min_pattern_count (default: 3) are promoted to schema nodes; hierarchical composition produces meta-schema nodes.

Precondition learning — PreconditionLearner extracts causal preconditions for all procedural schema nodes.

Active dream phase — structurally important but weakly encoded causal nodes are actively replayed via the recall engine to prevent decay below the pruning floor.

Schema Induction

Schema nodes are procedural-type EpisodicNode instances promoted from repeated patterns. They are connected to their constituent episodic nodes via schema_of edges. They do not decay and are never pruned.

Hierarchy:

episodic_node_A ─┐ episodic_node_B ─┤ schema_of → schema_node_1 ─┐ │ │ composed_of → meta_schema_node episodic_node_C ─┤ schema_of → schema_node_2 ─┘ episodic_node_D ─┘ 

Level-1 schema nodes are promoted from recurring entity/action patterns. Level-2 meta-schema nodes are composed from level-1 schemas that share an action sequence prefix. This supports up to schema_hierarchy_levels (default: 2) levels of abstraction.

Multi-Agent / Social Memory

In multi-robot or multi-agent deployments, agents can ingest memories from other agents by setting source_agent and observation_type on the /acquire endpoint. The recall engine can then filter by agent or by observation type, enabling queries like:

"What did agent_002 observe about Zone B?" → get_by_agent('agent_002') filtered by location entity

"What communicated memories do I have about obstacle avoidance?" → recall(query=..., source_agent=..., observation_type='communicated')

Social memories are tracked separately in Prometheus to support audit requirements.

Ethics & Sentience Monitoring

ethics/sentience.py monitors the SNN spike density as a proxy for internal activity. Four graduated response levels are configured:

LevelDefault ThresholdActionlog0.50Record event to audit ledgeralert0.60Emit Prometheus alertthrottle0.70Reduce ingestion ratehalt0.85Suspend ingestion entirely 

ethics/blackbox.py maintains an encrypted audit ledger (PostgreSQL in production, SQLite for local dev) recording all threshold crossings. The encryption key is loaded strictly from the ENCRYPTION_KEY environment variable — the system raises EnvironmentError on startup if it is absent, deliberately preventing key generation at runtime (a lost key corrupts all stored audit data).

Configuration Reference

All configuration is managed in config.py via the CONFIG dict. Values are overridden by environment variables.

KeyDefaultDescriptiondimension512Embedding dimensionuse_snntrueEnable SNN salience pathuse_cross_encodertrueEnable cross-encoder re-rankinguse_predictive_codingtrueEnable predictive coding salienceuse_pattern_minertrueEnable advanced pattern miningdream_active_recalltrueEnable active dream phaseschema_hierarchy_levels2Max schema abstraction depthschema_compose_min_count2Min schemas to compose into meta-schemaprecondition_hop_depth2Causal predecessor hops for preconditionsprecondition_min_freq0.6Min entity frequency to retain as preconditionpredictive_hidden_dim256Forward model hidden layer widthpredictive_error_weight0.3Prediction error weight in saliencedream_min_causal_successors2Min successors for dream phase selectiondream_hit_strength_boost0.05Strength boost per active dream hitdream_replay_steps20Dream phase iterations per consolidationpattern_min_support0.3Min support for sequential patternspattern_max_length4Max sequential pattern lengthfaiss_index_typehnswhnsw or ivfpq (use ivfpq for >10M memories)consolidation_merge_threshold0.92Cosine similarity threshold for node mergingdecay_interval_seconds300Background decay pass intervalconsolidation_interval_seconds3600Background consolidation pass interval 

Hardware Guidance

HardwareRecommended ConfigJetson Orin NX 16GBUSE_SNN=true, USE_CROSS_ENCODER=true, USE_PREDICTIVE_CODING=true, DREAM_ACTIVE_RECALL=trueRaspberry Pi 5USE_SNN=false, USE_CROSS_ENCODER=false, USE_PREDICTIVE_CODING=false, DREAM_ACTIVE_RECALL=falseIntel NUC companionUSE_SNN=true, USE_CROSS_ENCODER=true, USE_PREDICTIVE_CODING=true, DREAM_ACTIVE_RECALL=true>10M memoriesFAISS_INDEX_TYPE=ivfpq — train index before deployment 

Latency budgets:

Predictive coding adds ~1–5ms per ingestion on CPU (MLP forward + single Adam step). Disable on the hot path for constrained edge devices.

The active dream phase scales linearly with dream_replay_steps × recall_latency — budget ~100ms per consolidation pass on Jetson.

Project Structure

memoria_machinae/ ├── config.py ← Central CONFIG dict + validation ├── metrics.py ← Prometheus counters, histograms, gauges ├── auth.py ← JWT authentication ├── scheduler.py ← APScheduler background tasks ├── stack.py ← Top-level memory stack + dream phase ├── service.py ← FastAPI service layer ├── event_filter.py ← Significance filtering before ingestion ├── feedback.py ← Recall feedback ingestion ├── query_builder.py ← Recall query construction ├── models/ │ ├── manager.py ← Model lifecycle management │ ├── spike_transformer.py ← SNN salience model │ ├── multimodal_fusion.py ← SigLIP + SentenceTransformer fusion │ ├── salience.py ← Composite salience scoring │ └── predictive.py ← [NEW v1.3] Predictive coding forward model ├── memory/ │ ├── store.py ← FAISS index + persistence │ └── graph.py ← EpisodicNode + MemoryGraph (NetworkX) ├── ethics/ │ ├── blackbox.py ← Encrypted audit ledger │ └── sentience.py ← Spike density monitoring + response levels ├── pipeline/ │ └── pipeline.py ← End-to-end ingestion pipeline ├── recall/ │ └── recall.py ← Hybrid recall engine (vector + graph + strength + recency) ├── schema/ │ ├── inducer.py ← Schema induction + hierarchical composition │ ├── precondition.py ← [NEW v1.3] PreconditionLearner │ └── pattern_miner.py ← [NEW v1.3] Community detection + sequential pattern mining ├── tests/ │ ├── test_recall.py │ ├── test_consolidation.py │ ├── test_salience.py │ ├── test_schema.py │ ├── test_precondition.py ← [NEW v1.3] │ └── test_pattern_miner.py ← [NEW v1.3] ├── main.py ← Demo script for all v1.3 features └── requirements.txt 

Requirements

# Core ML torch>=2.2.0 sentence-transformers>=2.7.0 transformers>=4.40.0 Pillow>=10.0.0 scikit-learn>=1.4.0 # NLP (transformer-based NER) spacy>=3.7.0 spacy-transformers>=1.3.0 # After install: python -m spacy download en_core_web_trf # Vector index faiss-cpu>=1.8.0 # or faiss-gpu for CUDA # Service / API fastapi>=0.111.0 uvicorn[standard]>=0.29.0 httpx>=0.27.0 pydantic>=2.7.0 # Auth & encryption PyJWT>=2.8.0 cryptography>=42.0.0 # Async DB asyncpg>=0.29.0 # Background scheduler apscheduler>=3.10.4 # Observability prometheus-client>=0.20.0 # Graph algorithms (community detection) networkx>=3.3.0 # Testing pytest>=8.0.0 pytest-asyncio>=0.23.0 

Environment variables required before first run:

export ENCRYPTION_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())") export JWT_SECRET=$(openssl rand -hex 32) # Optional: export POSTGRES_DSN=postgresql+asyncpg://user:pass@host/db 

