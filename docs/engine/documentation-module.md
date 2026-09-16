# Documentation Module

The Documentation Module lets the engine index your project's existing documentation and inject relevant context into every capability's LLM prompt — so the code generator knows your auth requirements, the architect sees your ADRs, and the bug fixer reads your error-handling policy.

```bash
pip install "antcrew[docs]"   # adds python-docx, pypdf, networkx, gitpython, boto3
```

---

## Quick start

### CLI

```bash
antcrew engine "Add user authentication" \
  --schema ./docs/schema.yaml \
  --docs-dir ./docs \
  --tech Python --tech FastAPI \
  --output ./my-api
```

`--schema` points to your [schema file](#schema-file) (optional — runs with defaults if omitted).  
`--docs-dir` is the directory to index. The engine uploads all supported files, then injects relevant snippets into each capability before it calls the LLM.

### Python API

```python
from antcrew_engine.documentation import DocumentationManager

mgr = DocumentationManager(schema_path="docs/schema.yaml")
mgr.bulk_upload("./docs")

results = mgr.search("JWT authentication", top_k=5)
context = mgr.get_context_for_agent("BackendDev", "login flow")
```

---

## Schema file

A schema.yaml describes your doc types and tells the engine which type matters for which capability.

```yaml
documentation_schema:
  org_name: Acme Corp

  document_types:
    - id: srs
      name: Software Requirements Specification
      category: functional
      purpose: Define what the system must do
      parser: markdown          # markdown | text | docx | pdf | jira
      related_to: [design, adr]

    - id: design
      name: Technical Design
      category: technical
      purpose: Architecture and implementation decisions
      parser: markdown
      related_to: [srs]

    - id: adr
      name: Architecture Decision Record
      category: technical
      purpose: Rationale for major decisions
      parser: markdown

  agent_hints:
    Architect:
      - "For requirements: search Software Requirements Specification (srs)"
      - "For prior decisions: search Architecture Decision Record (adr)"
    CodeGenerator:
      - "For requirements: search Software Requirements Specification (srs)"
      - "For design details: search Technical Design (design)"
```

The engine reads `agent_hints` to decide which doc types to query per capability. If no hints match the capability class name, it falls back to a generic search across all indexed documents.

### File naming convention

Files can auto-detect their type using the pattern `{doc_type}.{project}.{ext}`:

```
docs/
  srs.claims-processing.md      → doc_type=srs, project=claims-processing
  design.auth-service.md        → doc_type=design, project=auth-service
  adr.0042-jwt-choice.md        → doc_type=adr
```

Files that don't follow the convention are classified by extension (`.md → markdown`, `.docx → docx`, `.pdf → pdf`, `.json → jira`).

---

## Supported parsers

| Parser | Extensions | Notes |
|--------|-----------|-------|
| `markdown` | `.md`, `.markdown` | Extracts headings as sections, counts words |
| `text` | `.txt` | Plain text; treats the whole file as one section |
| `docx` | `.docx`, `.doc` | Requires `python-docx` |
| `pdf` | `.pdf` | Requires `pypdf`; extracts page text |
| `jira` | `.json` | Parses Jira ticket JSON exports; falls back to plain text |

---

## Storage backends

| Backend | Config | Notes |
|---------|--------|-------|
| `local` *(default)* | `path: ./documentation` | Files stored in a local directory |
| `git` | `path: ./documentation` | Commit each document as a git blob; requires `gitpython` |
| `s3` | `bucket`, `prefix`, `region` | Requires `boto3` |

```python
mgr = DocumentationManager(
    schema_path="schema.yaml",
    storage_type="s3",
    storage_config={"bucket": "my-docs", "prefix": "v2/", "region": "eu-west-1"},
)
```

---

## Search and semantic index

By default the index uses **keyword search** (TF-IDF-like term overlap, no dependencies). Install ChromaDB to upgrade to **semantic search**:

```bash
pip install chromadb
```

When ChromaDB is present, documents are embedded on upload and searched by cosine similarity. The upgrade is transparent — no code changes needed.

```python
results = mgr.search("JWT authentication requirements", top_k=5)
# results: list of {"id": ..., "content": ..., "metadata": ..., "score": ...}

# Filter by doc type or category
results = mgr.search_by_type("login flow", "srs", top_k=3)
results = mgr.search_by_category("authentication", "functional", top_k=5)
```

---

## Knowledge graph

The module maintains a relation graph between documents based on `related_to` in the schema. Install NetworkX to enable full graph algorithms (cycle detection, subgraph traversal):

```bash
pip install networkx
```

Without NetworkX, the graph falls back to an adjacency dict that supports `get_neighbors` and basic traversal.

```python
related = mgr.get_related_documents("srs/my-spec.md")
# → [{"id": "design/my-design.md", "doc_type": "design", ...}]
```

---

## BaseExecutor integration

All built-in capabilities inherit from `BaseExecutor` and gain two methods:

```python
executor.set_documentation(doc_manager)   # attach the manager
context_str = executor._doc_context("JWT login flow", max_chars=3000)
```

`_doc_context()` returns a formatted string ready to prepend to any LLM prompt:

```python
# Inside a custom executor's _run() method:
prompt = f"{self._doc_context(goal.description)}\n\nTask: {goal.description}"
result = self._call(system_prompt, prompt)
```

Returns `""` when no manager is attached or no relevant documents are found — safe to always call.

---

## Custom executor example

```python
from antcrew_engine.capabilities.base import BaseExecutor
from antcrew_engine.engine import CapabilityDescriptor, CapabilityResult, EMPTY_DELTA

class SecurityChecker(BaseExecutor):
    descriptor = CapabilityDescriptor(
        name="security_checker",
        description="Reviews code for security issues against project security policy.",
        conditions_produced=["security_verified"],
    )

    def _run(self, store, goal):
        doc_context = self._doc_context("security requirements threat model OWASP")
        system = "You are a security reviewer."
        user = f"{doc_context}\n\nReview the following code for security issues:\n\n{goal.description}"
        review = self._call(system, user)
        # ... process review and return CapabilityResult
```

---

## Statistics and validation

```python
stats = mgr.get_statistics()
# {"total_documents": 12, "by_type": {"srs": 3, "design": 5, ...}, "by_category": {...}, "index": {...}}

report = mgr.validate_against_schema()
# {"present_types": ["srs", "design"], "missing_types": ["adr"], "errors": []}
```

---

## Optional dependencies summary

```toml
# Install only what you need
pip install "antcrew[docs]"          # all doc-related deps
pip install "antcrew[memory]"        # ChromaDB for semantic search
pip install python-docx              # .docx parsing
pip install pypdf                    # .pdf parsing
pip install networkx                 # full knowledge graph
pip install gitpython                # git storage backend
pip install boto3                    # S3 storage backend
```
