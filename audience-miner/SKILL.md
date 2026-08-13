---
name: audience-miner
description: Build a named prospect list from the people who engage with a competitor's LinkedIn posts. Use when the user gives a LinkedIn profile URL of a competitor, an influencer, or any account whose audience they want, and asks who is worth contacting. Fetches the account's recent posts, the people who commented and reacted under each one, enriches them with title and company, and ranks them on what they wrote and how often they showed up.
---

# Competitor audience miner

The people who engage with a competitor's posts have declared interest in a
topic, in public, with their job title attached. This skill turns that into a
named, ranked list. A written comment carries more intent than a reaction, so
comments drive the ranking and reactions widen the net.

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
claude mcp add --transport http socialrouter https://mcp.socialrouter.io/mcp --scope user --header "Authorization: Bearer sr_live_your_key_here"
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

One LinkedIn profile URL, of the form `https://www.linkedin.com/in/<handle>`.
Pass it to the tools exactly as the user typed it, never rebuild it.

Ask for the target buyer in one line as well, for example "RevOps leads at
Series A to C SaaS". Ranking uses it. If the user does not give one, rank on
engagement frequency alone and say so in the output.

If the user offers a CSV of names, stop and tell them this skill discovers
people, it does not enrich a list they already have.

## Run

The catalogue is live and services change. Read it before spending credits:

`list_services` with `platform: "linkedin"`.

### 1. The competitor's recent posts

`run` with `service: "linkedin/profile.posts"`, `inputs: ["<profile URL>"]`,
`limit: 5`.

Keep each post URL. `linkedin/post.likes` and `linkedin/post.comments` both need
the numeric activity ID, so only posts whose URL contains `activity-<digits>` can
be mined. Skip the others and report how many were skipped.

Each post carries `comments_count` and `total_reactions`. When more than 5 posts
come back, mine the ones with the highest ratio of comments to reactions first. A
post people argued under gathers a more committed audience than one that
collected reflex likes.

### 2. Who commented on each post

One call, every post URL at once:

`run` with `service: "linkedin/post.comments"`, `inputs: [<all post URLs>]`,
`limit: 100`.

`limit` is per post, not per call, so 5 posts at `limit: 100` bills up to 500
records. Start at 50 and raise it only if the list comes back thin.

`options.postedLimit` narrows to a time window, one of `any`, `24h`, `week`,
`month`, `3months`, `6months`, `year`. Use `3months` when the user wants people
who are active now rather than a historical roster.

Each record carries `name`, `title`, `profile_url`, `comment`, `posted_at`,
`likes` on the comment, `replies_count`, and `post_url` to group rows by post.
Replies are off, so every record is a root comment. Drop rows where
`is_post_author` is true, that is the account owner answering in its own thread,
and rows where `author_type` is `company`, a page is not a prospect.

### 3. Who reacted to each post

`run` with `service: "linkedin/post.likes"`, `inputs: [<all post URLs>]`.

Reactions cost more per record than comments and say less. Run this step only
when the comment pass returns fewer than 20 usable people, or when the user
explicitly asks for reach over intent.

Merge both sets into one table keyed by profile URL. For each person, record how
many posts they commented under, how many they reacted to, and the text of their
comments.

### 4. Fill in title and company

Use what `linkedin/post.comments` already returns. Only for people missing a
title, and only for those who reach tier 3 or above:

`run` with `service: "linkedin/profile.info"`, `inputs: [<their profile URLs>]`.

Enrichment is the cheapest call in the chain and the easiest to overspend on.
Never enrich the full list.

## Ranking

A written comment outranks a reaction, and a comment that says something outranks
one that says "Great post".

1. **Commented on 2 or more of the mined posts, and the title matches the target
   buyer.**
2. **Commented once with substance, and the title matches the target buyer.**
   Substance means a question, a disagreement, an experience, or a named tool.
   More than roughly 15 words is a usable proxy.
3. **Commented once, any content, or reacted to 3 or more posts.**
4. **Reacted only.** Baseline, and thin. Include only if the tiers above return
   fewer than 10 people.

Congratulation comments, emoji-only comments and tag-a-colleague comments carry
no intent. Push them down to tier 3 whatever their author's title.

Drop the account owner and anyone whose current company matches theirs. They are
colleagues, not prospects.

## Output

A table: name, title, company, profile URL, tier, the number of posts they
commented under, and their strongest comment quoted in full. That quote is the
opening line of the outreach message, so it is the most useful column of the
table.

Close with the run cost. Add up the cost reported by each response rather than
estimating it, and state the number of people processed and the number kept.

If fewer than 10 people survive tier 3, say the source posts were too thin
instead of padding the list with tier-4 rows. Suggest mining a second account
rather than lowering the bar.

## What this skill does not do

It does not read replies to comments. `scrapeReplies` is off on the provider
side, so a sub-thread conversation stays invisible and only root comments are
ranked.

It does not search for posts by topic. The entry point is one account, so an
audience that never engaged with that account is out of reach.

`post.comments` exists on Facebook, Instagram, Reddit, TikTok and YouTube. The
same chain works there.

## When something fails

The tools return the reason in plain text. Three cases are worth handling
directly:

- **No credits left.** Point to
  [socialrouter.io/dashboard](https://www.socialrouter.io/dashboard) and stop.
- **Unknown service.** The catalogue changed. Read `list_services` again and pick
  from what it returns, never guess a replacement.
- **A post URL is rejected.** It carries no numeric activity ID. Skip that post,
  report it, do not retry.

Surface anything else verbatim rather than reinterpreting it.
