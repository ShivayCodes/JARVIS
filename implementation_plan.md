# Implementation Plan — Fully Local Self-Learning AI Engine

We will remove the dependencies on external LLM APIs (OpenAI/Anthropic) and replace them with a completely local, self-contained AI algorithm. This engine will ingest data from a dedicated dataset folder and use custom NLP algorithms for semantic understanding, response generation, and continuous learning.

## Proposed Changes

### 1. Dataset Folder Structure
- Create a folder `dataset_pool/` in the project root. This is where the AI will monitor and automatically load JSON, XML, and TXT files.
- Sub-folders:
  - `dataset_pool/faq/` — FAQ documents and Q&A pairs
  - `dataset_pool/documentation/` — Technical documentation
  - `dataset_pool/knowledge/` — General knowledge base
  - `dataset_pool/training/` — Training data for Markov models

### 2. Custom AI Algorithm

#### [NEW] `algorithm.py`
Core semantic understanding engine with the following components:

**TF-IDF & Cosine Similarity Module:**
- Implement **TF-IDF Vectorizer** from scratch (using only Python standard library / math)
- Calculate document-query similarity using cosine distance
- Support multi-document retrieval with ranking
- Cache computed TF-IDF vectors for performance optimization

**Word-Association & Markov Synthesis Model:**
- Build **n-gram frequency tables** (bigrams, trigrams) from the dataset
- Generate contextual sentence completions using Markov chain synthesis
- Support custom seed words for directed response generation
- Fallback mechanism: if direct match fails, use synthesis model

**Weight-Reinforcement Learning:**
- Maintain response path confidence weights in SQLite
- `+1` feedback increases probability weight of successful paths
- `-1` feedback decreases probability and flags for retraining
- Decay mechanism: old weights decay over time to allow model evolution
- Dynamic recalibration: re-weight paths based on feedback ratio (success/total attempts)

**Advanced Features:**
- Contextual memory: track conversation history for coherent multi-turn responses
- Entity recognition: extract named entities (names, dates, locations) from queries
- Semantic chunking: break large documents into retrievable semantic units
- Confidence scoring: assign confidence levels to generated responses

#### [NEW] `semantic_matcher.py`
- **Fuzzy string matching** for typo-tolerant query handling
- **Semantic similarity** calculation using word embeddings (pre-computed static vectors)
- **Query expansion**: automatically expand queries with synonyms from a static thesaurus
- **Relevance filtering**: threshold-based filtering of low-confidence matches

#### [NEW] `memory_manager.py`
- **SQLite-backed persistent memory** for learned responses
- Schema:
  - `responses` table: (query_hash, response, confidence, last_used, feedback_count)
  - `entities` table: (entity_type, entity_value, context)
  - `conversation_history` table: (session_id, timestamp, user_msg, bot_response, feedback)
- **Memory compaction**: periodically clean up low-confidence responses
- **Session management**: maintain conversation context with unique session IDs

### 3. Ingestion & Dataset Loading

#### [MODIFY] `dataset_learner.py`
Extended data ingestion with multiple format support:

**File Format Support:**
- JSON: structured FAQ format, nested Q&A pairs, metadata inclusion
- XML: hierarchical knowledge structure with schema validation
- TXT: plain text files, markdown support, heading-based chunking
- CSV: tabular Q&A data (columns: question, answer, category, confidence)

**Ingestion Pipeline:**
- Auto-scan `dataset_pool/` on startup for new/updated files
- **File watcher**: monitor folder for runtime dataset additions
- **Deduplication**: detect and merge duplicate Q&A pairs
- **Metadata extraction**: extract tags, categories, and source attribution
- **Version tracking**: record dataset versions for incremental updates
- **Validation**: schema validation and content sanitization

**Data Processing:**
- Chunk large documents into semantic units
- Normalize text (lowercase, remove special chars, tokenization)
- Extract key phrases for indexing
- Build vocabulary and frequency maps

#### [NEW] `dataset_validator.py`
- **Schema validation** for structured datasets
- **Quality checks**: detect incomplete Q&A pairs, malformed data
- **Conflict detection**: identify and report contradictory information
- **Coverage analysis**: report gaps in knowledge base

### 4. Engine & Planner Routing

#### [MODIFY] `engine.py`
Core execution engine refactored for local operation:

- Remove `LLMClient` and all external API dependencies
- Initialize **LocalAIEngine** from `algorithm.py`
- **Query router**: directs queries to appropriate handler:
  - Direct match → retrieval module
  - No match → synthesis module
  - Ambiguous → clarification prompts
- **Response pipeline**:
  1. Parse and normalize query
  2. Search semantic index
  3. Apply entity recognition
  4. Generate response (retrieval or synthesis)
  5. Apply confidence filtering
  6. Store in memory with initial weight
- **Error handling**: graceful fallbacks for unknown queries

#### [NEW] `query_processor.py`
- **Intent classification**: categorize user intent (question, command, clarification)
- **Query normalization**: standardize input format
- **Abbreviation expansion**: handle common acronyms
- **Negation handling**: understand "not", "don't", "never"
- **Multi-part query**: decompose complex queries into sub-queries

#### [DELETE] `llm.py`
- Remove OpenAI/Anthropic wrapper entirely

#### [MODIFY] `planner.py`
Adapt task decomposition for local algorithm:

- Replace LLM-based planning with rule-based task decomposition
- **Workflow rules**: if-then rules for common task patterns
- **State machine**: track task progress and dependencies
- **Feedback integration**: adjust planning based on response feedback
- **Priority system**: handle urgent/important tasks first

#### [NEW] `feedback_handler.py`
- **Feedback collection**: API endpoints for user ratings/corrections
- **Feedback processing**: analyze and weight user input
- **Batch learning**: aggregate feedback for periodic model updates
- **A/B testing**: compare response variants with feedback scores

### 5. Configuration & Setup

#### [MODIFY] `config.py`
- Remove external API configurations (OpenAI keys, Anthropic credentials)
- Add local paths:
  - `DATASET_POOL_DIR = os.path.join(_root_dir, "dataset_pool")`
  - `MEMORY_DB_PATH = os.path.join(_root_dir, "data", "memory.db")`
  - `CACHE_DIR = os.path.join(_root_dir, "cache")`
- Add algorithm tuning parameters:
  - `TF_IDF_MIN_DF = 1` (minimum document frequency)
  - `TF_IDF_MAX_DF = 0.95` (maximum document frequency)
  - `SIMILARITY_THRESHOLD = 0.5` (minimum relevance score)
  - `MARKOV_ORDER = 2` (n-gram size)
  - `WEIGHT_DECAY_RATE = 0.95` (feedback weight decay)
  - `CONFIDENCE_MIN_THRESHOLD = 0.3` (minimum response confidence)
- Add performance settings:
  - `VECTOR_CACHE_SIZE = 10000`
  - `BATCH_FEEDBACK_SIZE = 100`
  - `MEMORY_COMPACTION_INTERVAL = 7` (days)

#### [NEW] `environment_setup.py`
- Auto-create dataset folder structure on first run
- Initialize SQLite memory database
- Load and cache static resources (thesaurus, entity gazetteers)
- Validate all configuration parameters

### 6. Testing & Validation

#### [NEW] `test_local_ai.py`
Comprehensive test suite:

**Data Loading Tests:**
- Test JSON, XML, TXT, CSV loading from `dataset_pool`
- Validate file watcher detects new datasets
- Verify deduplication logic
- Test error handling for malformed files

**Algorithm Tests:**
- TF-IDF vectorization correctness
- Cosine similarity calculations
- Top-k retrieval ranking accuracy
- Markov synthesis output quality

**Integration Tests:**
- End-to-end query → response pipeline
- Multi-turn conversation coherence
- Feedback learning and weight updates
- Session persistence across restarts

**Performance Tests:**
- Query latency benchmarks (target: <100ms for cached queries)
- Memory footprint tracking
- Large dataset handling (1M+ Q&A pairs)
- Concurrent query handling

#### [NEW] `test_semantic_matching.py`
- Fuzzy match accuracy
- Synonym expansion correctness
- Entity recognition precision/recall
- Query normalization consistency

#### [NEW] `test_reinforcement_learning.py`
- Weight update calculations
- Feedback decay mechanics
- Confidence scoring consistency
- Path selection bias with learned weights

#### [NEW] `benchmark_suite.py`
- Compare response quality against baseline
- Track improvement metrics over time
- Memory efficiency analysis
- Inference speed profiling

### 7. Advanced Features

#### [NEW] `conversation_manager.py`
- **Session tracking**: unique session IDs for user conversations
- **Context window**: maintain last N messages for coherence
- **Pronoun resolution**: track referents ("it", "that", etc.)
- **Topic persistence**: remember conversation topic across turns
- **Conversation analytics**: log and analyze conversation patterns

#### [NEW] `entity_extractor.py`
- Named entity recognition (NER) for locations, persons, organizations, dates
- Custom entity dictionary support
- Entity linking to knowledge base
- Context-aware extraction

#### [NEW] `response_ranker.py`
- Multi-factor ranking of candidate responses:
  - Semantic similarity score
  - Learned weight from feedback
  - Recency bias (recent responses ranked higher)
  - Diversity factor (avoid repetition)
- Ensemble ranking combining multiple factors

#### [NEW] `cache_manager.py`
- **LRU cache** for frequent queries and computed vectors
- **Persistent cache**: serialize cache to disk for startup speedup
- **Cache invalidation**: automatic invalidation on dataset updates
- **Cache statistics**: track hit/miss rates for optimization

#### [NEW] `online_learning_engine.py`
- **Incremental learning**: update models without full retraining
- **Stream processing**: handle continuous dataset updates
- **Drift detection**: identify and adapt to knowledge base changes
- **Model versioning**: maintain multiple model versions for rollback

### 8. CLI & User Interfaces

#### [MODIFY] `main.py`
- Add `--init` flag to initialize fresh setup
- Add `--stats` flag to display knowledge base statistics
- Add `--validate` flag to run validation suite
- Add `--benchmark` flag to run performance tests
- Interactive CLI mode improvements:
  - Show confidence scores for responses
  - Allow inline feedback (👍/👎)
  - Display retrieval details on demand
  - Session ID display for context tracking

#### [NEW] `admin_interface.py`
- Dataset management commands (add, remove, list, validate)
- Memory database inspection tools
- Weight/feedback statistics dashboard
- Cache management utilities
- Model evaluation tools

---

## Verification Plan

### Automated Tests
1. **Data Integrity Tests:**
   - Verify JSON/XML/TXT/CSV loading from `dataset_pool/`
   - Validate no data loss during ingestion
   - Check deduplication accuracy

2. **Algorithm Correctness Tests:**
   - TF-IDF semantic retrieval matches correct documents
   - Cosine similarity calculations are accurate
   - Response generation synthesizes coherent text
   - Markov chain produces valid n-grams

3. **Learning Tests:**
   - Reinforcement learning updates weights correctly
   - Positive feedback increases response probability
   - Negative feedback decreases probability
   - Confidence scores reflect accuracy

4. **Integration Tests:**
   - Full query → response pipeline works end-to-end
   - Multi-turn conversations maintain context
   - Session persistence works across restarts
   - File watching detects new datasets

5. **Performance Tests:**
   - Single query latency < 100ms (cached)
   - Batch query throughput > 100 qps
   - Memory footprint < 500MB for 100K Q&A pairs
   - Cache hit rate > 70% for typical workloads

### Manual Verification Steps

1. **Initial Setup:**
   ```bash
   python main.py --init
   python main.py --stats
   ```

2. **Dataset Loading:**
   - Place sample JSON/XML/TXT files in `dataset_pool/`
   - Verify folder auto-scan on startup
   - Check database records with `python main.py --inspect`

3. **Interactive Testing:**
   ```bash
   python main.py --cli
   # Try queries, provide feedback (👍/👎)
   # Verify responses improve with feedback
   ```

4. **Validation Runs:**
   ```bash
   python main.py --validate  # Data quality checks
   python main.py --benchmark # Performance benchmarks
   ```

5. **Offline Verification:**
   - Ensure no API calls are made (monitor network)
   - Verify offline operation without API keys
   - Check all responses are deterministic

### Success Criteria

- ✅ System runs completely offline without any API keys
- ✅ Ingests 100K+ Q&A pairs with <2s startup time
- ✅ Responds to queries within 100ms (cached)
- ✅ Achieves >80% user satisfaction with feedback system
- ✅ Learns and improves with user feedback over time
- ✅ Maintains conversation context across multiple turns
- ✅ Gracefully handles unknown queries with helpful fallbacks
- ✅ Persistent memory preserves learning across sessions

---

## Directory Structure

```
JARVIS/
├── jarvis_x/
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py (MODIFIED)
│   │   ├── engine.py (MODIFIED)
│   │   ├── algorithm.py (NEW)
│   │   ├── semantic_matcher.py (NEW)
│   │   ├── memory_manager.py (NEW)
│   │   ├── cache_manager.py (NEW)
│   │   └── llm.py (DELETE)
│   ├── learning/
│   │   ├── __init__.py
│   │   ├── dataset_learner.py (MODIFIED)
│   │   ├── dataset_validator.py (NEW)
│   │   ├── query_processor.py (NEW)
│   │   ├── feedback_handler.py (NEW)
│   │   ├── online_learning_engine.py (NEW)
│   │   └── entity_extractor.py (NEW)
│   ├── reasoning/
│   │   ├── __init__.py
│   │   ├── planner.py (MODIFIED)
│   │   └── response_ranker.py (NEW)
│   ├── conversation/
│   │   ├── __init__.py
│   │   └── conversation_manager.py (NEW)
│   ├── admin/
│   │   ├── __init__.py
│   │   └── admin_interface.py (NEW)
│   └── main.py (MODIFIED)
├── tests/
│   ├── __init__.py
│   ├── test_local_ai.py (NEW)
│   ├── test_semantic_matching.py (NEW)
│   ├── test_reinforcement_learning.py (NEW)
│   └── benchmark_suite.py (NEW)
├── dataset_pool/
│   ├── faq/
│   ├── documentation/
│   ├── knowledge/
│   └── training/
├── data/
│   └── memory.db (created at runtime)
├── cache/
│   └── (vector cache files, created at runtime)
├── config.yaml
├── main.py (MODIFIED)
├── requirements.txt
└── README.md

```

---

## Implementation Priority

### Phase 1: Core Engine (Weeks 1-2)
- [ ] Implement `algorithm.py` (TF-IDF, cosine similarity)
- [ ] Implement `semantic_matcher.py`
- [ ] Implement `memory_manager.py` with SQLite schema
- [ ] Update `engine.py` to use local AI engine

### Phase 2: Data Ingestion (Weeks 2-3)
- [ ] Enhance `dataset_learner.py` for JSON/XML/TXT/CSV
- [ ] Create `dataset_validator.py`
- [ ] Implement file watcher for `dataset_pool/`
- [ ] Test with sample datasets

### Phase 3: Learning & Feedback (Weeks 3-4)
- [ ] Implement `feedback_handler.py`
- [ ] Add reinforcement learning weights
- [ ] Create `response_ranker.py`
- [ ] Build conversation tracking in `memory_manager.py`

### Phase 4: Advanced Features (Weeks 4-5)
- [ ] Implement `entity_extractor.py`
- [ ] Build `conversation_manager.py`
- [ ] Add `online_learning_engine.py`
- [ ] Create `cache_manager.py`

### Phase 5: Testing & Optimization (Weeks 5-6)
- [ ] Write comprehensive test suites
- [ ] Run benchmark suite
- [ ] Performance optimization
- [ ] Documentation

### Phase 6: Deployment & Polish (Week 6+)
- [ ] CLI improvements
- [ ] Admin interface
- [ ] Final validation
- [ ] Release preparation

---

## Dependencies

Core dependencies (Python stdlib only, plus minimal external):
- `sqlite3` (stdlib) — persistent memory
- `json`, `xml.etree` (stdlib) — data parsing
- `math`, `collections` (stdlib) — algorithm implementation
- `re` (stdlib) — regex/text processing
- `pathlib`, `os` (stdlib) — file operations
- `datetime` (stdlib) — timestamps
- `unittest`, `pytest` — testing (optional)

Optional for enhanced features:
- `numpy` — faster vector math (optional optimization)
- `nltk` or `spacy` — NLP enhancements (optional)

**No external LLM APIs or closed-source dependencies.**

---

## Notes

- All algorithms use Python standard library to maximize portability and minimize dependencies
- Fully offline operation — no internet required
- Single SQLite database provides persistent learning across sessions
- Modular design allows easy addition of new ingestion formats and learning algorithms
- Extensive testing ensures reliability and enables safe iteration
