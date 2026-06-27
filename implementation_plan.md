# Implementation Plan — Fully Local Self-Learning AI Engine

We will remove the dependencies on external LLM APIs (OpenAI/Anthropic) and replace them with a completely local, self-contained AI algorithm. This engine will ingest data from a dedicated dataset folder, index it using a custom TF-IDF and Cosine Similarity model (built from scratch), synthesize responses using a word-association Markov-style chain, and optimize itself based on user feedback.

## Proposed Changes

### 1. Dataset Folder
- Create a folder [dataset_pool](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/dataset_pool) in the project root. This is where the AI will monitor and automatically load JSON, XML, HTML, and TXT files to learn from scratch.

### 2. Custom AI Algorithm
#### [NEW] [algorithm.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/core/algorithm.py)
- Implement a custom **TF-IDF Vectorizer & Cosine Similarity matcher** from scratch (using only Python standard library / math) for semantic retrieval of facts, Q&A, and documentation.
- Implement a **Word-Association & Markov Synthesis model** to generate/complete sentences when a direct match is not found.
- Implement a weight-reinforcement mechanism that raises the probability of paths that receive positive feedback (+1) and lowers paths that receive negative feedback (-1).

### 3. Ingestion Upgrades
#### [MODIFY] [dataset_learner.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/learning/dataset_learner.py)
- Extend `DatasetLearner` to support JSON datasets (both structured FAQ formats and general text lists).
- Add support for plain text (`.txt`) files.
- Automatically scan the `dataset_pool` folder upon startup to ingest new knowledge.

### 4. Engine & Planner Routing
#### [MODIFY] [engine.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/core/engine.py)
- Remove `LLMClient` (OpenAI/Anthropic keys/clients).
- Initialize the new local `LocalAIEngine` from `algorithm.py`.
- Route freeform queries (previously routed to Claude) through the custom local AI algorithm.

#### [DELETE] [llm.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/core/llm.py)
- Remove the LLM wrapper file.

#### [MODIFY] [planner.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/reasoning/planner.py)
- Adapt the task decomposition planner to work with the local algorithm's decisions and rule triggers.

### 5. Config clean up
#### [MODIFY] [config.py](file:///c:/Users/Shubham%20Varma/Desktop/JARVIS/JarvisX_System/jarvis_x/core/config.py)
- Remove external API configs and add `DATASET_POOL_DIR = os.path.join(_root_dir, "dataset_pool")`.

---

## Verification Plan

### Automated Tests
- Create a test script `test_local_ai.py` that verifies:
  1. JSON/XML/TXT loading from `dataset_pool`.
  2. TF-IDF semantic retrieval matches the correct documents.
  3. Response generation synthesizes text.
  4. Reinforcement learning updates SQLite memory.

### Manual Verification
- Place sample JSON/XML files in `dataset_pool`.
- Launch `python main.py --cli` and check if it successfully ingests datasets and responds offline without any API keys.
