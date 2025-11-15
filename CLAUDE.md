# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Model Context Protocol (MCP) server that enables programmatic management of GitHub wiki pages. It exposes 5 tools (`write_wiki_page`, `read_wiki_page`, `append_to_wiki_page`, `list_wiki_pages`, `delete_wiki_page`) that operate by cloning the `.wiki.git` repository, performing Git operations, and pushing changes back.

## Build Commands

**Use pnpm, not npm** - This project uses pnpm as its package manager.

```bash
# Install dependencies
pnpm install

# Build TypeScript to JavaScript
pnpm build

# Watch mode for development
pnpm dev
```

The compiled output goes to `dist/` and includes source maps and type declarations.

## Architecture

### Two-Layer Design

1. **`src/index.ts`** - MCP Server Layer
   - Implements the MCP protocol using `@modelcontextprotocol/sdk`
   - Defines tool schemas with JSON Schema validation
   - Uses the low-level `Server` class (deprecated warning is expected - we need the advanced API)
   - Handles tool routing via `CallToolRequestSchema`
   - All responses are JSON-stringified text content blocks
   - Type casting: Uses `as unknown as TypeName` to bridge MCP's `Record<string, unknown>` to strongly-typed interfaces

2. **`src/wiki-operations.ts`** - Git Operations Layer
   - Contains all business logic for wiki page operations
   - Pattern: Clone → Modify → Commit → Push → Cleanup
   - Uses `simple-git` library (import with named export: `import { simpleGit }`)
   - Every operation creates a temporary directory (`os.tmpdir()`) and cleans it up in `finally` blocks
   - Page name normalization: Converts spaces to hyphens, strips non-alphanumeric chars (except `-` and `_`)

### Critical Implementation Details

**Wiki URL Format**: `https://{token}@github.com/{owner}/{repo}.wiki.git`
- Token is embedded in the URL for authentication
- Wiki repos are separate from main repos (`.wiki.git` suffix)

**Git Workflow**:
1. `cloneWiki()` - Creates temp dir, clones wiki repo
2. Perform file operations (write/read/delete)
3. `git.add()` → `git.commit()` → `git.push('origin', 'master')`
4. `cleanup()` - Always remove temp dir (even on errors)

**Error Handling**:
- `cloneWiki()` cleans up temp dir on clone failure
- All exported functions use `try/finally` with `tmpDir` tracking
- Errors are caught, formatted, and returned as `{success: false, error: string}` with `isError: true`

## Module System

This project uses ES modules (`"type": "module"` in package.json):
- All imports must include `.js` extension even for TypeScript files: `'./wiki-operations.js'`
- TypeScript config: `"module": "Node16"` and `"moduleResolution": "Node16"`
- Compiled to ES2022 target

## Testing the Server

To test locally:
1. Build with `pnpm build`
2. Add to Claude Desktop config at `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):
```json
{
  "mcpServers": {
    "github-wiki": {
      "command": "node",
      "args": ["/absolute/path/to/github_wiki_mcp/dist/index.js"]
    }
  }
}
```
3. Restart Claude Desktop
4. The server runs via stdio transport and logs to stderr

## GitHub Wiki Requirements

Before the server can work with a repository:
- The wiki must be enabled in repo Settings → Features → Wiki
- At least one page must exist (created manually via web UI)
- The GitHub token must have `repo` scope

## Integration Pattern

This server is designed to work alongside `code-analysis-context-mcp` for automated documentation workflows:
1. Code analysis MCP extracts architecture/API/database info
2. This server writes the extracted information as wiki pages
3. Example: "Analyze codebase and create Architecture wiki page" triggers both MCPs
