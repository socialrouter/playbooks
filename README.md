# @socialrouter/playbooks

Go-to-market playbooks for coding agents. Each one drives the [SocialRouter](https://www.socialrouter.io) MCP server to pull real social data into the agent's context, then does something useful with it.

One playbook, one plugin. Install the one you came for, ignore the rest.

## Install

Two steps, whichever playbook you came for. Add the marketplace once:

```
/plugin marketplace add socialrouter/playbooks
```

Then install the one you want, by name:

```
/plugin install post-finder@socialrouter
/plugin install audience-miner@socialrouter
```

The names are in the table below. Installing one does not install the others.

Then restart the session, or run `/reload-plugins`.

Each playbook declares the hosted SocialRouter MCP server, so there is no config file to write and no key to paste. The first call opens your browser on a consent screen; approve it once and you are connected. If nothing happens, run `/mcp`, pick **socialrouter**, and sign in there.

New accounts start with free credits. No card, no subscription.

## Playbooks

| Plugin | Install | What it does |
| --- | --- | --- |
| [`post-finder`](post-finder/SKILL.md) | `/plugin install post-finder@socialrouter` | Turns a profile URL into that account's recent posts in full text, on LinkedIn, Instagram, Facebook, TikTok, YouTube and Reddit. |
| [`audience-miner`](audience-miner/SKILL.md) | `/plugin install audience-miner@socialrouter` | Turns a competitor's LinkedIn audience into a named prospect list, ranked in four tiers on what people wrote and how often they showed up. |

`post-finder` returns the post URLs that `audience-miner` takes as input, so the two chain. Each also works on its own.

## Somewhere without a browser

On a server, in CI, or in any client that cannot open a consent screen, authenticate with an API key instead. Create one at [socialrouter.io/dashboard/keys](https://www.socialrouter.io/dashboard/keys) (it is shown once) and add the server by hand:

```bash
claude mcp add --transport http socialrouter https://mcp.socialrouter.io/mcp --scope user --header "Authorization: Bearer sr_live_your_key_here"
```

A configured header replaces the sign-in: a client holding a key never sees the response that starts the OAuth flow.

## Other clients

The playbooks are Claude Code plugins, but the MCP server behind them works anywhere. Claude Desktop, Cursor, Codex, VS Code and Gemini CLI are covered at [docs.socialrouter.io/mcp/introduction](https://docs.socialrouter.io/mcp/introduction).

## License

MIT
