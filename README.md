# ODBC MCP Server

An MCP (Model Context Protocol) server that enables LLM tools like Claude Desktop to query databases via ODBC connections. This server allows Claude and other MCP clients to access, analyze, and generate insights from database data while maintaining security and read-only safeguards.

## Features

- Connect to any ODBC-compatible database
- Support for multiple database connections
- Flexible configuration through config files or Claude Desktop settings
- Read-only safeguards to prevent data modification
- Per-connection table restrictions via `exclude_tables` and `include_tables`
- Easy installation with UV package manager
- Detailed error reporting and logging

## Prerequisites

- Python 3.10 or higher
- UV package manager
- ODBC drivers for your database(s) installed on your system
- For Sage 100 Advanced: ProvideX ODBC driver

## Installation

```bash
git clone https://github.com/tylerstoltz/mcp-odbc.git
cd mcp-odbc
uv venv
.venv\Scripts\activate # On Mac / Linux: source .venv/bin/activate (untested)
uv pip install -e .
```

## Configuration

The server can be configured through:

1. A dedicated config file
2. Environment variables
3. Claude Desktop configuration

### General Configuration Setup

Create a configuration file (`.ini`) with your database connection details:

```ini
[SERVER]
default_connection = my_database
max_rows = 1000
timeout = 30

[my_database]
dsn = MyDatabaseDSN
username = your_username
password = your_password
readonly = true
```

### Restricting Access to Specific Tables

Use `exclude_tables` and `include_tables` on any connection to block access to sensitive tables (accounting, payroll, etc.). Both accept a comma-separated list of exact table names and/or wildcard prefixes ending in `*`. The restriction is enforced at the tool level, before any query reaches the database:

- `list-tables` — restricted tables are omitted from the results
- `get-table-schema` — raises an error if the requested table is restricted
- `execute-query` — rejects any query whose SQL references a restricted table (checked after `FROM`, `JOIN`, `INTO`, or `UPDATE`)

**How the two lists interact:** `exclude_tables` blocks access by default, and `include_tables` re-authorizes specific exceptions on top of that. A table matching both is **allowed** — `include_tables` always wins. Tables matching neither list are allowed by default.

This means:

- `exclude_tables` alone blocks a category, with no exceptions:

  ```ini
  exclude_tables = AP_*, AR_*, GL_*, Payment, TimeCard, TimeCard_Operation
  ```

- `exclude_tables` + `include_tables` blocks a category but carves out specific exceptions within it — e.g. block all accounting tables except a summary view:

  ```ini
  exclude_tables = GL_*, AP_*, AR_*, Payment
  include_tables = GL_Summary
  ```

- `exclude_tables = *` turns the connection into a strict deny-by-default allow-list — only tables listed in `include_tables` are reachable, everything else is blocked:

  ```ini
  exclude_tables = *
  include_tables = Customer, Order_*, Product, Vendor
  ```

- `include_tables` on its own (without `exclude_tables`) does **not** restrict anything — it only guarantees access to the listed tables; everything else stays reachable. Use `exclude_tables = *` if you want a real allow-list.

**Note:** table detection in `execute-query` relies on a regex match rather than a full SQL parser. It reliably catches normal `SELECT`/`FROM`/`JOIN` queries, but is not a substitute for restricting access at the database/ODBC driver level for high-sensitivity data — test with representative queries before relying on it as your only safeguard.

### SQLite Configuration

For SQLite databases with ODBC:

```ini
[SERVER]
default_connection = sqlite_db
max_rows = 1000
timeout = 30

[sqlite_db]
dsn = SQLite_DSN_Name
readonly = true
```

### Sage 100 ProvideX Configuration

ProvideX requires special configuration for compatibility. Use this minimal configuration for best results:

```ini
[SERVER]
default_connection = sage100
max_rows = 1000
timeout = 60

[sage100]
dsn = YOUR_PROVIDEX_DSN
username = your_username
password = your_password
company = YOUR_COMPANY_CODE
readonly = true
```

**Important notes for ProvideX:**
- Use a minimal configuration - adding extra parameters may cause connection issues
- Always set `readonly = true` for safety
- The `company` parameter is required for Sage 100 connections
- Avoid changing connection attributes after connection is established

### Claude Desktop Integration

To configure the server in Claude Desktop:

1. Open or create `claude_desktop_config.json`:
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

2. Add MCP server configuration:

```json
{
  "mcpServers": {
    "odbc": {
      "command": "uv",
      "args": [
        "--directory",
        "C:\\path\\to\\mcp-odbc",
        "run",
        "odbc-mcp-server",
        "--config", 
        "C:\\path\\to\\mcp-odbc\\config\\your_config.ini"
      ]
    }
  }
}
```

## Usage

### Starting the Server Manually

```bash
# Start with default configuration
odbc-mcp-server

# Start with a specific config file
odbc-mcp-server --config path/to/config.ini
```

### Using with Claude Desktop

1. Configure the server in Claude Desktop's config file as shown above
2. Restart Claude Desktop
3. The ODBC tools will automatically appear in the MCP tools list

### Available MCP Tools

The ODBC MCP server provides these tools:

1. **list-connections**: Lists all configured database connections
2. **list-available-dsns**: Lists all available DSNs on the system
3. **test-connection**: Tests a database connection and returns information
4. **list-tables**: Lists all tables in the database
5. **get-table-schema**: Gets schema information for a table
6. **execute-query**: Executes an SQL query and returns results

## Example Queries

Try these prompts in Claude Desktop after connecting the server:

- "Show me all the tables in the database"
- "What's the schema of the Customer table?"
- "Run a query to get the first 10 customers"
- "Find all orders placed in the last 30 days"
- "Analyze the sales data by region and provide insights"

## Troubleshooting

### Connection Issues

If you encounter connection problems:

1. Verify your ODBC drivers are installed correctly
2. Test your DSN using the ODBC Data Source Administrator
3. Check connection parameters in your config file
4. Look for detailed error messages in Claude Desktop logs

### ProvideX-Specific Issues

For Sage 100/ProvideX:
1. Use minimal connection configuration (DSN, username, password, company)
2. Make sure the Company parameter is correct
3. Use the special ProvideX configuration template
4. If you encounter `Driver not capable` errors, check that autocommit is being set at connection time

### Missing Tables

If tables aren't showing up:

1. Verify user permissions for the database account
2. Check if the company code is correct (for Sage 100)
3. Try using fully qualified table names (schema.table)
4. Check whether `exclude_tables` / `include_tables` is configured on the connection — restricted tables are hidden from `list-tables` and blocked from queries by design (see [Restricting Access to Specific Tables](#restricting-access-to-specific-tables))

## License

MIT License - Copyright (c) 2024