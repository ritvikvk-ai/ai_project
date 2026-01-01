# Finetuning ROS2 RAG Assistant

Retrieval-augmented generation (RAG) system built for ROS2, Nav2, MoveIt2, and Gazebo questions. The stack extracts content from the ROS 2 docs repo, stores it in MongoDB, generates spaCy embeddings, indexes them in Qdrant, and routes user queries to Llama-3-70B on Together via a Gradio interface. A fine-tuned Llama-3-8B variant trained on `data.json` is also published on Hugging Face: https://huggingface.co/Data-harjai/ai_project_fine_tuned_llama.

Project by Ritvik Vasantha Kumar (rv2459) and Ansh Harjai (ah7163).

## What this solves
- Pulls authoritative ROS 2 docs and code snippets into a searchable knowledge base.
- Lets robotics engineers ask natural-language questions and get answers grounded in retrieved source material.
- Reduces model hallucinations by pairing dense retrieval (Qdrant) with a strong generator (Llama 3).

## How it works
- **ETL** (`app/etl_pipeline.py`): Clones https://github.com/ros2/ros2_documentation, strips markup, and writes Markdown/reST/text plus code files into MongoDB (`RAG_DB.raw_data`) with metadata (file name, repo, URL). Logged via ClearML.
- **Featurization** (`app/featurization_pipeline.py`): Chunks each document into ~200-word windows, embeds with `en_core_web_lg` (300-dim), and upserts into Qdrant collection `rag_vectors` with payload containing the chunk text and file metadata.
- **Retrieval** (`app/retrieve_from_qdrant.py`): Converts a user query to a spaCy embedding and returns the top 5 matching chunks from Qdrant.
- **Generation** (`app/llm_connection.py`): Builds a system/user prompt and calls `meta-llama/Llama-3-70b-chat-hf` on Together. Responses are shown in a Gradio UI (`app/app.py`). The dropdown option “Initialize Database” runs the ETL and featurization pipelines before answering questions.
- **Fine-tuning**: `data.json` plus `fine-tune.ipynb` show how Llama-3-8B-Instruct was fine-tuned; inference for that model is available via Hugging Face (link above).

## Why these choices
- **ROS 2 docs as the corpus**: High-quality, domain-authoritative material for Nav2/MoveIt2/Gazebo questions; keeps answers aligned with upstream guidance.
- **MongoDB for raw storage**: Handles mixed file types (text + code) with flexible schemas; easy upserts during repeated ETL runs.
- **spaCy `en_core_web_lg` embeddings**: Fast, CPU-friendly 300-d vectors that avoid GPU dependency for indexing; good enough semantic fidelity for docs-style queries.
- **Qdrant for vector search**: Lightweight, self-hosted ANN store with payload filtering and simple Docker deployment; matches the project’s need for ~hundreds of thousands of chunks.
- **Llama-3-70B on Together**: Offloads heavy inference to a hosted model for higher answer quality than the locally fine-tuned 8B while keeping local hardware requirements modest.
- **Gradio UI**: Minimal code to expose chat plus an “Initialize Database” control for end-to-end demos.
- **ClearML tracking**: Optional experiment/run logging to keep ETL/featurization reproducible and observable.
- **Fine-tuned 8B checkpoint**: Provides a smaller, cheaper fallback model trained on `data.json`; useful for offline/Ollama-style deployments when Together is unavailable.

## Repository layout
- `app/app.py` – Gradio chat UI and orchestration hook for ETL, featurization, retrieval, and LLM calls.
- `app/etl_pipeline.py` – GitHub clone/clean/load into MongoDB with ClearML tracking.
- `app/featurization_pipeline.py` – spaCy embedding generation and Qdrant upload.
- `app/retrieve_from_qdrant.py` – Query embedding + nearest-neighbor search in Qdrant.
- `app/llm_connection.py` – Together API call (set your API key in `together_ai_api`).
- `app/Dockerfile`, `docker-compose.yml` – Containerization and service wiring for the app, MongoDB, Qdrant, and ClearML.
- `data.json` – Supervised QA data used for fine-tuning.
- `Embedding_demo.ipynb`, `Extraction_demo.ipynb`, `llm_connection.ipynb`, `Demo_LLM.mov`, `Screenshots/` – Demos and assets showing the workflow end to end.

## Prerequisites
- Docker and Docker Compose (or Python 3.9+ if running locally).
- Together API key (set `together_ai_api` in `app/llm_connection.py` before building/running).
- ClearML credentials if you want experiment tracking (`CLEARML_API_ACCESS_KEY`, `CLEARML_API_SECRET_KEY`, `CLEARML_API_HOST`).
- Ports available: MongoDB 27017, Qdrant 6333, Gradio 7860.
- If running without Docker, install dependencies from `app/requirements.txt` and download the spaCy model: `python -m spacy download en_core_web_lg`.

## Quickstart (Docker)
1. Add your Together API key to `app/llm_connection.py` (`together_ai_api = "..."`). Optional: fill ClearML keys in `clearmmll.txt` or pass them as env vars at run time.
2. Build the app image (includes the spaCy model): `docker build -f app/Dockerfile -t ros2-rag-app ./app`.
3. Create a network and start dependencies:
   - `docker network create my_network`
   - `docker run -d --name mongodb --network my_network -p 27017:27017 mongo:5.0`
   - `docker run -d --name qdrant --network my_network -p 6333:6333 qdrant/qdrant:v1.3.0`
4. Run the app container on the same network:  
   `docker run --rm --name rag-app --network my_network -p 7860:7860 ros2-rag-app`
5. Open http://localhost:7860, choose “Initialize Database” to run ETL + featurization, then ask questions or use the sample prompts.

> Note: The included `docker-compose.yml` starts the same services; update the app service port mapping to `7860:7860` if you prefer Compose.

## Local development (without Docker)
1. `cd app && python -m venv .venv && source .venv/bin/activate`
2. `pip install -r requirements.txt && python -m spacy download en_core_web_lg`
3. Set `together_ai_api` in `llm_connection.py`, ensure MongoDB and Qdrant are running locally (default ports), then start the app: `python app.py`.

## Using the notebooks and assets
- `fine-tune.ipynb` – Colab-ready notebook to train Llama-3-8B-Instruct on `data.json` and push to Hugging Face.
- `Embedding_demo.ipynb`, `Extraction_demo.ipynb`, `llm_connection.ipynb` – Individual steps of the pipeline for debugging or teaching.
- `Demo_LLM.mov` and `Screenshots/` – UI walk-through of the Gradio chat flow.

## What happens when you click “Initialize Database”
1. Clone and clean `ros2/ros2_documentation` into MongoDB (`RAG_DB.raw_data`).
2. Chunk + embed each document with spaCy and upload to Qdrant collection `rag_vectors`.
3. Subsequent queries embed the question, retrieve the top 5 chunks, build a prompt, and call Llama-3-70B on Together for the final answer.
