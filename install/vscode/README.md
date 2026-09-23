# VS Code, Cursor, Windsurf, Zed and other editors

Most editors read an MCP config file. The whole of GoldSeam's is:

```json
{
  "mcpServers": {
    "goldseam": {
      "type": "http",
      "url": "https://goldseam.ksaworks.com/mcp"
    }
  }
}
```

No key, no token, no local process to run — GoldSeam is remote and public. Put that in your editor's
MCP configuration, reload, and the GoldSeam tools appear.
