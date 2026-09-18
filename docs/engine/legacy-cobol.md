# Legacy / COBOL Support

antcrew has first-class support for organisations running COBOL on IBM AS/400 (iSeries) mainframes. You can index COBOL programs as documentation, add AI capabilities without rewriting them, connect to DB2 for i databases, and translate programs to Python/Java/Go.

```bash
pip install "antcrew[docs,legacy]"
```

---

## What's included

| Feature | Package | CLI command |
|---|---|---|
| Parse `.cbl`, `.cob`, `.cpy` files into the doc index | `antcrew[docs]` | `antcrew engine --docs-dir` |
| `org_type: legacy` in schema — enables COBOL-aware routing | `antcrew[docs]` | — |
| `AS400Connector` — query DB2 for i tables | `antcrew[legacy]` | — |
| `COBOLAugment` — add AI to COBOL without rewriting | `antcrew[legacy]` | `antcrew augment-cobol` |
| COBOL → Python/Java/Go structural translation | `polytranslate` | — |

---

## COBOL documentation indexing

### Supported file types

| Extension | Type | Parser |
|---|---|---|
| `.cbl`, `.cob` | COBOL program | `CobolParser` |
| `.cpy`, `.copy` | Copybook | `CobolParser` (copybook mode) |

The COBOL parser handles both **fixed-format** (cols 1-6 sequence, col 7 indicator) and **free-format** COBOL. It extracts:

- `PROGRAM-ID`, `AUTHOR`, `DATE-WRITTEN`
- Data items: WORKING-STORAGE, FILE, LINKAGE sections — level, name, PIC clause
- Paragraph names from PROCEDURE DIVISION
- `COPY` and `CALL` statements
- Division structure

Each parsed program becomes a human-readable summary + structured metadata in the doc index, available for semantic search and agent context injection.

### Schema configuration for legacy organisations

Set `org_type: legacy` and configure `if_legacy.cobol_support` in your schema YAML:

```yaml
documentation_schema:
  org_name: Acme Corp
  org_type: legacy          # legacy | microservices | mixed

  if_legacy:
    cobol_support:
      enabled: true
      as400_connection: false   # true if you use AS400Connector
      copybook_parsing: true    # parse .cpy/.copy as COBOL
      batch_job_docs: true      # treat COBOL programs as batch-job docs

  document_types:
    - id: cobol_program
      name: COBOL Batch Program
      category: technical
      parser: cobol

    - id: copybook
      name: COBOL Copybook
      category: technical
      parser: cobol

    - id: jcl_procedure
      name: JCL Procedure
      category: operational
      parser: text

  path_rules:
    - prefix: "cobol/"
      doc_type: cobol_program
    - prefix: "copybooks/"
      doc_type: copybook
    - prefix: "jcl/"
      doc_type: jcl_procedure

  query_hints:
    - pattern: "batch|orden|order processing|ORDPRC"
      doc_types: [cobol_program]
    - pattern: "copybook|estructura|field layout"
      doc_types: [copybook]
```

```python
from antcrew_engine.documentation.schema import DocumentationSchemaRegistry

reg = DocumentationSchemaRegistry()
reg.load_from_file("schema.yaml")

print(reg.is_legacy)       # True
print(reg.cobol_support)   # CobolSupport(enabled=True, as400_connection=False, ...)
```

---

## AS400Connector

Read-only access to IBM DB2 for i (AS/400) databases via ODBC. Requires `pip install antcrew[legacy]` which adds `pyodbc`.

### Prerequisites

- IBM iSeries Access ODBC driver installed on the machine running antcrew
- ODBC DSN configured for your AS/400 system, **or** a full connection string

### Usage

```python
from antcrew.integrations.as400 import AS400Connector

conn = AS400Connector(
    dsn="MY_AS400_DSN",       # pre-configured ODBC DSN
    username="MYUSER",
    password="MYPASS",
    library="ORDLIB",         # default schema/library
)

# Column metadata for a table
schema = conn.get_schema("ORDLIB.ORDHDRF")
for col in schema.columns:
    print(f"{col.name:30} {col.data_type}({col.length})")

# COBOL copybook layout derived from the schema
print(conn.get_copybook("ORDLIB.ORDHDRF"))

# First N rows as a list of dicts (read-only)
rows = conn.sample_data("ORDLIB.ORDHDRF", limit=5)
```

```python
# Context manager — closes connection automatically
with AS400Connector(dsn="MY_AS400_DSN", username="U", password="P") as conn:
    schema = conn.get_schema("CUSTLIB.CUSTMST")
```

### Connection string form

```python
conn = AS400Connector(
    connection_string="Driver={IBM i Access ODBC Driver};System=192.168.1.10;",
    username="MYUSER",
    password="MYPASS",
)
```

### DB2-to-COBOL PIC mapping

`get_copybook()` maps DB2 column types to COBOL PIC clauses:

| DB2 type | PIC clause |
|---|---|
| `CHAR(n)` / `VARCHAR(n)` | `X(n)` |
| `DECIMAL(p,s)` | `9(p-s)V9(s)` |
| `INTEGER` | `S9(9) COMP-4` |
| `SMALLINT` | `S9(4) COMP-4` |
| `BIGINT` | `S9(18) COMP-4` |
| `DATE` | `X(10)` |
| `TIMESTAMP` | `X(26)` |

---

## COBOLAugment — add AI without rewriting

`COBOLAugment` analyses a COBOL program and generates three artefacts:

1. **Python AI wrapper** — a class with `process()` + AI enrichment hooks, bridging to your COBOL via REST, MQ, or batch file exchange
2. **COBOL bridge caller** — a COBOL program (`PROGRAM-ID. XXXAI`) that calls `ANTCREW-SEND` to invoke the Python service
3. **Deployment guide** — step-by-step instructions for three integration patterns (REST adapter, IBM MQ, batch file exchange)

### CLI

```bash
antcrew augment-cobol ORDPRC.cbl --requirement "Add ML fraud scoring to order processing"
```

Options:

| Flag | Description |
|---|---|
| `--requirement` / `-r` | What AI capability to add (required) |
| `--model` / `-m` | LLM to use for wrapper generation. Omit for a static template. |
| `--output` / `-o` | Output directory. Defaults to the COBOL file's directory. |
| `--guide` | Also print the deployment guide to stdout. |
| `--dry-run` | Preview output without writing files. |

### Python API

```python
from antcrew.augment.cobol import COBOLAugment

aug = COBOLAugment(llm=my_llm)   # llm is optional — omit for static templates
result = aug.augment("ORDPRC.cbl", requirement="Add ML fraud scoring")

print(result.python_wrapper)     # Python class with AI hooks
print(result.cobol_caller)       # COBOL bridge program
print(result.deployment_guide)   # Markdown deployment instructions

# Write to disk
from pathlib import Path
Path("ordprc_ai.py").write_text(result.python_wrapper)
Path("ORDPRCAI.cbl").write_text(result.cobol_caller)
Path("ORDPRC_deployment_guide.md").write_text(result.deployment_guide)
```

### Analysis only

Use `COBOLAnalyzer` directly if you only need the structural analysis:

```python
from antcrew.augment.cobol import COBOLAnalyzer

analysis = COBOLAnalyzer().analyze("ORDPRC.cbl")

print(analysis.program_id)         # "ORDPRC"
print(analysis.paragraphs)         # ["MAIN-PARA", "VALIDATE-ORDER", ...]
print(analysis.working_storage_fields)  # [DataItem(level=1, name="WS-ORDER-ID", ...)]
print(analysis.external_calls)     # [ExternalCall(kind="CALL", target="VALDATE"), ...]
print(analysis.summary())          # human-readable overview
```

### Integration patterns

The generated deployment guide covers three patterns:

**Option A — REST adapter (recommended for AS/400)**

```
COBOL ORDPRC  →  ORDPRCAI.cbl  →  HTTP POST /process  →  OrdprcAI (Python)
```

The COBOL bridge calls `ANTCREW-SEND` (a C or RPG subprogram that wraps IBM HTTP Client). The Python service runs on any Linux/Windows server accessible from the AS/400.

**Option B — IBM MQ trigger**

The COBOL program writes to an MQ queue instead of calling HTTP. The Python service listens via `ibm-mq`. Reliable, asynchronous, no synchronous coupling.

**Option C — Batch file exchange**

COBOL writes input to a flat file; Python reads and processes it overnight. Suitable for batch jobs with no real-time requirement.

---

## polytranslate — full structural translation

For teams that want to migrate COBOL programs to Python/Java/Go rather than augmenting them:

```bash
pip install polytranslate
```

```python
from polytranslate.languages.cobol import CobolParser
from polytranslate.targets.python import PythonGenerator

ast = CobolParser().parse_file("ORDPRC.cbl")
files = PythonGenerator().generate(ast)

for f in files:
    f.write("./output")
    # Produces: ordprc_model.py, ordprc_logic.py, ordprc_runner.py
```

The translation is **structural** — every data item and paragraph is mapped to an equivalent Python construct, producing a working skeleton for developer review. It does not evaluate COBOL expressions or simulate COBOL runtime behaviour.

See the [polytranslate repository](https://github.com/iagop03/polytranslate) for the full pipeline architecture and how to add new target languages.

---

## Choosing the right tool

| Scenario | Recommended tool |
|---|---|
| Index COBOL programs in the doc system | `antcrew[docs]` + `org_type: legacy` |
| Query DB2 for i schema from Python | `AS400Connector` |
| Keep COBOL running, add AI logic alongside | `antcrew augment-cobol` |
| Migrate COBOL to Python/Java/Go | `polytranslate` |
| Full AI-driven rewrite with agent team | `antcrew run --team CodeMigrationTeam` |
