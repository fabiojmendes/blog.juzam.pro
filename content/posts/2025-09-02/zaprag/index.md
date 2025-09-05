+++
title = "Ask your WhatsApp: build a private RAG with LlamaIndex"
draft = true
date = '2025-09-02T20:46:38-04:00'
tags = ["llamaindex", "rag", "whatsapp", "privacy", "ai", "duckdb"]
[cover]
image = "zaprag.svg"
+++

## Why build a WhatsApp RAG?

I have a very active group chat with my friends on WhatsApp. At the time of
writing, it is a bit over half a million messages. Since LLMs became a thing, I
always wondered how I could use this data to something useful—or at the very
least, prank my friends.

Last year I tried a few different approaches to fine tune a model using the chat
data, but I didn't work all that well. Fine tuning a model on commodity hardware
is a challenge in itself and the results were underwhelming. So I dropped that
idea for a while. While going through the material for the
[HuggingFace Agents Course](https://huggingface.co/learn/agents-course/unit2/llama-index/components)
though, it became very clear that RAG (Retrieval Augmented Generation) would be
a perfect fit for what I was trying to do.

This post shows how easy it is to set up a RAG on top of your WhatsApp chat
logs. You are going to export your messages, parse the .txt files, index them
with LlamaIndex, generate embeddings and store them to DuckDB, and ask questions
locally running Ollama. You’ll end with a small chat application that you can
talk and ask questions about your conversation log. The best part is that
everything can run from your local machine, so you don't have to upload any of
this sensitive data to cloud providers.

## Prerequisites

- Python 3.10+
- One or more WhatsApp chat exports in `.txt` format
- [`ollama`](https://ollama.com/download) running a local model (e.g., `llama3`
  or `gpt-oss`)
- [`uv`](https://docs.astral.sh/uv/getting-started/installation/) to manage
  Python dependencies

Create a new project using `uv` and add the dependencies:

```shell
uv init whatsapp-rag
cd whatsapp-rag
uv add \
  llama-index-llms-ollama \
  llama-index-vector-stores-duckdb \
  llama-index-embeddings-huggingface \
  gradio
```

## Export your chats

Create a new directory for the chat data inside your project:

```shell
mkdir input
```

You need to export your chat messages to a text file.

- iOS: Chat → Contact info → Export Chat → Without Media → Save/Share .txt
- Android: Chat → More → Export chat → Without media
- Name each file clearly: `family.txt`, `work.txt`, etc., and place them in the
  `./input` folder.

## Ingest chat logs

Create a new file named `ingest.py` and populate with this content:

```python
from llama_index.core import SimpleDirectoryReader, StorageContext, VectorStoreIndex
from llama_index.core.node_parser import TokenTextSplitter
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.vector_stores.duckdb import DuckDBVectorStore


vector_store = DuckDBVectorStore("duck.db", persist_dir="./data/")
storage_context = StorageContext.from_defaults(vector_store=vector_store)
embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-m3")

splitter = TokenTextSplitter(chunk_size=512, separator="\r\n")

documents = SimpleDirectoryReader("./input/").load_data()

index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    transformations=[splitter],
    embed_model=embed_model,
    show_progress=True,
)
```

The script loads your exported `.txt` chats, splits them into retrieval‑friendly
chunks, calculates the embeddings, and persists everything to DuckDB:

- Vector store: `DuckDBVectorStore("duck.db", persist_dir="./data/")` stores
  both vectors and metadata on disk under `./data/`, so you can reuse the index
  without re‑ingesting.
- Embeddings: `BAAI/bge-m3` is a strong multilingual embedding model that runs
  locally via Hugging Face. You can swap it for a smaller/faster model if
  needed.
- Chunking: `TokenTextSplitter(chunk_size=512, separator="\r\n")` breaks the raw
  chat text along line breaks, keeping messages together while limiting token
  length for better retrieval.
- Reader: `SimpleDirectoryReader("./input/")` loads every `.txt` file in the
  folder and attaches basic file metadata (e.g., filename).
- Index build: `VectorStoreIndex.from_documents(...)` generates embeddings for
  each chunk and writes them to DuckDB with progress reporting.

After running this once, the built index is persisted and can be opened later
for querying without reprocessing the input files.

To run the script you can use this command:

```shell
uv run ingest.py
```

## Main chat app

Next, you have the actual RAG application that uses the index generated on the
previous step.

```python
import gradio
from llama_index.core import VectorStoreIndex
from llama_index.core.prompts import ChatMessage
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.llms.ollama import Ollama
from llama_index.vector_stores.duckdb import DuckDBVectorStore

embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-m3", device="cpu")
vector_store = DuckDBVectorStore.from_local("./data/duck.db")
index = VectorStoreIndex.from_vector_store(vector_store, embed_model=embed_model)


llm = Ollama(
    model="gpt-oss:20b",
    request_timeout=300,
    context_window=1024 * 10,
)

engine = index.as_chat_engine(
    llm=llm,
    similarity_top_k=5,
    system_prompt=(
        "You are a helpful assistant that searches WhatsApp "
        "messages to answer questions"
    ),
    streaming=True,
)


def stream(input: str, history: list[dict[str, str]]):
    chat_history = [
        ChatMessage(role=item["role"], content=item["content"]) for item in history
    ]
    content = ""
    for token in engine.stream_chat(input, chat_history=chat_history).response_gen:
        content += token
        yield content


chat = gradio.ChatInterface(
    fn=stream,
    type="messages",
    title="RacinhoGPT",
).launch()
```

This file wires the stored index to an LLM and a simple chat UI:

- Load index: `DuckDBVectorStore.from_local("./data/duck.db")` reopens the
  previously persisted vectors, and `VectorStoreIndex.from_vector_store(...)`
  prepares a retriever over them using the same embedding model.
- Local LLM: `Ollama(model="gpt-oss:20b")` runs a local model for generation.
  You can replace it with another Ollama model (e.g., `llama3`) if preferred.
- Chat engine: `index.as_chat_engine(...)` handles retrieval‑augmented
  generation with `top_k=5` similar chunks and a concise system prompt.
- Streaming: `engine.stream_chat(...)` yields tokens as they are generated; the
  `stream` function accumulates and streams them back to Gradio for a live UI.
- History: Incoming `history` messages are converted to `ChatMessage`s so the
  LLM can keep context across turns.
- UI: `gradio.ChatInterface` provides a minimal chat app you can open in the
  browser. Title is arbitrary—rename freely.

Once launched, type a question like “When did we discuss the ski trip?” and the
assistant retrieves relevant messages from your chats and answer grounded on
those snippets.

```shell
uv run main.py
```

## Cleanup

WhatsApp exports vary by locale (date order, 12/24h time) and platform. We’ll
use a tolerant regex and the `dateutil` parser, and we’ll stitch multi‑line
messages back together.

Create a parser that yields LlamaIndex `Document`s—one per chat—containing
formatted message lines and useful metadata per document.

Notes:

- This groups each chat export into a single `Document`, then uses the chunker
  to create semantic nodes for retrieval.
- Metadata includes `chat`, `start`, `end`, and `messages` for context and
  future filters.
- For purely local processing, keep the default HuggingFace embedding and do not
  set a cloud LLM. You can still retrieve relevant passages; answering will be a
  simple synthesis from the retrieved chunks. For generative answers locally,
  enable Ollama.

If you prefer OpenAI for LLM/embeddings, configure instead:

## 3) First queries (CLI)

Build the index and ask:

If you didn’t enable an LLM (Ollama/OpenAI), the answer may be terse; you’ll
still see relevant source snippets. Enable a model to get fluent answers.

## Privacy, accuracy, and scale

- Local by default: Using HuggingFace embeddings keeps your data on disk. Add a
  local LLM via Ollama to keep generation local too. If you switch to cloud
  models, understand what data leaves your machine.
- Parsing differences: WhatsApp export formats vary. If your locale isn’t parsed
  correctly, tweak the regex or pass a fixed `dayfirst` setting.
- Chunking: Adjust `chunk_size`/`chunk_overlap` for your dataset length and
  retrieval quality. Larger chunks can help preserve context of multi‑message
  exchanges; smaller chunks improve pinpoint retrieval.
- Persistence: Reusing the `storage/` directory avoids re‑indexing on every run.
  Delete it to rebuild after adding new exports.
- Big histories: If you have many chats, consider splitting `Document`s by month
  (or week) to speed retrieval and enable date filters in metadata.

## Nice extensions

- Sender/date filters: Add `MetadataFilters` to restrict by chat name or time
  window.
- Summaries per chat: Build monthly summaries with `SummaryIndex` and link to
  raw sources.
- Notifications: A cron job to re‑index new exports dropped into the folder.
- Multi‑query or query fusion: Improve recall using `QueryFusionRetriever`.

## Troubleshooting

- No results returned: Check that `.txt` files actually have lines matching the
  regex (look for `" - Name: "` after a timestamp). Some exports include system
  lines; those are skipped.
- Wrong dates: Switch the order in the parser (set `dayfirst=True` or `False`
  consistently).
- Slow answers: Increase `top_k` a bit but keep it reasonable (6–10). For large
  corpora, consider a stronger embedding model.

## Wrap‑up

With a few dozen lines of parsing and LlamaIndex’s indexing/query APIs, you get
a private, semantic interface to your WhatsApp history. Drop new exports into
the folder and re‑run to stay up to date. In a follow‑up, we can add sender/date
filters and monthly summaries for more focused recall.
