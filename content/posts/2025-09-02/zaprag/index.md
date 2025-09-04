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

```shell
uv run ingest.py
```

## Main chat app

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

```shell
uv run main.py
```

WhatsApp exports vary by locale (date order, 12/24h time) and platform. We’ll
use a tolerant regex and the `dateutil` parser, and we’ll stitch multi‑line
messages back together.

Create a parser that yields LlamaIndex `Document`s—one per chat—containing
formatted message lines and useful metadata per document.

```python
# rag_whatsapp.py (parser + indexing + query)
from __future__ import annotations

import re
from pathlib import Path
from dataclasses import dataclass
from datetime import datetime
from typing import List, Tuple
from dateutil import parser as dateparser

from llama_index.core import Document, VectorStoreIndex, StorageContext, Settings, load_index_from_storage
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Optional: local LLM via Ollama (uncomment if you have it)
# from llama_index.llms.ollama import Ollama

# Optional: OpenAI (if you prefer cloud)
# from llama_index.llms.openai import OpenAI
# from llama_index.embeddings.openai import OpenAIEmbedding


@dataclass
class Message:
    ts: datetime
    sender: str
    text: str


MSG_RE = re.compile(
    r"^(?P<date>\d{1,2}[\/\-.]\d{1,2}[\/\-.]\d{2,4}),?\s+"
    r"(?P<time>\d{1,2}:\d{2}(?:\s?[AP]M)?)\s+-\s+"
    r"(?P<sender>[^:]+):\s+(?P<text>.*)$"
)


def parse_export(path: Path) -> List[Message]:
    messages: List[Message] = []
    current: Tuple[datetime, str, List[str]] | None = None  # (ts, sender, lines)

    def flush_current():
        nonlocal current
        if current is None:
            return
        ts, sender, lines = current
        text = "\n".join(lines).strip()
        messages.append(Message(ts=ts, sender=sender.strip(), text=text))
        current = None

    with path.open("r", encoding="utf-8", errors="ignore") as f:
        for raw in f:
            line = raw.rstrip("\n")
            m = MSG_RE.match(line)
            if m:
                # New message header
                flush_current()
                when = f"{m.group('date')} {m.group('time')}"
                # Try parsing with both dayfirst variants to handle locales
                ts = None
                for dayfirst in (True, False):
                    try:
                        ts = dateparser.parse(when, dayfirst=dayfirst)
                        break
                    except Exception:
                        pass
                if ts is None:
                    # Fallback: skip this line
                    continue
                current = (ts, m.group("sender"), [m.group("text")])
            else:
                # Continuation of previous message or system line; attach
                if current is None:
                    # Some exports contain preamble lines; skip
                    continue
                current[2].append(line)

    flush_current()
    return messages


def load_chats(dir_path: Path) -> List[Document]:
    docs: List[Document] = []
    for file in sorted(dir_path.glob("*.txt")):
        msgs = parse_export(file)
        if not msgs:
            continue
        msgs.sort(key=lambda m: m.ts)
        lines = [f"[{m.ts:%Y-%m-%d %H:%M}] {m.sender}: {m.text}" for m in msgs]
        text = "\n".join(lines)
        meta = {
            "chat": file.stem,
            "start": msgs[0].ts.isoformat(),
            "end": msgs[-1].ts.isoformat(),
            "messages": len(msgs),
        }
        docs.append(Document(text=text, metadata=meta))
    return docs


def build_or_load_index(data_dir: Path, storage_dir: Path) -> VectorStoreIndex:
    storage_dir.mkdir(parents=True, exist_ok=True)

    # Configure models (local, privacy‑friendly defaults)
    Settings.embed_model = HuggingFaceEmbedding(
        model_name="sentence-transformers/all-MiniLM-L6-v2"
    )
    # If you have Ollama running locally, uncomment to answer with a local LLM
    # Settings.llm = Ollama(model="llama3", request_timeout=60.0)

    if any(storage_dir.iterdir()):
        storage_context = StorageContext.from_defaults(persist_dir=str(storage_dir))
        return load_index_from_storage(storage_context)

    docs = load_chats(data_dir)
    if not docs:
        raise SystemExit(f"No .txt exports found in {data_dir}")

    # Chunking setup
    Settings.node_parser = SentenceSplitter(chunk_size=800, chunk_overlap=80)

    index = VectorStoreIndex.from_documents(docs, show_progress=True)
    index.storage_context.persist(persist_dir=str(storage_dir))
    return index


def ask(index: VectorStoreIndex, question: str, top_k: int = 6):
    # Simple query engine; you can add postprocessors/filters later
    qe = index.as_query_engine(similarity_top_k=top_k)
    return qe.query(question)


if __name__ == "__main__":
    import argparse

    p = argparse.ArgumentParser(description="WhatsApp RAG over exports")
    p.add_argument("question", nargs="?", help="Question to ask")
    p.add_argument("--data", default="data/whatsapp_exports", help="Folder with .txt exports")
    p.add_argument("--store", default="storage", help="Index storage folder")
    p.add_argument("--top_k", type=int, default=6, help="Retriever top_k")
    args = p.parse_args()

    data_dir = Path(args.data)
    storage_dir = Path(args.store)
    index = build_or_load_index(data_dir, storage_dir)

    if not args.question:
        print("Index ready. Ask a question, e.g.:\n  python rag_whatsapp.py \"When did we schedule dentist?\"")
    else:
        resp = ask(index, args.question, top_k=args.top_k)
        print("\nAnswer:\n", str(resp))
        print("\nSources:")
        for i, sn in enumerate(resp.source_nodes, 1):
            meta = sn.metadata or {}
            print(f"  [{i}] chat={meta.get('chat')} score={sn.score:.3f}")
            # Print a short preview
            preview = sn.text.strip().splitlines()[:3]
            for pl in preview:
                print("     ", pl)
```

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

```python
# Settings.llm = OpenAI(model="gpt-4o-mini")
# Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
```

## 3) First queries (CLI)

Build the index and ask:

```shell
uv run rag_whatsapp.py "When did we confirm the dentist appointment?"
uv run rag_whatsapp.py "Share the address Fabio sent for dinner"
uv run rag_whatsapp.py "Summarize the last 2 weeks from the family group"
```

If you didn’t enable an LLM (Ollama/OpenAI), the answer may be terse; you’ll
still see relevant source snippets. Enable a model to get fluent answers.

## 4) Optional: tiny Streamlit UI

Add a minimal local UI to ask questions and inspect sources.

```python
# app.py
from pathlib import Path
import streamlit as st
from llama_index.core import StorageContext, load_index_from_storage, Settings
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Optional LLMs
# from llama_index.llms.ollama import Ollama

STORAGE = Path("storage")

@st.cache_resource(show_spinner=False)
def load_index():
    Settings.embed_model = HuggingFaceEmbedding(
        model_name="sentence-transformers/all-MiniLM-L6-v2"
    )
    # Settings.llm = Ollama(model="llama3", request_timeout=60.0)
    sc = StorageContext.from_defaults(persist_dir=str(STORAGE))
    return load_index_from_storage(sc)


st.set_page_config(page_title="WhatsApp RAG", page_icon="💬", layout="wide")
st.title("WhatsApp RAG 💬🔎")

idx = load_index()
q = st.text_input("Ask a question", placeholder="When did we confirm the dentist appointment?")
top_k = st.slider("Results", 3, 12, 6)

if q:
    qe = idx.as_query_engine(similarity_top_k=top_k)
    with st.spinner("Thinking..."):
        resp = qe.query(q)
    st.subheader("Answer")
    st.write(str(resp))

    st.subheader("Sources")
    for i, sn in enumerate(resp.source_nodes, 1):
        with st.expander(f"[{i}] chat={sn.metadata.get('chat')} score={sn.score:.3f}"):
            st.code(sn.text, language="")
```

Run:

```shell
uv run streamlit run app.py
```

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
