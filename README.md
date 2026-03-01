# docx-mcp-csharp

A .NET MCP server for .docx document manipulation, built on the official Microsoft Open XML SDK. Designed for LLM agents — provides document exploration, code execution, and validation tools over the MCP protocol via stdio transport, shipped as a Docker container.

## Architecture

```
LLM (Claude, etc.)
  │
  ├── read_document_tldr    ← "tree" — structural overview
  ├── read_document_part    ← "cat"  — read specific XML parts
  ├── search_document       ← "grep" — search across all parts
  ├── list_openxml_classes  ← API discovery for code writing
  ├── execute_openxml       ← run C# snippets against OpenXML SDK
  └── validate_document     ← "compiler" — catch errors with XPaths
        │
        ▼
  Open XML SDK (.NET) → .docx file
```

The core idea: instead of pre-building tools for every operation, expose the full OpenXML SDK through a generic code execution engine. The LLM explores the document structure, discovers SDK methods, writes C# snippets, executes them, and validates the result — with a compiler-like feedback loop for self-correction.

## Status

**In development.** See [tasks/prd-docx-mcp-csharp.md](tasks/prd-docx-mcp-csharp.md) for the full PRD with 12 user stories.

## Usage (planned)

```json
{
  "mcpServers": {
    "docx-mcp": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "-v", "${workspaceFolder}:/work", "docx-mcp-csharp"]
    }
  }
}
```

No .NET installation required — everything runs inside the Docker container.

## Relationship to v1 (docx-mcp)

This is a ground-up rewrite of [docx-mcp](https://github.com/samchung95/docx-mcp), the Python/lxml-based MCP server.

| | v1 (docx-mcp) | v2 (docx-mcp-csharp) |
|---|---|---|
| **Language** | Python | C# / .NET 8 |
| **OOXML library** | lxml + zipfile (manual XML) | Microsoft Open XML SDK (official) |
| **Tool design** | ~20 curated high-level tools | Generic `execute_openxml` + exploration tools |
| **Validation** | Custom `validate_document` (checks borders, styles) | `OpenXmlValidator` (full OOXML schema validation) |
| **Doc exploration** | `read_document_text`, `list_headings`, `read_tables` | Docx-as-codebase: `tldr` / `read_part` / `search` |
| **Coverage** | Limited to what tools were built for | Full SDK — any operation the SDK supports |
| **Deployment** | `pip install` / `uv run` | Docker container (zero-install) |
| **Transport** | stdio | stdio (SSE on roadmap) |

### What carried over from v1

- **Knowledge extraction pipeline** — `extract_document_knowledge`, `save_knowledge`, `get_knowledge` with the same JSON schema, so existing reference stores are compatible
- **Orchestrator/extractor skills** — agent workflow instructions, updated for v2's tool set
- **Error handling pattern** — structured JSON on success, error strings on failure

### When to reference v1

- **Knowledge JSON schema** — v2 produces the same format, so check v1's `extract_document_knowledge` output if you need the schema definition
- **Style guide generation** — v1 has `generate_style_guide` and `save_style_guide` tools that may be ported later
- **Replace text logic** — v1's cross-run text replacement algorithm (handles text spanning multiple XML runs) is well-tested and may inform v2's approach

v1 repo: https://github.com/samchung95/docx-mcp

## Tech Stack

- **.NET 8 LTS**
- **[MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk)** — `ModelContextProtocol` NuGet package
- **[Open XML SDK](https://github.com/dotnet/Open-XML-SDK)** — `DocumentFormat.OpenXml` v3.x
- **[Roslyn Scripting](https://github.com/dotnet/roslyn)** — `Microsoft.CodeAnalysis.CSharp.Scripting` for runtime C# execution
- **Docker** — multi-stage build for zero-install deployment

## Development Notes

**Primary dev machine is Windows.** This causes some known friction:

- **Docker volume permissions:** Windows file ownership doesn't map cleanly to Linux container UIDs. Files created inside the container may appear as root-owned on the host, or the container may fail to write to mounted volumes. Workarounds:
  - Use `--user $(id -u):$(id -g)` when running containers (doesn't apply on Windows natively)
  - Set permissive permissions on the mounted directory
  - Use a named volume instead of a bind mount for working files
- **File I/O path separators:** Windows uses `\`, .NET inside Docker uses `/`. Always use `Path.Combine()` or forward slashes in tool parameters — never hardcode `\`.
- **File locking:** Windows locks open files more aggressively than Linux. If a .docx is open in Word, the MCP server (or Docker container) may fail to read/write it. Close Word before running tools against the same file.
- **Line endings:** Git may auto-convert `LF` to `CRLF` on checkout. The `.gitattributes` or `git config core.autocrlf` setting matters for Dockerfile and shell scripts — these must stay `LF`.
- **Docker Desktop required:** Docker commands assume Docker Desktop for Windows is installed and running with WSL 2 backend.

## License

MIT
