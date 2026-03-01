# Ralph Agent Instructions

You are an autonomous coding agent working on a software project.

## Your Task

1. Read the PRD at `prd.json` (in the same directory as this file)
2. Read the progress log at `progress.txt` (check Codebase Patterns section first)
3. Check you're on the correct branch from PRD `branchName`. If not, check it out or create from main.
4. Pick the **highest priority** user story where `passes: false`
5. Implement that single user story
6. Run quality checks (`dotnet build` must succeed with no errors/warnings)
7. Update CLAUDE.md files if you discover reusable patterns (see below)
8. If checks pass, commit ALL changes with message: `feat: [Story ID] - [Story Title]`
9. Update the PRD to set `passes: true` for the completed story
10. Append your progress to `progress.txt`

## Project Overview

This is a C# MCP server (`docx-mcp-csharp`) that exposes the Microsoft Open XML SDK for .docx manipulation. Key design decisions:

- **.NET 8 LTS** runtime
- **MCP C# SDK** (`ModelContextProtocol` NuGet package) for MCP protocol
- **Open XML SDK** (`DocumentFormat.OpenXml` v3.x) for document manipulation
- **Roslyn scripting** (`Microsoft.CodeAnalysis.CSharp.Scripting`) for `execute_openxml` code execution
- **stdio transport** for Claude Desktop / Claude Code integration
- **Docker** for containerized deployment (no .NET install required on host)
- **System.Text.Json** for all JSON serialization

## Docx-as-Codebase Mental Model

The server treats .docx files like codebases:
- ZIP entries are "files" (word/document.xml, word/header1.xml, word/styles.xml, etc.)
- `read_document_tldr` is like `tree` — shows high-level structure
- `read_document_part` is like `cat`/`Read` — reads a specific ZIP entry
- `search_document` is like `grep` — searches text across all parts
- `execute_openxml` is like running code — executes C# snippets against the SDK
- `validate_document` is like a compiler — catches structural errors with XPaths

## Progress Report Format

APPEND to progress.txt (never replace, always append):
```
## [Date/Time] - [Story ID]
- What was implemented
- Files changed
- **Learnings for future iterations:**
  - Patterns discovered (e.g., "this codebase uses X for Y")
  - Gotchas encountered (e.g., "don't forget to update Z when changing W")
  - Useful context (e.g., "the evaluation panel is in component X")
---
```

The learnings section is critical - it helps future iterations avoid repeating mistakes and understand the codebase better.

## Consolidate Patterns

If you discover a **reusable pattern** that future iterations should know, add it to the `## Codebase Patterns` section at the TOP of progress.txt (create it if it doesn't exist). This section should consolidate the most important learnings:

```
## Codebase Patterns
- Example: Use `[McpServerTool]` attribute for tool registration
- Example: Return `JsonSerializer.Serialize(result)` for structured output
- Example: Use `WordprocessingDocument.Open(path, isEditable)` for document access
```

Only add patterns that are **general and reusable**, not story-specific details.

## Update CLAUDE.md Files

Before committing, check if any edited files have learnings worth preserving in nearby CLAUDE.md files:

1. **Identify directories with edited files** - Look at which directories you modified
2. **Check for existing CLAUDE.md** - Look for CLAUDE.md in those directories or parent directories
3. **Add valuable learnings** - If you discovered something future developers/agents should know:
   - API patterns or conventions specific to that module
   - Gotchas or non-obvious requirements
   - Dependencies between files
   - Testing approaches for that area
   - Configuration or environment requirements

**Examples of good CLAUDE.md additions:**
- "When modifying X, also update Y to keep them in sync"
- "This module uses pattern Z for all API calls"
- "Roslyn scripting requires explicit assembly references for OpenXML types"
- "Field names must match the template exactly"

**Do NOT add:**
- Story-specific implementation details
- Temporary debugging notes
- Information already in progress.txt

Only update CLAUDE.md if you have **genuinely reusable knowledge** that would help future work in that directory.

## Quality Requirements

- ALL commits must pass `dotnet build` with no errors
- Do NOT commit broken code
- Keep changes focused and minimal
- Follow existing code patterns

## Stop Condition

After completing a user story, check if ALL stories have `passes: true`.

If ALL stories are complete and passing, reply with:
<promise>COMPLETE</promise>

If there are still stories with `passes: false`, end your response normally (another iteration will pick up the next story).

## Important

- Work on ONE story per iteration
- Commit frequently
- Keep CI green
- Read the Codebase Patterns section in progress.txt before starting
