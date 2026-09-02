---
name: post-finder
description: Pull what an account recently published on LinkedIn, Instagram, Facebook, TikTok or YouTube, and return the posts as text the agent can reason over. Use when the user gives a profile URL or a handle and asks what that account has been saying, what a competitor is posting, or wants a topic tracked over time.
---

# Post finder

An account's own posts are the cheapest public statement of what it cares
about. This skill turns a profile URL into that account's recent posts, in
full text, ready to be read.

## Setup

The skill runs on the SocialRouter MCP server. Check whether the tools
`list_services` and `run` are already available. If they are, skip to Input.

This plugin ships an `.mcp.json` pointing at the hosted server, so there is
nothing to configure. The first call comes back `401`, the client registers
itself, and the browser opens on the consent screen. One approval and the
connection stays.

If the tools are missing, tell the user to run `/mcp`, pick **socialrouter**,
and sign in. New accounts start with free credits, no card and no subscription.

Outside this plugin, the server is added with:

```bash
claude mcp add --transport http socialrouter https://mcp.socialrouter.io/mcp --scope user
```

An API key also works, and is the right answer wherever no browser can open, on
a server or in CI:

```bash
claude mcp add --transport http socialrouter https://mcp.socialrouter.io/mcp --scope user --header "Authorization: Bearer ***"
```

Keys are created at
[socialrouter.io/dashboard/keys](https://www.socialrouter.io/dashboard/keys) and
shown once. A configured header replaces the sign-in: a client holding a key
never sees the `401` that starts the OAuth flow, so leave the header out unless
the user asks for it.

The server is hosted, so nothing is installed and no code is written. Other
clients (Claude Desktop, Cursor, Codex, VS Code, Gemini CLI) are covered at
[docs.socialrouter.io/mcp/introduction](https://docs.socialrouter.io/mcp/introduction).

## Input

One or more profile URLs. Pass them to the tools exactly as the user typed
them, never rebuild one from a handle: a display name is not a handle, and a
guessed URL returns someone else's posts without failing.

When the user gives a bare handle with no platform, ask which platform rather
than trying several. Each attempt is billed.

Ask what the user is looking for in one line, for example "what they say about
pricing". Selection uses it. Without it, return the posts in reverse
chronological order and say so.

## Run

The catalogue is live and services change. Read it before spending credits:

`list_services` with `platform: "<the platform>"`.

### 1. Pick the service for the platform

| Platform | Service | Input |
|---|---|---|
| LinkedIn | `linkedin/profile.posts` | `https://www.linkedin.com/in/<handle>` |
| Instagram | `instagram/profile.posts` | `https://instagram.com/<handle>` |
| Instagram reels | `instagram/profile.reels` | same profile URL |
| Facebook | `facebook/profile.posts` | `https://www.facebook.com/<handle>` |
| TikTok | `tiktok/profile.videos` | `https://www.tiktok.com/@<handle>` |
| YouTube | `youtube/channel.videos` | channel URL |
| YouTube shorts | `youtube/channel.shorts` | channel URL |
| Reddit | `reddit/subreddit.posts` | `https://www.reddit.com/r/<subreddit>` |

Reddit is the exception in that table: the unit is a subreddit, not a person.

X and Bluesky have no per-account posts service, only `post.info` on a single
post URL. When the user asks for an X account's timeline, say that plainly
instead of substituting another platform.

### 2. Pull the posts

One call, every profile URL of the same platform at once:

`run` with `service: "<the service>"`, `inputs: [<all profile URLs>]`,
`limit: 10`.

`limit` is per profile, not per call, so 5 profiles at `limit: 10` bills up to
50 records. Start at 5 for a quick read and raise it only when the user asks
for history.

Profiles on different platforms cannot share a call. Group them by platform,
one `run` per group.

Each record carries the post text, its URL, its date and its engagement counts.
Keep the post URL on every row, it is what makes a claim checkable and what
`audience-miner` consumes next.

### 3. Time windows

Only three services filter at the source, and each spells it differently:

- `instagram/profile.posts` and `instagram/profile.reels`:
  `options.onlyPostsNewerThan`, either `YYYY-MM-DD` or a relative value such as
  `"2 months"`.
- `tiktok/profile.videos`: `options.postedAfter` and `options.postedBefore`,
  both `YYYY-MM-DD`.
- `reddit/subreddit.posts`: `options.postedAfter`, `options.postedBefore`, plus
  `sort` and `time`.

`linkedin/profile.posts`, `facebook/profile.posts` and the YouTube services
take no date window. They return the most recent items, so filter by date after
reading rather than raising `limit` to reach further back.

## What to drop

- Reposts with no added commentary. The account endorsed something, it did not
  say anything.
- Pure promotion, launches and recruiting posts, unless the user asked about
  the company's announcements.

Keep everything else, including short posts. Selection happens against the
user's one-line brief, not against a quality judgement of your own.

## Output

A table, one row per post: date, the post in full or its first three lines when
it runs long, engagement counts, and the post URL.

Above the table, three lines at most naming what the account keeps coming back
to. Support each with a post URL from the table. An observation with no post
behind it does not go in.

When nothing in the run matches the user's brief, say so and show the posts
anyway. An account that never mentions the topic is itself the answer, and
padding it with a loose reading hides that.

Close with the run cost. Add up the cost reported by each response rather than
estimating it, and state the number of profiles read and the number of posts
returned.

## What this skill does not do

It does not read the comments under the posts. That is `audience-miner`, which
takes the post URLs this skill returns.

It does not search by topic across a platform. The entry point is an account.
`linkedin/post.search`, `tiktok/video.search` and `youtube/video.search` are
query services and a different chain.

It does not see private or restricted accounts. What is not public is not
reachable, and no retry changes that.

## When something fails

The tools return the reason in plain text. Three cases are worth handling
directly:

- **No credits left.** Point to
  [socialrouter.io/dashboard](https://www.socialrouter.io/dashboard) and stop.
- **Unknown service.** The catalogue changed. Read `list_services` again and
  pick from what it returns, never guess a replacement.
- **A profile URL is rejected.** The URL does not match the shape that service
  accepts, or the account is gone. Show the accepted format from
  `list_services`, skip that profile, do not retry.

Surface anything else verbatim rather than reinterpreting it.
