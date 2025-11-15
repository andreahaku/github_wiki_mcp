# GitHub Wiki MCP Server

MCP (Model Context Protocol) server for managing GitHub wiki pages programmatically.

## Features

This MCP server exposes the following tools:

- **`write_wiki_page`** - Create or update a wiki page
- **`read_wiki_page`** - Read the content of a wiki page
- **`append_to_wiki_page`** - Append content to an existing page
- **`list_wiki_pages`** - List all wiki pages in the repository
- **`delete_wiki_page`** - Delete a wiki page

## Installation

```bash
# Install dependencies
npm install

# Build the TypeScript project
npm run build
```

## Configuration

### 1. Create a GitHub Personal Access Token

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Select the **`repo`** scope (to access repository wikis)
4. Copy the generated token

### 2. Add the MCP server to Claude Desktop configuration

Edit the configuration file:
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Add this configuration:

```json
{
  "mcpServers": {
    "github-wiki": {
      "command": "node",
      "args": [
        "/absolute/path/to/github_wiki_mcp/dist/index.js"
      ]
    }
  }
}
```

Replace `/absolute/path/to/github_wiki_mcp` with the actual path to the project directory.

### 3. Restart Claude Desktop

Close and reopen Claude Desktop to load the new MCP server.

## Usage

### Example: Write a wiki page about architecture

You can tell Claude:

```
Analyze the codebase using code-analysis-context-mcp to extract
all information about the repository architecture and write
a dedicated wiki page on GitHub called Architecture.
```

Claude will:
1. Use the `code_analysis` tool to get architecture information
2. Generate structured markdown content
3. Use `write_wiki_page` to create the page on GitHub

### Direct Example: Create a wiki page

```
Create a wiki page called "API Documentation" on the repository
myusername/myrepo with this content:

# API Documentation

## Main Endpoints

- GET /api/users
- POST /api/users
- DELETE /api/users/:id
```

### Example: Read an existing page

```
Read the content of the wiki page "Architecture" from the repository
myusername/myrepo
```

### Example: Update an existing page

```
Add a "Deployment" section to the wiki page "Architecture"
with deployment instructions
```

## Tool Reference

### `write_wiki_page`

Creates or completely overwrites a wiki page.

**Parameters:**
- `owner` (string, required): GitHub username or organization
- `repo` (string, required): Repository name
- `token` (string, required): GitHub personal access token
- `pageName` (string, required): Page name (e.g., "Architecture")
- `content` (string, required): Markdown content of the page
- `commitMessage` (string, optional): Custom commit message

**Output:**
```json
{
  "success": true,
  "page": "Architecture.md",
  "url": "https://github.com/owner/repo/wiki/Architecture"
}
```

### `read_wiki_page`

Reads the content of an existing wiki page.

**Parameters:**
- `owner`, `repo`, `token`: Same as above
- `pageName` (string, required): Name of the page to read

**Output:**
```json
{
  "success": true,
  "page": "Architecture.md",
  "content": "# Architecture\n\n..."
}
```

### `append_to_wiki_page`

Appends content to the end of an existing page (or creates it if it doesn't exist).

**Parameters:**
- `owner`, `repo`, `token`: Same as above
- `pageName` (string, required): Page name
- `content` (string, required): Content to append
- `commitMessage` (string, optional): Commit message

### `list_wiki_pages`

Lists all wiki pages in the repository.

**Parameters:**
- `owner`, `repo`, `token`: Same as above

**Output:**
```json
[
  {
    "name": "Architecture",
    "path": "Architecture.md",
    "size": 2048
  },
  {
    "name": "API-Documentation",
    "path": "API-Documentation.md",
    "size": 1024
  }
]
```

### `delete_wiki_page`

Deletes a wiki page.

**Parameters:**
- `owner`, `repo`, `token`: Same as above
- `pageName` (string, required): Name of the page to delete
- `commitMessage` (string, optional): Commit message

## How It Works

The server uses the wiki Git repository (`.wiki.git`) instead of GitHub's REST API:

1. Clones the wiki repository to a temporary directory
2. Performs the requested operations (create/modify/delete `.md` files)
3. Commits and pushes the changes
4. Cleans up the temporary directory

This approach:
- Is more reliable than REST APIs for wikis
- Supports all standard Git operations
- Has no content size limitations
- Works with private repositories (if the token has permissions)

## Integration with Other MCPs

### With `code-analysis-context-mcp`

You can combine this server with your `code-analysis-context-mcp` to:

1. Automatically analyze the codebase
2. Extract documentation, architecture, APIs
3. Automatically write updated wiki pages

**Example workflow:**

```
Analyze the repository using code-analysis-context-mcp,
extract the complete architecture and create three wiki pages:
1. Architecture - overall project structure
2. API - endpoint documentation
3. Database - schema and relationships
```

Claude will use:
- `code_analysis.get_repository_architecture` → analysis
- `github_wiki.write_wiki_page` → Architecture page
- `github_wiki.write_wiki_page` → API page
- `github_wiki.write_wiki_page` → Database page

## Security

- **Never share your GitHub token**: The token has full access to repositories
- **Use tokens with minimum scope**: Only `repo` is necessary
- **Consider using expiring tokens**: For production environments
- **Don't commit the token**: Ensure it's only provided at runtime

## Development

```bash
# Watch mode during development
npm run dev

# Production build
npm run build

# Local installation for testing
npm link
```

## Troubleshooting

### Server doesn't connect

- Verify that the path in `claude_desktop_config.json` is absolute and correct
- Check that `npm run build` has been executed
- Verify Claude Desktop logs

### Authentication error

- Verify that the GitHub token is valid
- Check that the token has the `repo` scope
- Ensure the repository has wiki enabled (Settings → Wiki)

### Wiki doesn't exist

GitHub automatically creates the wiki repository only when the first page is created manually. If the repository has never had a wiki:

1. Go to `https://github.com/owner/repo/wiki`
2. Click "Create the first page"
3. Create a test page
4. Now the MCP server will be able to access the wiki

## License

MIT
