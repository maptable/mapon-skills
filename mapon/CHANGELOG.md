# Changelog

## 2026.09.3

- en: SKILL examples internationalized (Notion / Google Sheets sources) and
  the coordinate contract condensed to the WGS84 requirement. The zh SKILL
  is unchanged.

## 2026.09.2

- CLI: the `vocab` command was removed; the fmt-* builders validate against
  the built-in vocabulary snapshot (offline), and the server's rejection
  error lists the accepted options.

## 2026.09.1

- CLI: network commands ride the MCP endpoint (stateless JSON-RPC; `tools`
  lists the live catalog via `tools/list`, every tool command maps to
  `tools/call`) instead of the removed REST tool mirror. File upload keeps
  its dedicated multipart REST endpoint (`upload`).

## 2026.09.0

First packaged release of the skill: SKILL doc (zh canonical, en mirror),
`references/` docs, and the full CLI (`mapview_cli.py`) covering account,
network, local, and restfile commands.
