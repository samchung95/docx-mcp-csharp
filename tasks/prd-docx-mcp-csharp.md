# PRD: docx-mcp-csharp — .NET OpenXML MCP Server (v2)

## Introduction

Replace the Python/lxml docx-mcp server with a C# MCP server built on the official Microsoft Open XML SDK. Instead of curated high-level tools, expose a generic `execute_openxml` tool that accepts C# code snippets and runs them against the SDK — giving LLMs full access to every OpenXML method via RAG-assisted discovery. Include a built-in `OpenXmlValidator` feedback loop so the LLM can self-correct. Ship as a Docker image for zero-install deployment via stdio transport.

## Goals

- Provide full Open XML SDK coverage for .docx manipulation without needing to pre-build individual tools
- Treat .docx files like codebases — tree view for structure, search/grep for content across all parts
- Enable LLMs to discover and use any SDK method via RAG over OpenXML documentation (roadmap)
- Validate documents after every edit using `OpenXmlValidator` for a compiler-like feedback loop
- Ship as a Docker container so users don't need .NET installed locally
- Support stdio transport for direct integration with Claude Desktop / Claude Code
- Port the knowledge extraction and orchestrator skill from v1

## User Stories

### US-001: Set up .NET MCP server skeleton with stdio transport
**Description:** As a developer, I want a minimal C# MCP server that connects via stdio so that I can add tools incrementally.

**Acceptance Criteria:**
- [ ] New repo/directory `docx-mcp-csharp` with .NET 8+ project
- [ ] Uses the official [MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) (`ModelContextProtocol` NuGet package)
- [ ] Server starts, connects via stdio, and responds to `tools/list`
- [ ] Includes one placeholder tool (`ping`) that returns `"pong"` to verify connectivity
- [ ] `dotnet build` succeeds with no errors
- [ ] CLAUDE.md ported from v1 with updated instructions for C# project

### US-002: Add `execute_openxml` tool — C# snippet execution engine
**Description:** As an LLM, I want to execute arbitrary C# code snippets against the Open XML SDK so that I can perform any document manipulation without needing pre-built tools.

**Acceptance Criteria:**
- [ ] Tool `execute_openxml(file_path: string, code: string, output_path: string?)` registered
- [ ] `code` parameter is a C# snippet that receives a `WordprocessingDocument doc` variable in scope
- [ ] Snippet is compiled and executed using Roslyn scripting API (`Microsoft.CodeAnalysis.CSharp.Scripting`)
- [ ] The following namespaces are auto-imported for every snippet:
  - `DocumentFormat.OpenXml`
  - `DocumentFormat.OpenXml.Packaging`
  - `DocumentFormat.OpenXml.Wordprocessing`
  - `DocumentFormat.OpenXml.Validation`
  - `System.Linq`
- [ ] If `output_path` is null, overwrites the original file
- [ ] Returns JSON: `{ "status": "ok", "output": "<any Console.WriteLine output>", "saved_to": "..." }`
- [ ] On compilation error: returns `{ "status": "error", "errors": ["line 3: CS1002 ; expected", ...] }`
- [ ] On runtime exception: returns `{ "status": "error", "exception": "NullReferenceException: ...", "stackTrace": "..." }`
- [ ] Snippet execution has a 30-second timeout to prevent infinite loops
- [ ] `dotnet build` succeeds

### US-003: Add `validate_document` tool using OpenXmlValidator
**Description:** As an LLM, I want to validate a .docx file after editing so that I can detect and fix structural errors before delivering the document.

**Acceptance Criteria:**
- [ ] Tool `validate_document(file_path: string, file_format: string?)` registered
- [ ] `file_format` defaults to `"Microsoft365"`, also accepts `"Office2019"`, `"Office2016"`, `"Office2013"`, `"Office2010"`, `"Office2007"`
- [ ] Uses `OpenXmlValidator` to validate the document
- [ ] Returns JSON: `{ "valid": true/false, "error_count": N, "errors": [...] }`
- [ ] Each error includes: `description`, `error_type`, `xpath`, `part_uri`
- [ ] Returns error string if file not found or not a valid .docx
- [ ] `dotnet build` succeeds

### US-004: Add `read_document_tldr` tool — high-level document overview
**Description:** As an LLM, I want a concise summary of a document's structure so that I can understand its layout before diving into details — like running `tree` on a codebase.

**Acceptance Criteria:**
- [ ] Tool `read_document_tldr(file_path: string)` registered
- [ ] Returns a tree-like structural overview of the entire document including:
  - **Document parts inventory:** lists all ZIP entries (word/document.xml, word/header1.xml, word/footer1.xml, word/styles.xml, etc.) with sizes
  - **Body outline:** heading hierarchy with nesting (H1 → H2 → H3), paragraph count per section, table locations (row x col dimensions)
  - **Header/footer summary:** for each header/footer part — content type (paragraph, table), text preview (first 80 chars), table dimensions
  - **Style summary:** total style count, top 10 most-used paragraph styles with usage counts
  - **Section properties:** page size, orientation, margins, columns
  - **Metadata:** title, author, created date
- [ ] Output is compact — aims for <2000 tokens for a typical 20-page document
- [ ] Returns error string if file not found or not a valid .docx
- [ ] `dotnet build` succeeds

### US-005: Add `read_document_part` tool — read specific ZIP entries
**Description:** As an LLM, I want to read the contents of a specific part (XML file) inside the docx so that I can inspect details of any component — like reading a specific file in a codebase.

**Acceptance Criteria:**
- [ ] Tool `read_document_part(file_path: string, part_path: string, format: string?)` registered
- [ ] `part_path` is the ZIP entry path (e.g., `"word/document.xml"`, `"word/header1.xml"`, `"word/styles.xml"`)
- [ ] `format` accepts `"structured"` (default) or `"raw"`
- [ ] `"structured"` mode:
  - For `word/document.xml`: returns ordered content blocks (paragraphs with style/text/numbering, tables with cell text, section properties)
  - For `word/header*.xml` / `word/footer*.xml`: returns paragraphs + tables
  - For `word/styles.xml`: returns style definitions with properties
  - For `word/numbering.xml`: returns numbering definitions
  - For other parts: falls back to raw XML
- [ ] `"raw"` mode: returns the raw XML string (pretty-printed)
- [ ] Supports `offset` and `limit` parameters for large parts (line-based, like reading a long file)
- [ ] Returns error if part_path doesn't exist in the ZIP
- [ ] `dotnet build` succeeds

### US-006: Add `search_document` tool — grep across all document parts
**Description:** As an LLM, I want to search for text patterns across all parts of a docx so that I can find content without knowing which XML file it's in — like running grep on a codebase.

**Acceptance Criteria:**
- [ ] Tool `search_document(file_path: string, pattern: string, part_filter: string?, context_lines: int?)` registered
- [ ] `pattern` is a text search string (supports simple wildcards: `*` for any text)
- [ ] Searches across ALL text-bearing parts: document body, headers, footers, comments, footnotes, endnotes
- [ ] Each match includes: `part_path` (which ZIP entry), `location` (paragraph index or XPath), `context` (surrounding text), `style` (paragraph style if applicable)
- [ ] `part_filter` optionally restricts search to specific parts (e.g., `"word/header*"`, `"word/document.xml"`)
- [ ] `context_lines` defaults to 1 — number of surrounding paragraphs to include
- [ ] Returns match count and list of matches
- [ ] Returns empty list (not error) if no matches found
- [ ] `dotnet build` succeeds

### US-007: Add `list_openxml_classes` tool for API discovery
**Description:** As an LLM, I want to browse available OpenXML classes and their methods so that I can write correct code snippets for `execute_openxml`.

**Acceptance Criteria:**
- [ ] Tool `list_openxml_classes(namespace: string?, search: string?)` registered
- [ ] With no args: returns top-level namespaces (`DocumentFormat.OpenXml.Wordprocessing`, etc.)
- [ ] With `namespace`: returns classes in that namespace with brief descriptions
- [ ] With `search`: fuzzy-matches class/method names across all namespaces
- [ ] Each result includes: `full_name`, `kind` (class/method/property/enum), `summary`, `signature`
- [ ] Data extracted from SDK XML doc comments at build time
- [ ] `dotnet build` succeeds

### US-008: Add knowledge extraction tools (port from v1)
**Description:** As an orchestrator agent, I want to extract style catalogs, XML tag inventories, and relationships from documents so that I can build a knowledge base for document generation.

**Acceptance Criteria:**
- [ ] Tool `extract_document_knowledge(file_path: string)` registered
- [ ] Returns structured JSON matching v1's output: styles, xml_tags, relationships, numbering, table_formats, section_properties, metadata
- [ ] Style entries include: name, style_id, type, base_style, is_default, usage_count, paragraph_properties, run_properties
- [ ] Style inheritance is resolved to compute effective properties
- [ ] Tool `save_knowledge(knowledge: string, reference_dir: string)` registered — persists to JSON files
- [ ] Tool `get_knowledge(reference_dir: string, file_name: string?, style_name: string?, tag_name: string?)` registered — retrieves from reference store
- [ ] `dotnet build` succeeds

### US-009: Add orchestrator and extractor skill tools
**Description:** As an orchestrator agent, I want skill instructions that teach me how to coordinate the knowledge extraction and document creation workflow.

**Acceptance Criteria:**
- [ ] Tool `get_orchestrator_skill()` registered — returns markdown workflow instructions
- [ ] Tool `get_extractor_skill()` registered — returns markdown extraction instructions
- [ ] Orchestrator skill references: `read_document_tldr`, `read_document_part`, `search_document`, `extract_document_knowledge`, `save_knowledge`, `list_openxml_classes`, `execute_openxml`, `validate_document`
- [ ] Orchestrator skill includes the explore → plan → execute → validate → fix loop
- [ ] Extractor skill covers full knowledge extraction pipeline
- [ ] `dotnet build` succeeds

### US-010: Create `skill.orchestrator.md` for agent workflow documentation
**Description:** As a user or agent framework, I want a standalone markdown file that describes the complete orchestrator workflow so that any agent system can use the MCP server effectively.

**Acceptance Criteria:**
- [ ] File `skill.orchestrator.md` created at repo root
- [ ] Documents the full workflow: discover → extract knowledge → plan edits → execute → validate → deliver
- [ ] Documents the "docx as codebase" exploration workflow: `read_document_tldr` → `read_document_part` → `search_document`
- [ ] Includes the code generation loop with examples: `list_openxml_classes` → write snippet → `execute_openxml` → `validate_document` → fix
- [ ] Shows example `execute_openxml` snippets for common operations (add paragraph, add table, modify header, set styles)
- [ ] Documents the validation feedback loop with example error → fix cycle
- [ ] Lists all available tools with parameter schemas
- [ ] Includes a "Quick Start" section with a complete end-to-end example

### US-011: Dockerfile and Docker build for containerized deployment
**Description:** As a user, I want to run the MCP server via Docker so that I don't need .NET installed locally.

**Acceptance Criteria:**
- [ ] `Dockerfile` at repo root, multi-stage build:
  - Stage 1: `dotnet/sdk:8.0` — restore, build, publish
  - Stage 2: `dotnet/aspnet:8.0` (or `runtime:8.0`) — minimal runtime image
- [ ] `docker build -t docx-mcp-csharp .` completes successfully
- [ ] `docker run -i --rm docx-mcp-csharp` starts the MCP server on stdio
- [ ] Volume mount `-v /path/to/docs:/work` provides access to host documents
- [ ] Container size is under 500MB
- [ ] Includes `docker-compose.yml` for convenience with pre-configured volume mount
- [ ] README includes Claude Desktop / Claude Code configuration examples:
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

### US-012: Security sandboxing for execute_openxml
**Description:** As a server operator, I want code execution to be sandboxed so that malicious or buggy snippets can't compromise the host system.

**Acceptance Criteria:**
- [ ] Roslyn scripting runs in a restricted context — no `System.IO.File` access outside the working directory
- [ ] No `System.Diagnostics.Process` (can't spawn processes)
- [ ] No `System.Net` (can't make network calls)
- [ ] Allowed assemblies whitelist: only OpenXML SDK, System.Linq, System.Text.Json, System.Collections
- [ ] Memory limit: snippets that allocate >256MB are terminated
- [ ] Timeout: 30 seconds max execution time
- [ ] Docker container runs as non-root user
- [ ] `dotnet build` succeeds

## Functional Requirements

- FR-1: The server must communicate via stdio transport using the MCP protocol
- FR-2: `execute_openxml` must compile and execute C# snippets using Roslyn scripting API
- FR-3: `execute_openxml` must provide `doc` (WordprocessingDocument) as a pre-bound global variable
- FR-4: `validate_document` must use `OpenXmlValidator` and return structured error information with XPaths
- FR-5: `read_document_tldr` must return a compact tree-like overview of the document structure in <2000 tokens
- FR-6: `read_document_part` must support both structured and raw XML output for any ZIP entry
- FR-7: `search_document` must search text across all text-bearing parts (body, headers, footers, comments, footnotes)
- FR-8: `list_openxml_classes` must provide browsable API surface for the Wordprocessing namespace
- FR-9: Knowledge extraction must produce output compatible with v1's JSON schema
- FR-10: All tools must return structured JSON on success and error strings on failure (matching v1 pattern)
- FR-11: The Docker container must support volume mounts for document access
- FR-12: Code execution must be sandboxed with restricted assembly access, memory limits, and timeouts

## Non-Goals

- No Excel (.xlsx) or PowerPoint (.pptx) support in v2 — Word only
- No HTTP/SSE transport — stdio only for now
- No GUI or web interface
- No real-time collaborative editing
- No document format conversion (e.g., .doc to .docx)
- No cloud storage integration (OneDrive, SharePoint)

## Technical Considerations

- **Runtime:** .NET 8 LTS
- **MCP SDK:** `ModelContextProtocol` NuGet package (official C# SDK)
- **OpenXML:** `DocumentFormat.OpenXml` NuGet package (v3.x)
- **Code Execution:** `Microsoft.CodeAnalysis.CSharp.Scripting` (Roslyn) for snippet compilation
- **Serialization:** `System.Text.Json` for all JSON output
- **Docker base:** `mcr.microsoft.com/dotnet/sdk:8.0` (build) + `mcr.microsoft.com/dotnet/runtime:8.0` (runtime)
- **v1 compatibility:** Knowledge JSON schema stays the same so existing reference stores can be reused
- **Docx-as-codebase analogy:** ZIP entries are "files", `read_document_tldr` is `tree`, `read_document_part` is `cat`/`Read`, `search_document` is `grep`

## Success Metrics

- LLM can perform any document manipulation that the OpenXML SDK supports (no "tool not available" dead ends)
- Validation catches 100% of structural errors that would cause Word to show a repair dialog
- `read_document_tldr` gives enough context for the LLM to plan edits without reading the full document
- `search_document` finds content across all parts in a single call (no need to guess which XML file)
- Docker image builds in under 5 minutes, container starts in under 3 seconds
- End-to-end: LLM can clone a template, modify headers/content/tables, validate, and produce a clean .docx in a single conversation

## Roadmap (post-v2)

### RAG knowledge base for Open XML SDK documentation
Add a `search_openxml_docs(query, max_results)` tool that indexes the full Open XML SDK API documentation and common usage patterns. This would enable the LLM to discover methods it hasn't seen before, rather than relying on `list_openxml_classes` alone.

**Considerations:**
- **Index source:** NuGet XML doc comments + Microsoft Learn pages for richer examples
- **Implementation options:** Lucene.NET for full-text search, or pre-computed embeddings with cosine similarity
- **Build-time vs. runtime:** Index should be pre-built during Docker image build
- **Size budget:** Need to balance index quality vs. Docker image size

### SSE/HTTP transport
Add optional HTTP transport for persistent server deployments (multi-user, shared infrastructure).

### Excel/PowerPoint support
Extend to .xlsx and .pptx using the same pattern (execute_openxml already works with SpreadsheetDocument and PresentationDocument).

### Snippet library
Pre-built, tested C# snippets for common operations that the LLM can reference instead of writing from scratch every time.

## Open Questions

- **Roslyn cold start:** First `execute_openxml` call will be slow due to Roslyn compilation. Should we pre-warm the scripting engine at container startup?
- **Snippet caching:** Should compiled snippets be cached for repeated use (e.g., "add paragraph" gets compiled once)?
- **v1 migration:** Should we provide a migration tool that converts v1 reference stores, or just document the compatible schema?
- **search_document regex:** Should pattern support full regex or just simple wildcards? Regex is powerful but error-prone for LLMs.
