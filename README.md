# PDF Router Retriever

A CrewAI-based retrieval system that routes a user question to either a local research-paper PDF or the web, retrieves relevant information, and produces a concise answer.

## System Architecture

```text
User query
    |
    v
Router Agent
    |
    | recommends PDF or WEB
    v
Retriever Agent
    |-----------------------------|
    |                             |
    v                             v
Custom PDFSearchTool          TavilySearchTool
    |                             |
    v                             v
Gemini embeddings             Tavily Web Search API
    |
    v
FAISS vector store
    |
    v
Relevant PDF passages
    |
    v
Concise final answer
```

### Main components

- **CrewAI** coordinates the agents and tasks.
- **Gemini** provides the language model used by the agents and the embedding model used for PDF retrieval.
- **PyPDFLoader** loads the research paper.
- **RecursiveCharacterTextSplitter** divides the PDF into searchable text chunks.
- **FAISS** stores the chunk embeddings and performs similarity search locally.
- **Tavily** searches the web when the question requires current or general web information.
- **python-dotenv** loads API keys from `.env` instead of storing them in the notebook.

## Agent Responsibilities

### Router Agent

The Router Agent analyzes the user’s question and selects exactly one retrieval source:

- **PDF** when the answer should come from the indexed research paper.
- **WEB** when the question requires current, general, or web-based information.

The Router returns a clear recommendation in this form:

```text
Tool: PDF
Reason: The question concerns information contained in the research paper.
```

or:

```text
Tool: WEB
Reason: The question requires current or general web information.
```

The Router does not own retrieval tools. Its responsibility is classification and routing.

### Retriever Agent

The Retriever Agent receives:

- The original user query.
- The Router Agent’s recommendation through task context.

It then:

1. Interprets the Router’s recommendation.
2. Invokes the appropriate PDF or web tool.
3. Uses the retrieved information to answer the original question.
4. Produces a concise response of no more than 100 words using bullet points.

The Retriever owns both retrieval tools because it is responsible for executing the selected retrieval operation and summarizing the result.

## Coordination Flow

The Crew executes two sequential tasks:

1. **Routing task**
   - Receives `{query}`.
   - The Router Agent selects `PDF` or `WEB`.
   - The task output is passed to the next task as context.

2. **Retrieval task**
   - Receives `{query}` directly.
   - Receives the routing task output through `context=[task_route]`.
   - The Retriever Agent invokes the recommended tool.
   - The final answer is returned as `result.raw`.

Conceptually:

```text
query
  -> task_route
  -> Router Agent recommendation
  -> task_retrieve context
  -> Retriever Agent
  -> PDF or web tool
  -> final answer
```

## PDF Retrieval Design

The project uses a custom `PDFSearchTool` based on CrewAI’s `BaseTool`:

1. `PyPDFLoader` loads `transformer_research_paper-dataset.pdf`.
2. `RecursiveCharacterTextSplitter` creates chunks with overlap.
3. `GoogleGenerativeAIEmbeddings` converts each chunk into a vector.
4. `FAISS.from_documents` builds a local vector index.
5. `similarity_search(query, k=3)` returns the three most relevant chunks.
6. The Retriever Agent uses those passages to formulate the answer.

This approach keeps document retrieval local after embedding creation. The search itself does not require OpenAI.

## Configuration and Secrets

API keys are loaded from `.env`:

```text
TAVILY_API_KEY=your-tavily-key
GEMINI_API_KEY=your-gemini-key
```

The notebook loads them with `python-dotenv` and exposes them to the relevant libraries. The `.env` file is excluded through `.gitignore` and must not be committed.

An `.env.example` file may be provided with empty placeholder values, but it must not contain real keys.

## Challenges and Trade-offs

### Built-in `PDFSearchTool` versus a custom tool

The built-in CrewAI `PDFSearchTool` can handle PDF processing internally, which makes it convenient. However, in the installed version it initialized an internal OpenAI-dependent component even when Gemini configuration was supplied. This caused:

```text
The OPENAI_API_KEY environment variable is not set.
```

The custom FAISS tool was selected instead because it provides explicit control over embeddings and avoids an unwanted OpenAI dependency.

**Trade-off:** the custom implementation requires more code and manual setup, but its behavior and provider choice are clear.

### Pydantic and `BaseTool`

CrewAI’s `BaseTool` is Pydantic-based. Internal objects such as the embedding model and FAISS vector store cannot be assigned as undeclared normal attributes. They are declared with `PrivateAttr` instead.

**Trade-off:** this adds a small amount of boilerplate but keeps the custom tool compatible with CrewAI and Pydantic validation.

### Router output is text

The Router does not directly pass a Python enum or a tool object to the Retriever. It returns text through task context. Therefore, the Router output must be unambiguous, such as `Tool: PDF` or `Tool: WEB`.

**Trade-off:** text-based coordination is simple and flexible, but the Retriever must correctly interpret the recommendation. A structured output schema could make this more robust in a future version.

### Retriever has access to both tools

The Router chooses the source, but the Retriever owns both tools and decides which one to invoke based on the Router’s output.

**Trade-off:** this preserves the two-agent design, but the Retriever must follow the routing instruction reliably. A deterministic Python router could provide stricter enforcement, at the cost of reducing the Router Agent’s flexibility.

### Local PDF indexing versus managed retrieval

FAISS provides a lightweight local vector store and is suitable for this single-document project.

**Trade-off:** it is simple and inexpensive, but the index is rebuilt when the notebook setup runs and is not designed for multi-user production workloads. A persistent vector database would be more appropriate for larger collections or deployed applications.

### Environment and dependency setup

The project required packages such as `faiss-cpu`, `pypdf`, `langchain-google-genai`, `crewai`, and `python-dotenv`. Notebook environments can retain stale imports or variables after package changes.

**Operational practice:** restart the notebook kernel after installing packages or changing environment configuration, then run the cells in order.

## Running the Project

1. Create and activate the project virtual environment.
2. Install the packages listed in the notebook.
3. Place `transformer_research_paper-dataset.pdf` in the project directory.
4. Create `.env` with valid Gemini and Tavily keys.
5. Restart the notebook kernel.
6. Run the cells in order:
   - dependencies and environment loading
   - Tavily tool
   - custom FAISS PDF tool
   - Gemini LLM
   - agents
   - tasks
   - Crew execution

## Security Notes

- Never commit `.env`.
- Never hard-code API keys in the notebook.
- Rotate any keys that were previously exposed in source files, screenshots, or notebook output.
- Avoid committing large or proprietary PDF files unless the project requires them.
