# TinyBuild Skills

Claude Code plugin marketplace for [TinyBuild Studio](https://blastoff.tinybuild.workers.dev) products. Install once, get skill packages for every TinyBuild tool you use.

## Install

```
/plugin marketplace add tinybuild/skills
/plugin install <plugin>@tinybuild-skills
/reload-plugins
```

Then authorize the corresponding MCP server when prompted.

## Plugins

### `blastoff`

Submit your product to 30 launch directories from inside Claude Code. The agent reads your saved profile + directory recipes + AI-generated copy variants from the BlastOff MCP server, then drives the browser to fill the form. You review and hit submit.

```
/plugin install blastoff@tinybuild-skills
```

**Required MCP server:** `https://blastoff.tinybuild.workers.dev/mcp`

Connect with: `claude mcp add --transport http blastoff https://blastoff.tinybuild.workers.dev/mcp`

**Also recommended:** a browser-control MCP (Playwright MCP or chrome-devtools MCP) so the agent can drive the form. Without one, the skill falls back to "here's what to fill manually" + the recipe data.

## What's coming

More TinyBuild products will land here as they ship. Each follows the same shape: free OSS plugin + free profile/data tier on the underlying MCP server + paid plan for the heavy AI calls (Slideshot model).

## License

MIT.
