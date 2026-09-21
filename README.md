# mapon-skills

[MapOn](https://maponai.notion.site/MapOn-AI-Help-Center-21f3af4543c580559f5ddce74dc514f0) Agent Skill — turn tabular data (CSV / Excel / Google Sheets / Notion, or any spreadsheet) into a **shareable interactive map**: choropleth, point, icon, and heatmap layers with built-in geocoding and region matching. Maps are shared via a single link — viewers need no account. Region coverage is worldwide: country-level boundaries everywhere, with per-country admin hierarchies (province / city / district) where published.

## Install

Requires Python 3.9+ (standard library only, no dependencies to install).

**skills.sh** (recommended):

```bash
npx skills add maptable/mapon-skills
```

**Manual install**: copy the whole [`mapon/`](mapon/) directory into your agent's skills directory.

Global install (works across projects):

```bash
cp -r mapon/ ~/.trae/skills/     # Trae
cp -r mapon/ ~/.zcode/skills/    # ZCode
cp -r mapon/ ~/.claude/skills/   # Claude Code
```

Project-local install (current project only):

```bash
cp -r mapon/ .codebuddy/skills/  # CodeBuddy
cp -r mapon/ .agents/skills/     # any other agent with an Agent Skills compatible directory
```

## Usage

Once installed, just ask your agent to build a map in conversation — e.g. "turn this spreadsheet into a choropleth by province" or "plot these addresses as a heatmap". Connection, login, geocoding, and region matching are all handled automatically by the agent; you get back a shareable map link.

## License

[Apache-2.0](LICENSE)
