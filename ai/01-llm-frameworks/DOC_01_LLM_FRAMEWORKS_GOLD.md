# 🧠 MODULE 1 — LLM Frameworks (GOLD EDITION)
### LangChain · LangGraph · LlamaIndex · Hugging Face

---

## 🧭 Where You Are

```
[✅ DOC_00 — Master Index]
[📍 DOC_01 — LLM Frameworks — GOLD]  ← YOU ARE HERE
[ ] DOC_02 — Model Serving & Inference
[ ] DOC_03 — Vector Storage
[ ] DOC_04 — Frontend / UI
[ ] DOC_05 — Capstone Project
```

**Previous:** [Master Index](DOC_00_MASTER_INDEX.md) &nbsp;|&nbsp; **Next:** [Module 2 — Model Serving](DOC_02_MODEL_SERVING.md)

---

## 🗺️ The Big Picture First (Read This Before Anything)

Before you touch a single line of code, here is the one mental model that makes everything in this module click:

> **These 4 tools are not competitors. They are layers. Each one solves a different problem at a different level of abstraction.**

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR AI APPLICATION                       │
├─────────────────────────────────────────────────────────────┤
│  HUGGING FACE  ←  "Which model do I use and how do I run it?"│
├─────────────────────────────────────────────────────────────┤
│  LANGCHAIN     ←  "How do I chain prompts and tools?"        │
├─────────────────────────────────────────────────────────────┤
│  LLAMAINDEX    ←  "How do I feed my own data to the model?"  │
├─────────────────────────────────────────────────────────────┤
│  LANGGRAPH     ←  "How do I control the flow of reasoning?"  │
└─────────────────────────────────────────────────────────────┘
```

A real app often uses all four. A simple chatbot might only use LangChain. A research agent needs all of them. Knowing *which layer your problem lives in* is the skill this module builds.

**Time investment:** ~3–4 days. One tool per day. Don't rush.

---

---

# 🔗 PART 1: LANGCHAIN
## The Assembly Line for LLM Logic

---

### 🧠 Brain Anchor
> *LangChain = LEGO for LLM apps. Each piece snaps together. The `|` pipe is the snap.*

---

### Why Does LangChain Even Exist?

Without a framework, building even a simple AI feature looks like this:

```python
# Without LangChain — raw OpenAI
import openai

def answer_question(topic, question):
    client = openai.OpenAI()
    
    # Manually format the prompt string
    prompt = f"You are an expert on {topic}. Answer this question clearly: {question}"
    
    # Manually call the API
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Manually extract the content
    return response.choices[0].message.content
```

That's fine for one function. Now imagine you have 15 functions, each with slightly different prompts, models, parsers, memory, and tools. You end up with a tangled mess of strings, API calls, and custom parsing logic scattered across your codebase.

LangChain gives you **composable, reusable pieces** so your AI logic is structured, readable, and maintainable.

---

### The Core Concept: LCEL — The Pipe Operator `|`

LCEL stands for **LangChain Expression Language**. It's the `|` operator.

This is the single most important thing to understand in LangChain:

```python
chain = prompt | llm | parser
result = chain.invoke({"question": "What is AI?"})
```

Each `|` passes the output of the left side as input to the right side — exactly like Unix pipes in a terminal.

Let's make this crystal clear with a side-by-side:

```
Unix:        cat file.txt | grep "error" | sort | uniq
LangChain:   prompt | llm | output_parser | next_step
```

**What flows through the pipe?**

```
Input dict  →  [Prompt Template]  →  Formatted string
            →  [LLM]              →  AIMessage object
            →  [StrOutputParser]  →  Plain string
            →  [Next Chain]       →  Whatever you want
```

> 💡 **Gotcha #1:** The `|` operator in LangChain is NOT Python's bitwise OR. LangChain overrides the `__or__` dunder method on its objects. This is why `prompt | llm` returns a runnable chain, not a number.

---

### The 6 Building Blocks of LangChain

Learn these 6 things and you know 90% of LangChain:

---

#### 1. 📝 Prompt Templates — "Mad Libs for AI"

A prompt template is just a string with `{variables}` that get filled in at runtime.

```python
from langchain.prompts import ChatPromptTemplate

# Simple template
prompt = ChatPromptTemplate.from_template(
    "Explain {topic} to a {audience} in {style} style."
)

# At runtime, variables get filled in:
formatted = prompt.format_messages(
    topic="neural networks",
    audience="5-year-old",
    style="storytelling"
)
# Result: "Explain neural networks to a 5-year-old in storytelling style."
```

**System + Human messages (most common pattern):**

```python
from langchain.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}. Always respond in {language}."),
    ("human", "{question}")
])

# Now call it:
chain = prompt | llm
chain.invoke({"role": "chef", "language": "French", "question": "How do I make pasta?"})
```

> 🏢 **In the Wild:** Every AI product with a persona (Claude, Gemini, character chatbots) uses system prompts exactly like this. The "personality" is a `SystemMessagePromptTemplate`.

---

#### 2. 🤖 LLMs and Chat Models — "The Brain"

LangChain wraps any LLM behind a standard interface. You can swap OpenAI for Ollama for Anthropic with one line change.

```python
# OpenAI
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# Anthropic (Claude)
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-sonnet-4-6")

# Ollama (local, free)
from langchain_community.llms import Ollama
llm = Ollama(model="llama3.2")

# The rest of your code is IDENTICAL regardless of which LLM you use
chain = prompt | llm | parser
```

**Temperature explained:**
```
temperature=0.0  →  Deterministic, always same answer. Good for: facts, code, extraction
temperature=0.7  →  Balanced creativity. Good for: general chat, summaries
temperature=1.0  →  Very creative / random. Good for: brainstorming, stories
temperature>1.0  →  Often incoherent. Avoid.
```

> ⚡ **Gotcha #2:** `ChatOpenAI` and `OpenAI` are DIFFERENT classes. `ChatOpenAI` is for chat models (gpt-4, claude, etc.) and returns `AIMessage` objects. `OpenAI` is for the older completion API. Use `ChatOpenAI` unless you specifically need the old API.

---

#### 3. 🔧 Output Parsers — "The Translator"

LLMs return `AIMessage` objects. Output parsers convert them to something useful.

```python
from langchain.schema.output_parser import StrOutputParser
from langchain.output_parsers import CommaSeparatedListOutputParser
from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

# 1. String parser (most common)
str_parser = StrOutputParser()
chain = prompt | llm | str_parser
# Returns: "plain text string"

# 2. List parser
list_parser = CommaSeparatedListOutputParser()
chain = prompt | llm | list_parser
# Returns: ["item1", "item2", "item3"]

# 3. Structured output with Pydantic (POWERFUL)
class MovieReview(BaseModel):
    title: str = Field(description="Movie title")
    rating: float = Field(description="Rating from 1-10")
    summary: str = Field(description="One sentence summary")

parser = PydanticOutputParser(pydantic_object=MovieReview)
prompt = ChatPromptTemplate.from_template(
    "Review this movie: {movie}\n{format_instructions}"
)
chain = prompt | llm | parser
result = chain.invoke({
    "movie": "Inception",
    "format_instructions": parser.get_format_instructions()
})
# Returns: MovieReview(title="Inception", rating=9.2, summary="A mind-bending thriller...")
```

> 💡 **Pro tip:** Pydantic parsers are gold for production apps. Instead of parsing LLM output with regex (fragile), you tell the LLM exactly what JSON structure to return and Pydantic validates it.

---

#### 4. 🧠 Memory — "Short-Term Brain"

By default, LLMs are stateless — every call is independent. Memory gives your chain a sense of history.

```python
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

llm = ChatOpenAI(model="gpt-4o-mini")
memory = ConversationBufferMemory()

conversation = ConversationChain(llm=llm, memory=memory)

# Turn 1
response = conversation.predict(input="My name is Rahul")
print(response)  # "Hi Rahul! Nice to meet you."

# Turn 2 — it remembers!
response = conversation.predict(input="What's my name?")
print(response)  # "Your name is Rahul!"
```

**Memory types — which one to use:**

| Memory Type | What it stores | Token cost | When to use |
|---|---|---|---|
| `ConversationBufferMemory` | Full history | High (grows forever) | Short conversations |
| `ConversationBufferWindowMemory` | Last K turns | Fixed | Chat apps |
| `ConversationSummaryMemory` | AI-summarized history | Medium | Long sessions |
| `ConversationTokenBufferMemory` | Last N tokens | Fixed | When you care about exact limits |

> ⚡ **Gotcha #3:** `ConversationBufferMemory` will blow your context window for long conversations. For anything meant to run for 10+ turns, use `ConversationSummaryMemory` or `ConversationBufferWindowMemory(k=5)`.

---

#### 5. 🛠️ Tools — "Superpowers for the LLM"

Tools let an LLM take actions in the real world — search the web, run Python, call APIs.

```python
from langchain.tools import tool
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.prompts import ChatPromptTemplate

# Define a custom tool
@tool
def get_stock_price(ticker: str) -> str:
    """Get the current stock price for a given ticker symbol."""
    # In real life: call a finance API here
    prices = {"AAPL": "182.50", "GOOGL": "140.22", "MSFT": "378.85"}
    return prices.get(ticker.upper(), "Ticker not found")

@tool
def calculate(expression: str) -> str:
    """Calculate a mathematical expression."""
    try:
        return str(eval(expression))
    except:
        return "Invalid expression"

# Create an agent with tools
llm = ChatOpenAI(model="gpt-4o-mini")
tools = [get_stock_price, calculate]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful financial assistant."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({"input": "What's the price of Apple stock? And what's 182.50 * 1000?"})
# The LLM decides to call get_stock_price, then calculate, then responds.
```

> 🏢 **In the Wild:** ChatGPT's "Code Interpreter" is exactly this — a tool that runs Python. Perplexity's web search is a tool. Every AI with internet access has a search tool wired up via this exact pattern.

---

#### 6. 🔗 Chains — "Combining Everything"

Chains connect multiple steps. Here's a real-world pattern: the **Sequential Chain**:

```python
from langchain.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain.schema.output_parser import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini")
parser = StrOutputParser()

# Step 1: Generate blog post outline
outline_prompt = ChatPromptTemplate.from_template(
    "Create a 5-point blog post outline about: {topic}"
)

# Step 2: Write the intro based on the outline
intro_prompt = ChatPromptTemplate.from_template(
    "Write a compelling intro paragraph for this blog outline:\n{outline}"
)

# Step 3: Generate a catchy title
title_prompt = ChatPromptTemplate.from_template(
    "Write 3 catchy titles for this intro:\n{intro}"
)

# Chain them all together
outline_chain = outline_prompt | llm | parser
intro_chain = intro_prompt | llm | parser
title_chain = title_prompt | llm | parser

# Full pipeline
full_chain = (
    outline_chain
    | (lambda outline: {"outline": outline})
    | intro_chain
    | (lambda intro: {"intro": intro})
    | title_chain
)

result = full_chain.invoke({"topic": "AI frameworks for beginners"})
print(result)
```

---

### 🎯 When to Use LangChain vs Not

| Use LangChain when... | Skip LangChain when... |
|---|---|
| Chaining 2+ prompts together | Single, one-off API call |
| Need memory / conversation history | Simple script or CLI tool |
| Want to swap LLM providers easily | Ultra-low latency is critical |
| Building tools or agents | You just need `openai.chat.completions.create` |
| Want structured JSON output | App is <3 LLM calls |

---

### 🎮 Challenges — LangChain

**🟢 Rookie:** Build a 3-step chain: Topic → Outline → 3 catchy titles. Print each intermediate result.

**🟡 Builder:** Build a "Resume Analyzer" — take a job description and a resume, and return a Pydantic object with `match_score` (0–100), `missing_skills` (list), and `recommendation` (hire/reject/interview).

**🔴 Architect:** Build a tool-using agent with 3 custom tools: one that reads a local `.txt` file, one that counts words, one that summarizes. Chain them so the agent reads a file, counts its words, and summarizes it — in one agent call.

---

---

# 🔀 PART 2: LANGGRAPH
## When Straight Lines Aren't Enough

---

### 🧠 Brain Anchor
> *LangGraph = LangChain with a steering wheel. Chains go forward. Graphs can loop, branch, and U-turn.*

---

### The Problem LangGraph Solves

LangChain chains are linear: Step A → Step B → Step C → Done.

That works for 80% of tasks. But real agents need to:
- Loop back if the answer isn't good enough
- Choose different paths based on what they find
- Run steps in parallel
- Stop early when a condition is met

This is where chains break and LangGraph begins.

**The hiring manager analogy:**
```
Linear (LangChain):
    Screen resume → Phone call → Decision
    (No loops, no going back)

Real hiring process (LangGraph):
    Screen resume
    → If promising: Phone call
    → If phone call good: Technical interview
    → If failed: Loop back to screening more candidates
    → If passed: Offer
    → If offer rejected: Loop back to pipeline
    (Branches, loops, conditional routing — this is a graph)
```

---

### The 4 Core Concepts of LangGraph

---

#### 1. 📦 State — "The Whiteboard"

State is a shared data structure that every node in the graph can read from and write to. Think of it as a whiteboard in a meeting room — anyone can add to it, and everyone sees updates.

```python
from typing import TypedDict, List

class ResearchState(TypedDict):
    topic: str            # Input from user
    questions: List[str]  # Generated questions
    answers: List[str]    # Answers gathered
    draft: str            # Draft report
    revision_count: int   # How many times we've revised
    is_good: bool         # Quality check result
```

> 💡 **Always define state as a TypedDict.** This makes it inspectable, debuggable, and type-safe. Every node signature is `def my_node(state: MyState) -> MyState`.

---

#### 2. 🔷 Nodes — "The Workers"

Each node is a Python function that reads the current state, does something, and returns an updated state.

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

def generate_questions(state: ResearchState) -> ResearchState:
    """Node 1: Generate research questions from a topic."""
    response = llm.invoke(
        f"Generate 3 research questions about: {state['topic']}"
    )
    questions = response.content.split('\n')
    return {**state, "questions": questions}

def answer_questions(state: ResearchState) -> ResearchState:
    """Node 2: Answer each question."""
    answers = []
    for q in state['questions']:
        answer = llm.invoke(f"Answer this research question: {q}").content
        answers.append(answer)
    return {**state, "answers": answers}

def write_draft(state: ResearchState) -> ResearchState:
    """Node 3: Write a draft report from answers."""
    qa_pairs = "\n".join([f"Q: {q}\nA: {a}" for q, a in zip(state['questions'], state['answers'])])
    draft = llm.invoke(f"Write a cohesive report from these Q&A pairs:\n{qa_pairs}").content
    return {**state, "draft": draft, "revision_count": state['revision_count'] + 1}

def quality_check(state: ResearchState) -> ResearchState:
    """Node 4: Check if the draft is good enough."""
    result = llm.invoke(
        f"Is this report high quality? Answer only YES or NO.\n\n{state['draft']}"
    ).content
    return {**state, "is_good": "YES" in result.upper()}
```

---

#### 3. ➡️ Edges — "The Roads Between Workers"

Edges connect nodes. There are two types:

```python
from langgraph.graph import StateGraph, END

graph = StateGraph(ResearchState)

# Add nodes
graph.add_node("generate_questions", generate_questions)
graph.add_node("answer_questions", answer_questions)
graph.add_node("write_draft", write_draft)
graph.add_node("quality_check", quality_check)

# Normal edges (always go this direction)
graph.add_edge("generate_questions", "answer_questions")
graph.add_edge("answer_questions", "write_draft")
graph.add_edge("write_draft", "quality_check")

# Set start node
graph.set_entry_point("generate_questions")
```

---

#### 4. 🚦 Conditional Edges — "The Forks in the Road"

This is where the magic happens. Conditional edges route to different nodes based on state.

```python
def should_revise(state: ResearchState) -> str:
    """Routing function: returns the name of the next node."""
    if state['is_good']:
        return "done"          # Route to END
    elif state['revision_count'] >= 3:
        return "done"          # Max revisions reached, accept anyway
    else:
        return "revise"        # Route back to write_draft

# Add the conditional edge
graph.add_conditional_edges(
    "quality_check",            # From this node
    should_revise,              # Use this function to decide
    {
        "done": END,            # If "done", go to END
        "revise": "write_draft" # If "revise", loop back
    }
)

# Compile and run
app = graph.compile()

result = app.invoke({
    "topic": "climate change solutions",
    "questions": [],
    "answers": [],
    "draft": "",
    "revision_count": 0,
    "is_good": False
})

print(result['draft'])  # Final polished report
```

**The full flow visualized:**
```
                    [generate_questions]
                            │
                    [answer_questions]
                            │
                    ┌──[write_draft]◄──────────┐
                    │                           │ (if not good
                    ▼                           │  and < 3 tries)
              [quality_check]──── "revise" ────►┘
                    │
                  "done"
                    │
                  [END]
```

---

### 🎯 LangChain vs LangGraph: The Decision

```
┌─────────────────────────────────────────────────────────┐
│            Should I use LangChain or LangGraph?          │
│                                                           │
│  Does your flow go straight from A to B to C?           │
│  └─ YES → Use LangChain chains                          │
│                                                           │
│  Does your flow need to:                                 │
│  - Loop back based on a condition?         ┐             │
│  - Branch to different paths?              ├ YES → Use   │
│  - Run multiple steps in parallel?         │   LangGraph │
│  - Stop early based on a result?          ┘             │
└─────────────────────────────────────────────────────────┘
```

> 🏢 **In the Wild:** AutoGPT, Devin, and most "AI agent" products you've seen demoed are fundamentally LangGraph-style loops. The loop is: Think → Act → Observe → (good enough? stop : loop back to Think).

---

### 🎮 Challenges — LangGraph

**🟢 Rookie:** Build a graph with 2 nodes: `generate_answer` and `check_length`. If the answer is under 100 characters, route back to `generate_answer` with an instruction to be more detailed. Otherwise, go to END.

**🟡 Builder:** Build a "multi-agent debate" graph. Two nodes argue opposite sides of a topic. A third node judges. If the judge says "undecided", loop for another round (max 3 rounds).

**🔴 Architect:** Build a "code review" agent graph: Node 1 generates code for a task. Node 2 tests it (use Python `exec()` to run it). Node 3 fixes bugs if tests fail. Loop until code passes or 5 attempts reached.

---

---

# 📚 PART 3: LLAMAINDEX
## Giving Your LLM Access to Your Own Data

---

### 🧠 Brain Anchor
> *LlamaIndex = search engine + librarian for your AI. It makes your private data queryable by an LLM.*

---

### The RAG Problem (Made Visceral)

Imagine you just joined a new company. You need to answer a customer question. You know a lot about the world (like an LLM), but you don't know company-specific things: internal processes, product specs, past decisions, customer history.

Without LlamaIndex:
```
Customer: "What's the refund policy for orders over $200?"
LLM:      "I don't have access to your company's specific policies."  ← useless
```

With LlamaIndex:
```
Customer: "What's the refund policy for orders over $200?"
LlamaIndex: Searches your policy docs → finds the relevant paragraph
LLM:        "Based on your policy docs: Orders over $200 get 60 days for refunds..."  ← useful
```

This pattern — retrieve relevant docs, then generate an answer — is called **RAG: Retrieval-Augmented Generation**. LlamaIndex is the best tool for building it.

---

### The RAG Pipeline: 5 Stages

```
INDEXING TIME (one time setup):
Documents → [Chunking] → [Embedding] → [Vector Index] → Stored

QUERY TIME (every request):
Question → [Embed question] → [Similarity search] → [Top N chunks] → [LLM + chunks] → Answer
```

Let's break down each stage:

---

#### Stage 1: 📄 Document Loading

LlamaIndex can load from almost anywhere:

```python
from llama_index.core import SimpleDirectoryReader
from llama_index.readers.web import SimpleWebPageReader
from llama_index.readers.file import PDFReader

# From a folder (loads all .txt, .pdf, .md, etc.)
documents = SimpleDirectoryReader("./my_docs").load_data()

# From a specific PDF
documents = PDFReader().load_data(file="report.pdf")

# From web pages
documents = SimpleWebPageReader().load_data(
    urls=["https://docs.example.com/page1", "https://docs.example.com/page2"]
)

print(f"Loaded {len(documents)} documents")
print(f"First doc preview: {documents[0].text[:200]}")
```

---

#### Stage 2: ✂️ Chunking — "Why Size Matters"

A 100-page PDF can't fit in a single LLM context window. Chunking splits documents into smaller pieces.

```
Full document:  ████████████████████████████████████████  (too big)
After chunking: ████  ████  ████  ████  ████  ████  ████  (queryable)
```

**Chunk size tradeoffs:**

| Chunk size | Retrieval precision | Context richness | Tokens per query |
|---|---|---|---|
| Very small (128 tokens) | ✅ Very precise | ❌ No context around the fact | Low |
| Medium (512 tokens) | ✅ Good | ✅ Decent context | Medium |
| Large (2048 tokens) | ❌ Less precise | ✅ Rich context | High |

> 💡 **Sweet spot:** Start with `chunk_size=512`, `chunk_overlap=50`. The overlap ensures facts that span chunk boundaries don't get cut off.

```python
from llama_index.core import Settings
from llama_index.core.node_parser import SentenceSplitter

# Configure chunking globally
Settings.text_splitter = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=50
)
```

---

#### Stage 3: 🔢 Embedding — "Translating Meaning to Math"

Each chunk gets converted to a vector — a list of numbers that captures its semantic meaning.

```
Chunk: "Python is great for data science"
Embed: [0.12, -0.87, 0.34, 0.91, ... 1536 numbers total]

Chunk: "Machine learning with Python is popular"
Embed: [0.11, -0.84, 0.36, 0.89, ... similar numbers!]

Chunk: "I love eating pizza"
Embed: [0.78, 0.22, -0.56, 0.12, ... very different numbers!]
```

Similar meaning → similar vectors → found by similarity search.

LlamaIndex handles this automatically, but you can control which embedding model to use:

```python
from llama_index.core import Settings
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Option 1: OpenAI embeddings (paid, high quality)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-ada-002")

# Option 2: Free local embeddings (no API cost)
Settings.embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-small-en-v1.5")
```

---

#### Stage 4: 🗂️ Indexing — "Building the Library Catalog"

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Use free local embeddings + local LLM (zero cost!)
Settings.embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-small-en-v1.5")
Settings.llm = OpenAI(model="gpt-4o-mini")

# Load and index documents
documents = SimpleDirectoryReader("./docs").load_data()
index = VectorStoreIndex.from_documents(documents)

# Persist to disk so you don't re-index every time
index.storage_context.persist(persist_dir="./index_storage")
```

**Loading a persisted index:**
```python
from llama_index.core import StorageContext, load_index_from_storage

storage_context = StorageContext.from_defaults(persist_dir="./index_storage")
index = load_index_from_storage(storage_context)
# Index loaded instantly from disk — no re-embedding!
```

> ⚡ **Gotcha #4:** Always persist your index. Re-embedding 500 documents every time your app starts is slow and wasteful. Embed once, save, load from disk thereafter.

---

#### Stage 5: 🔍 Querying — "Asking Questions"

```python
# Basic query
query_engine = index.as_query_engine()
response = query_engine.query("What is the return policy?")
print(response.response)             # The answer
print(response.source_nodes)         # Where the answer came from

# Query with source citations
for node in response.source_nodes:
    print(f"Source: {node.metadata.get('file_name', 'unknown')}")
    print(f"Relevance score: {node.score:.3f}")
    print(f"Excerpt: {node.text[:150]}...")
    print("---")
```

**Query modes:**

```python
# Default: finds top 2 chunks, sends to LLM
query_engine = index.as_query_engine(similarity_top_k=2)

# More context (slower, more accurate)
query_engine = index.as_query_engine(similarity_top_k=5)

# Citation mode: automatically adds source references to the answer
from llama_index.core.query_engine import CitationQueryEngine
query_engine = CitationQueryEngine.from_args(index, similarity_top_k=3)
```

---

### Real Scenario: Company Knowledge Base

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI

Settings.llm = OpenAI(model="gpt-4o-mini")

# Your company docs folder
documents = SimpleDirectoryReader("./company_docs").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(similarity_top_k=3)

# Ask anything about your company
questions = [
    "What's the vacation policy?",
    "How do I submit an expense report?",
    "What are the engineering team's on-call hours?"
]

for question in questions:
    response = query_engine.query(question)
    print(f"\nQ: {question}")
    print(f"A: {response.response}")
```

---

### LlamaIndex vs LangChain: When to Use Which

This is the most common question in this module.

| Situation | Use |
|---|---|
| Querying your own documents (PDFs, notes, wikis) | **LlamaIndex** |
| Chaining prompts in sequence | **LangChain** |
| Need fine-grained control over retrieval strategy | **LlamaIndex** |
| Need tools + agents with web/API access | **LangChain** |
| Complex RAG with hybrid search, reranking | **LlamaIndex** |
| Memory and multi-turn conversation | **LangChain** |
| Both? | **Use LlamaIndex for retrieval, LangChain to orchestrate everything** |

> 🏢 **In the Wild:** Notion AI uses a LlamaIndex-style retrieval system to answer questions about your notes. Cursor uses it to answer questions about your codebase. Any "chat with your PDF" product is built on this exact pattern.

---

### 🎮 Challenges — LlamaIndex

**🟢 Rookie:** Put 5 text files about any topic you know (recipes, sports stats, travel notes) in a folder. Index them. Ask 5 questions that require finding information across multiple files. Print the source file for each answer.

**🟡 Builder:** Build a "Book Club Assistant" — index 3 different summaries/chapter notes. Ask cross-document questions like "What's the common theme across all three books?" Print the relevant excerpts LlamaIndex retrieved before forming the answer.

**🔴 Architect:** Build a RAG system that automatically evaluates its own answers. After retrieving and answering, send the answer + retrieved chunks back to the LLM and ask: "On a scale of 1–10, how well does this answer use the provided context?" If score < 7, retrieve more chunks and re-answer.

---

---

# 🤗 PART 4: HUGGING FACE
## The Open-Source Model Universe

---

### 🧠 Brain Anchor
> *Hugging Face = GitHub for AI models. 500,000+ models, free to download, yours to run.*

---

### Why Hugging Face Matters

OpenAI and Anthropic are powerful but they come with tradeoffs:

```
OpenAI / Claude (Closed):          Hugging Face (Open):
─────────────────────────          ───────────────────
✅ State of the art quality        ✅ Free to run
✅ Zero setup                      ✅ Works offline
❌ Costs money per token           ✅ Full control / customizable
❌ Data goes to their servers      ✅ Your data stays local
❌ Can change prices / shut down   ✅ Model weights are yours forever
❌ Rate limits                     ❌ Needs hardware / setup
```

For many use cases — internal tools, sensitive data, cost-sensitive apps, research — open-source models are the right choice.

---

### The 3 Things You'll Actually Use

---

#### 1. 🚀 Pipelines — "Zero to Inference in 2 Lines"

A pipeline is the fastest way to run any model. You name the task, pick a model, and go.

```python
from transformers import pipeline

# Text generation
generator = pipeline("text-generation", model="gpt2")
result = generator("Once upon a time in Silicon Valley,", max_new_tokens=100)
print(result[0]['generated_text'])

# Summarization (zero API key needed!)
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")
text = "LangChain is an open-source framework for developing applications powered by large language models (LLMs). It provides a standard interface for chains, integrates with various tools, and provides end-to-end chains for common applications."
summary = summarizer(text, max_length=60, min_length=20)
print(summary[0]['summary_text'])

# Sentiment analysis
classifier = pipeline("sentiment-analysis")
result = classifier("I absolutely love working with LangChain!")
print(result)  # [{'label': 'POSITIVE', 'score': 0.9998}]

# Translation
translator = pipeline("translation_en_to_fr", model="Helsinki-NLP/opus-mt-en-fr")
result = translator("Hello, how are you?")
print(result[0]['translation_text'])  # "Bonjour, comment allez-vous?"

# Question answering
qa = pipeline("question-answering", model="distilbert-base-cased-distilled-squad")
context = "LlamaIndex was created by Jerry Liu in 2022. It helps connect LLMs to external data sources."
result = qa(question="Who created LlamaIndex?", context=context)
print(result['answer'])  # "Jerry Liu"
```

**The full pipeline task list:**

| Task | Pipeline name | Common model |
|---|---|---|
| Text generation | `"text-generation"` | `gpt2`, `TinyLlama` |
| Summarization | `"summarization"` | `facebook/bart-large-cnn` |
| Translation | `"translation_en_to_fr"` | `Helsinki-NLP/opus-mt-en-fr` |
| Sentiment | `"sentiment-analysis"` | `distilbert-base-uncased-finetuned-sst-2-english` |
| Q&A | `"question-answering"` | `distilbert-base-cased-distilled-squad` |
| NER (entity extraction) | `"ner"` | `dslim/bert-base-NER` |
| Image classification | `"image-classification"` | `google/vit-base-patch16-224` |
| Text-to-image | `"text-to-image"` | `runwayml/stable-diffusion-v1-5` |

---

#### 2. 🔡 Tokenizers — "How Models See Text"

LLMs don't read words. They read tokens — roughly 3/4 of a word each.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

text = "LangChain is a powerful framework"
tokens = tokenizer.tokenize(text)
print(tokens)
# ['lang', '##chain', 'is', 'a', 'powerful', 'framework']

# Token IDs (what the model actually receives)
ids = tokenizer.encode(text)
print(ids)  # [101, 4902, 4606, 2003, 1037, 3928, 7705, 102]

# Decode back to text
decoded = tokenizer.decode(ids)
print(decoded)  # "[CLS] langchain is a powerful framework [SEP]"

# Why does this matter? Count tokens before sending to LLM
token_count = len(tokenizer.encode(text))
print(f"This text is {token_count} tokens")
```

> 💡 **Why tokenizers matter for you:** Pricing is per token. Context windows have token limits. "##chain" being a separate token explains why LLMs sometimes misspell compound words. Understanding tokenization helps you write better prompts.

---

#### 3. 🏋️ Model Loading — "Full Control"

For more control than pipelines offer, load the model and tokenizer directly:

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_name = "TinyLlama/TinyLlama-1.1B-Chat-v1.0"  # Small, free, fast

# Load tokenizer
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Load model (use float16 to save memory if you have GPU)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16 if torch.cuda.is_available() else torch.float32,
    device_map="auto"  # Automatically use GPU if available
)

# Format input with chat template
messages = [
    {"role": "system", "content": "You are a concise AI assistant."},
    {"role": "user", "content": "What is a transformer model?"}
]
formatted = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)

# Tokenize and generate
inputs = tokenizer(formatted, return_tensors="pt")
outputs = model.generate(
    **inputs,
    max_new_tokens=200,
    temperature=0.7,
    do_sample=True
)

# Decode output
response = tokenizer.decode(outputs[0][inputs['input_ids'].shape[1]:], skip_special_tokens=True)
print(response)
```

> ⚡ **Gotcha #5:** Models download to `~/.cache/huggingface/` on first run. A 7B model is ~14GB. Run `pip install huggingface_hub` and use `huggingface-cli download model-name` to pre-download before running your script, so you know the download isn't causing a timeout.

---

### Choosing the Right Model Size

```
SMALL (< 3B params) — CPU friendly, instant responses
├── TinyLlama 1.1B    → Best for: simple completions, testing
├── Phi-3 Mini 3.8B   → Best for: smart reasoning in small package
└── Gemma 2B          → Best for: general use on laptop

MEDIUM (7B–13B) — Needs GPU or powerful Mac M-chip
├── Mistral 7B        → Best for: instruction following, code
├── Llama 3.2 8B      → Best for: strong general performance
└── CodeLlama 7B      → Best for: code generation

LARGE (30B–70B) — Needs serious GPU hardware
├── Llama 3.1 70B     → Best for: complex reasoning
└── Mixtral 8x7B      → Best for: balanced speed + quality
```

---

### 🎮 Challenges — Hugging Face

**🟢 Rookie:** Run 3 different pipeline tasks on the same input text: sentiment analysis, summarization, and NER. Print all three outputs side by side.

**🟡 Builder:** Download `TinyLlama` and run it locally. Compare its response quality to GPT-2 on 5 questions. Write a short note on where each model shines and where it fails.

**🔴 Architect:** Build a "Local NLP Pipeline" — no API keys allowed. Take a paragraph of text and run it through: language detection → translation to English if not English → summarization → sentiment → keyword extraction. Chain all steps with Hugging Face pipelines.

---

---

# 🗺️ PART 5: THE MASTER DECISION MATRIX

---

When you're starting a new AI feature, use this to pick your tool(s):

```
WHAT ARE YOU BUILDING?
│
├── "I need to call an LLM and process the result"
│   └── LangChain — basic chain with StrOutputParser
│
├── "I need to chat with history/memory"
│   └── LangChain — ConversationChain + Memory
│
├── "I need to search / query my own documents"
│   └── LlamaIndex — VectorStoreIndex
│
├── "I need an AI agent that uses tools"
│   └── LangChain — create_tool_calling_agent + AgentExecutor
│
├── "I need an agent that can loop/retry/branch"
│   └── LangGraph — StateGraph with conditional edges
│
├── "I need to run a model locally for free"
│   └── Hugging Face — pipeline() or Ollama (Module 2)
│
├── "I need to query docs AND use tools AND have memory"
│   └── LlamaIndex for retrieval + LangChain for orchestration
│
└── "I need a complex multi-step agent that retrieves docs
    and can loop until satisfied"
    └── LangGraph (outer loop) + LlamaIndex (retrieval node)
        + LangChain (individual chains inside nodes)
```

---

# 🏗️ PART 6: MODULE 1 CAPSTONE BUILD

## The Multi-Step Q&A System

Build this end-to-end. It combines all 4 tools.

**What it does:** Takes a user question → searches your document library → generates an answer → checks quality → loops if not good enough → returns answer + sources.

```
User Question
      │
      ▼
[LangGraph: Orchestrator]
      │
      ├─── Node 1: [LlamaIndex] Search relevant docs
      │
      ├─── Node 2: [LangChain] Generate answer with context
      │
      ├─── Node 3: [LangChain] Quality check (is this a real answer?)
      │
      ├─── [Conditional] Good enough? → YES: return answer
      │                               → NO + attempts < 3: rephrase question and retry
      │                               → NO + attempts ≥ 3: return "I don't know"
      │
      └─── [Hugging Face] Translate answer if user language ≠ English
```

```python
# capstone_rag.py
from typing import TypedDict, Optional
from langgraph.graph import StateGraph, END
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser
from transformers import pipeline as hf_pipeline

# ─── State ───────────────────────────────────────────────────────
class QAState(TypedDict):
    question: str
    rephrased_question: str
    context: str
    sources: list
    answer: str
    quality_score: int
    attempts: int
    final_answer: str

# ─── Setup ───────────────────────────────────────────────────────
documents = SimpleDirectoryReader("./docs").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(similarity_top_k=3)

llm = ChatOpenAI(model="gpt-4o-mini")
str_parser = StrOutputParser()

# ─── Nodes ───────────────────────────────────────────────────────
def retrieve_context(state: QAState) -> QAState:
    """Node 1: Retrieve relevant document chunks."""
    q = state.get('rephrased_question') or state['question']
    response = query_engine.query(q)
    sources = [n.metadata.get('file_name', 'unknown') for n in response.source_nodes]
    return {**state, "context": response.response, "sources": list(set(sources))}

def generate_answer(state: QAState) -> QAState:
    """Node 2: Generate an answer using retrieved context."""
    prompt = ChatPromptTemplate.from_template("""
    Answer this question using only the provided context.
    If the context doesn't contain the answer, say "I couldn't find this in the documents."
    
    Context: {context}
    Question: {question}
    Answer:""")
    chain = prompt | llm | str_parser
    answer = chain.invoke({"context": state['context'], "question": state['question']})
    return {**state, "answer": answer, "attempts": state['attempts'] + 1}

def quality_check(state: QAState) -> QAState:
    """Node 3: Score answer quality 1-10."""
    prompt = ChatPromptTemplate.from_template("""
    Rate the quality of this answer (1-10).
    10 = complete, accurate, uses the context well.
    1 = "I don't know" or completely off-topic.
    Return ONLY a number.
    
    Answer: {answer}""")
    chain = prompt | llm | str_parser
    score_str = chain.invoke({"answer": state['answer']})
    try:
        score = int(''.join(filter(str.isdigit, score_str)))
    except:
        score = 5
    return {**state, "quality_score": score}

def rephrase_question(state: QAState) -> QAState:
    """Node 4: Rephrase the question for a better retrieval attempt."""
    prompt = ChatPromptTemplate.from_template(
        "Rephrase this question to be more specific and searchable: {question}"
    )
    chain = prompt | llm | str_parser
    rephrased = chain.invoke({"question": state['question']})
    return {**state, "rephrased_question": rephrased}

def finalize(state: QAState) -> QAState:
    """Node 5: Finalize the answer with sources."""
    if state['quality_score'] >= 6:
        sources_text = ", ".join(state['sources']) if state['sources'] else "general knowledge"
        final = f"{state['answer']}\n\n📎 Sources: {sources_text}"
    else:
        final = "I couldn't find a reliable answer in the available documents."
    return {**state, "final_answer": final}

# ─── Routing ─────────────────────────────────────────────────────
def route_after_quality(state: QAState) -> str:
    if state['quality_score'] >= 6:
        return "good"
    elif state['attempts'] >= 3:
        return "give_up"
    else:
        return "retry"

# ─── Build Graph ──────────────────────────────────────────────────
graph = StateGraph(QAState)

graph.add_node("retrieve", retrieve_context)
graph.add_node("generate", generate_answer)
graph.add_node("check_quality", quality_check)
graph.add_node("rephrase", rephrase_question)
graph.add_node("finalize", finalize)

graph.set_entry_point("retrieve")
graph.add_edge("retrieve", "generate")
graph.add_edge("generate", "check_quality")
graph.add_conditional_edges(
    "check_quality",
    route_after_quality,
    {"good": "finalize", "retry": "rephrase", "give_up": "finalize"}
)
graph.add_edge("rephrase", "retrieve")
graph.add_edge("finalize", END)

app = graph.compile()

# ─── Run ─────────────────────────────────────────────────────────
result = app.invoke({
    "question": "What is the refund policy?",
    "rephrased_question": "",
    "context": "",
    "sources": [],
    "answer": "",
    "quality_score": 0,
    "attempts": 0,
    "final_answer": ""
})

print(result['final_answer'])
```

---

## ✅ Module 1 Mastery Checklist

Mark these off before moving to Module 2:

**LangChain**
- [ ] I can build a `prompt | llm | parser` chain and run it
- [ ] I understand why `ChatOpenAI` and `OpenAI` are different
- [ ] I've used a `PydanticOutputParser` to get structured output
- [ ] I understand when to use `ConversationBufferMemory` vs `ConversationSummaryMemory`
- [ ] I've built at least one tool-using agent

**LangGraph**
- [ ] I can define a TypedDict state
- [ ] I can add nodes and edges to a StateGraph
- [ ] I've built at least one conditional edge that routes differently based on state
- [ ] I understand the LangChain vs LangGraph decision

**LlamaIndex**
- [ ] I've indexed a folder of documents and queried it
- [ ] I understand what chunking is and why chunk size matters
- [ ] I know how to persist an index so I don't re-embed on every run
- [ ] I can print the source file for each answer

**Hugging Face**
- [ ] I've run at least 3 different pipeline tasks
- [ ] I've run a model locally without an API key
- [ ] I understand the tradeoff between open-source and closed models
- [ ] I've compared a small model (GPT-2/TinyLlama) vs a larger model on the same prompt

---

## 🎁 Resources

| Resource | URL |
|---|---|
| LangChain LCEL Docs | https://python.langchain.com/docs/expression_language |
| LangGraph Quickstart | https://langchain-ai.github.io/langgraph/tutorials/introduction |
| LlamaIndex Starter Tutorial | https://docs.llamaindex.ai/en/stable/getting_started/starter_example |
| Hugging Face Course (free) | https://huggingface.co/learn/nlp-course |
| Hugging Face Model Hub | https://huggingface.co/models |
| Open LLM Leaderboard | https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard |

---

## 🔗 Navigation

| | |
|---|---|
| **← Previous** | [Master Index](DOC_00_MASTER_INDEX.md) |
| **Next →** | [Module 2: Model Serving & Inference](DOC_02_MODEL_SERVING.md) |

**In Module 2 you'll learn:** How to take everything you just built and serve it — run models with Ollama locally, expose your chains as REST APIs with FastAPI and LangServe, and deploy to the cloud with Modal and Replicate. The moment your LLM logic goes from "works on my machine" to "works for everyone."

---
*Module 1 of 5 · AI Frameworks Mastery Series · GOLD EDITION v1.0*
