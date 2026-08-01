---
name: schedulala
description: >
  Schedule and publish social media posts to 12 platforms (Twitter/X, Instagram,
  TikTok, LinkedIn, YouTube, Facebook, Threads, Bluesky, Pinterest, Mastodon,
  Telegram, Google Business Profile) with Schedulala. Use when the user wants to
  create, schedule, preview, or bulk-plan social posts, post photos/videos/PDFs,
  auto-post a first comment (link-in-first-comment), check post status or
  analytics, reply to or moderate comments, monitor keywords, or repurpose
  YouTube videos. Works through the hosted Schedulala MCP
  connector (claude.ai / ChatGPT, with interactive post-preview, analytics,
  calendar, inbox, and media-upload widgets), the local @schedulala/mcp-server
  (Claude Desktop, Claude Code, Cursor), or the schedulala CLI. Encodes the
  per-platform rules: TikTok privacy + postMode, Instagram media requirements,
  YouTube titles, Pinterest boards, character limits, and the safe workflow
  draft → validate/preview → confirm → publish → verify.
license: MIT
metadata:
  version: "1.0.0"
  docs: https://schedulala.com/developers/docs
---

# Schedulala

Schedulala schedules and publishes social media posts across 12 platforms, with
analytics, engagement (comments/replies), and social listening. Full REST API,
CLI, and connector docs: https://schedulala.com/developers/docs — link users
there; do not restate the REST reference.

## 1. Pick your interface

Detect what you have available, in this order:

| You have | Use | Extras you get |
|---|---|---|
| Hosted connector tools visible (`show_post_preview`, `attach_media`, …) | The MCP tools below, **preferring the interactive `show_*` widgets** | Inline widgets: faithful post previews, analytics cards, calendar, comments inbox, and the `attach_media` in-chat uploader (images to 20MB, videos to 500MB, PDFs to 100MB) |
| Local stdio MCP tools visible (`create_post` etc., no `show_*` tools) | The same core MCP tools, text results only | Runs via `npx -y @schedulala/mcp-server` with `SCHEDULALA_API_KEY` |
| Neither (terminal only) | The `schedulala` CLI: `npx schedulala <command>` | `--json` on every command; exit codes 0–7; see [references/cli.md](references/cli.md) |

The hosted connector is added in claude.ai / Claude Desktop via
**Settings → Connectors → Add custom connector → `https://schedulala.com/api/mcp`**
(OAuth sign-in, no API key handling; requires a paid Claude plan for custom
connectors). Details: https://schedulala.com/developers/docs/claude-connector

## 2. Auth and safety rails

- **Check auth before anything else**: `list_accounts` (MCP) or
  `schedulala whoami` (CLI). No accounts / auth error → send the user to
  connect: `connect_account` (MCP) or `schedulala init --email <email>` (CLI
  device flow).
- **OAuth, credentials, and payment happen ONLY on schedulala.com in the
  user's browser — never in the chat.** Never ask for passwords, app
  passwords, bot tokens, or card details. `connect_account` and
  `get_upgrade_link` return links; that is the entire flow.
- **Always confirm with the user before anything publishes** (`publishNow`,
  a near-term `scheduledFor`, `reply_to_comment`, `hide_comment`). Prefer
  drafts and previews first.
- API keys: `sk_live_` = real posting; `sk_test_` = sandbox (simulated, no
  real posting, no quota burn). CLI: `--sandbox` requires an `sk_test_` key.

## 3. Core publishing workflow

1. **Discover** — `list_accounts`: platforms, usernames, `accountId`s, health.
   `isConnected: false` + `needsReconnect: true` = dead token → tell the user
   to reconnect in the dashboard; do not post to it. When MORE THAN ONE account
   is connected on a platform, `accountId` is REQUIRED in the platforms array
   (the API returns a 400 listing the choices rather than silently picking).
2. **Draft** — write the content; per-platform text via the object form
   `{ platform, accountId, content }` or `platformSettings.<platform>.content`.
3. **Validate / preview** — `validate_post` checks limits, media counts, and
   required fields per platform WITHOUT posting. On the hosted connector,
   prefer `show_post_preview`: a read-only, faithful per-platform mockup
   (threads, image grids, carousels, PDF docs, 9:16 reels, polls, live char
   counts). Pass it the exact draft you'd give `create_post`.
4. **Confirm** — show the user what will go out, where, and when.
5. **Create** — `create_post` with exactly ONE scheduling mode:
   `scheduledFor` (ISO 8601, 5 minutes to 90 days out) / `publishNow: true` /
   `queue: true` (auto-schedule next open slot) / `status: "draft"`.
6. **Verify** — `get_post` for per-platform status, URLs, and errors;
   `retry_post` re-runs failed platforms (optionally a subset, optional
   `delay` seconds). CLI: `schedulala post:status <id> --watch`.

Editing: `update_post` works on drafts and scheduled posts only (send only
changed fields; `mediaItems` replaces ALL media). `cancel_post` deletes drafts
and cancels scheduled posts; already-processing/posted posts can't be cancelled.

Bulk: `bulk_create_posts` takes up to 25 create_post-shaped entries.
Threads (chains): `create_thread`, 2–25 entries, on twitter / threads /
bluesky / mastodon only, max 1 media item per entry; the whole chain publishes
from ONE postId — `get_post` that id for status.

**First comment (auto-plug)**: pass `firstComment` (plain string) or
`firstComments` (`[{target, text}]`) on `create_post` / `bulk_create_posts` /
`create_thread` to auto-post a comment ~15 seconds after publishing — the
"link in the first comment" workflow. Supported on linkedin, facebook,
instagram, youtube, twitter, threads, bluesky, mastodon (NOT tiktok /
pinterest / telegram / google-business — no comment API). Text and links
only. Comment limits differ from post limits — see
[references/platforms.md](references/platforms.md). A failed comment never
fails the post; `get_post` reports delivery per platform.

**Quota**: threads and bulk consume post quota PER ENTRY (a 5-entry thread = 5
posts). Validation and account-resolution failures don't burn quota. Check
`get_usage` (or `schedulala usage`) when planning volume; on `quota_exceeded`
or the sandbox plan, hand the user the link from `get_upgrade_link`.

## 4. Media decision tree

Full protocol and limits: [references/media.md](references/media.md)

- **User's file (on device or attached in chat)** → `attach_media` (hosted
  connector only): opens an in-chat drop zone; full quality, images ≤20MB,
  videos ≤500MB, PDFs ≤100MB. **NEVER read and base64-encode a user file
  yourself** — chat channels truncate large payloads and destroy quality.
  Without `attach_media` (stdio/CLI), have the user upload on the dashboard
  Media page, then find it with `list_media`.
- **Public http(s) URL** → `upload_media` (mode `url`): re-hosts on the CDN
  (images ≤20MB jpeg/png/webp, gif ≤15MB; videos ≤50MB mp4/mov/webm). Videos
  LARGER than 50MB: skip upload_media and pass the public URL directly in
  `mediaItems` — it is fetched at publish time. Re-hosted media expires after
  30 days if unused.
- **Image YOU generated in this conversation** (chart, card) →
  `begin_generated_media_upload` → `append_generated_media_chunk` (sequential
  `seq` from 0, ≤8000 base64 chars per chunk; 409 → resume from `expectedSeq`;
  429 → wait, resend same seq) → `finish_generated_media_upload`. Images only,
  ≤3MB decoded, session expires in 30 minutes.
- **Already uploaded** ("my latest upload", "that reel from earlier") →
  `list_media`, then reference the returned url.
- CLI: `--media <url[,url…]>` (type inferred from extension).

Multiple `mediaItems` = carousel (instagram, facebook, threads, tiktok,
pinterest; mastodon up to 4 images). PDF (`type: "pdf"`) = LinkedIn document
posts only.

## 5. Platform rules (the gotchas that reject posts)

Full per-platform schemas: [references/platforms.md](references/platforms.md)

| Platform | Must know |
|---|---|
| tiktok | `platformSettings.tiktok.privacy` (e.g. `PUBLIC_TO_EVERYONE`, `SELF_ONLY`) **AND** `platformSettings.tiktok.postMode` (`direct` = publish to profile, `inbox` = user finishes in the TikTok app) are BOTH required. **No default — ASK the user** if they haven't said. |
| instagram | At least one media item required. Reel: `platformSettings.instagram.postType: "reel"`. |
| youtube | Video required; `platformSettings.youtube.title` (≤100 chars) + description (≤5000). |
| pinterest | `platformSettings.pinterest.boardName` (+ 100-char title / 500-char description, optional `link`). |
| linkedin | PDF document post: mediaItem `type: "pdf"` + `platformSettings.linkedin.documentTitle`. |
| google-business | Exactly ONE image, no video; `postType` STANDARD / EVENT / OFFER with CTA buttons via platformSettings; 1500-char limit. |
| mastodon | `platformSettings.mastodon.visibility` (e.g. `public`, `unlisted`). |
| telegram | Polls via `platformSettings.telegram.pollData`. |

Character limits: twitter 280 · bluesky 300 · threads 500 · mastodon 500 ·
google-business 1500 · instagram 2200 · tiktok 2200 · linkedin 3000 ·
telegram 4096 · facebook 63206. (YouTube and Pinterest count title/description
via platformSettings instead.)

## 6. Analytics, engagement, listening

On the hosted connector prefer the widgets: `show_analytics` (one account's
growth + top posts), `show_leaderboard` (ranked posts, sort
engagement|likes|impressions|date), `show_comparison` (side-by-side 2–5
entities — YOU supply the metrics from the analytics tools), `show_calendar`
(upcoming + recent posts), `show_inbox` (comments with Reply/Hide actions),
`show_feeds` (keyword listening feed), `show_connections` (accounts + plan).

Data tools (all interfaces):

- `get_post_analytics` — per-post metrics + engagement rate. Platforms:
  instagram, twitter, bluesky, threads, facebook, linkedin, pinterest, tiktok,
  youtube, mastodon (NOT telegram / google-business). Omit `accountId` to
  merge top posts across all accounts (merged mode doesn't paginate).
- `get_follower_growth` — current count + 7d/30d growth + daily history.
  tiktok reports no follower counts; merged totals exclude YouTube.
- `get_best_posting_times` — top hours/days from engagement history.
- `get_video_transcript` / `get_video_retention` — YouTube only; retention has
  a 7-day cache and ~48h lag. Great for repurposing videos into posts.
- Engagement — comments + replies on facebook, instagram, youtube, linkedin,
  threads, bluesky, google-business; hide on facebook, instagram, youtube,
  threads; mentions on instagram, threads, bluesky. Reply targets differ per
  platform: instagram/youtube take `commentId`; facebook/linkedin take
  `commentId` or `postId`; threads takes either; **bluesky requires all four
  replyRefs** (`parentUri`, `parentCid`, `rootUri`, `rootCid`) from a
  comment's `platformData.replyRefs`. google-business "comments" are customer
  reviews — one owner reply, ≤4096 chars, replying again overwrites.
  `reply_to_comment` and `hide_comment` PUBLISH changes — confirm first.
- Listening — `search_feeds` searches public posts on bluesky + threads
  (threads has a Meta budget of 500 searches per rolling 7 days per account,
  shared with the dashboard — prefer bluesky for exploration);
  `lookup_profile` (bluesky only); `list_feed_keywords` /
  `update_feed_keywords` (max 20 per platform).

## 7. Errors → what to do

| Symptom | Do |
|---|---|
| 400 "Multiple <platform> accounts…" | Re-call with the object form + `accountId` from the listed choices. |
| `quota_exceeded` / sandbox plan ("posting disabled") | `get_upgrade_link` → give the user the link. Reads still work on sandbox. |
| Platform in `failed` state on `get_post` | Read the per-platform error, fix (often a missing platformSettings field), `retry_post`. |
| Account `needsReconnect: true` | User reconnects in the dashboard (`connect_account` link); don't post to it. |
| Validation rejection | `validate_post` the fixed draft before retrying; check the table in §5. |
| CLI exit code 2 | Auth: run `schedulala init` or set `SCHEDULALA_API_KEY`. |

Worked examples (create, tiktok, multi-account, thread, bulk): [examples/](../../examples/)
