---
name: twitch-dev-docs
description: Current Twitch developer platform knowledge (Helix API, EventSub, Conduits, Chat/IRC, auth, clips, extensions) distilled from the dev.twitch.tv changelog through 2026-11-10. Use for any Twitch API, EventSub, chatbot, IRC or Twitch integration code.
---

# Twitch Developer Platform — working knowledge (as of 2026-11-10)

Source: https://dev.twitch.tv/docs/change-log/ (full history 2018 → 2026-11-10).
This skill records the *current state* the changelog implies: what to use, what is dead,
and behaviours the docs now state precisely. Always re-check the live reference for
anything newer than 2026-11-10.

## 0. How to use the docs

- **New documentation style launched 2026-11-10.** Pages are served as Markdown:
  `https://dev.twitch.tv/docs/<section>/<page>.md#<anchor>` — e.g.
  `https://dev.twitch.tv/docs/api/reference/chat.md#send-chat-message`,
  `https://dev.twitch.tv/docs/eventsub/reference/chat.md#channel-chat-message`.
  API reference is split per resource: `api/reference/{ads,bits,ccls,channel-points,channels,charity,chat,clips,conduits,entitlements,eventsub,extensions,games,goals,guest-star,hype-train,moderation,polls,predictions,raids,schedule,search,streams,subscriptions,teams,users,videos,whispers}.md`.
  EventSub reference split by area: `eventsub/reference/{automod,channel,channel-moderation,channel-points,charity,chat,conduit,drop,extension,guest-star,hype-train,poll,prediction,stream,user}.md`.
- Changelog RSS: `https://dev.twitch.tv/docs/rss/change-log.xml` (since 2026-03-25). Prefer it for monitoring.
- Before writing integration code, fetch the specific `.md` reference page with WebFetch — fields and status codes were corrected across the whole reference on 2026-11-10.

## 1. What is dead → what to use instead

| Dead / removed | Date | Use instead |
|---|---|---|
| Twitch API v5 (Kraken) | 2022-02-28 | Helix (`https://api.twitch.tv/helix/...`) |
| WebSub webhooks (Get Webhook Subscriptions etc.) | 2021-09-16 | EventSub (webhook / websocket / conduit) |
| PubSub | deprecated 2025-04; migration guide removed 2026-06-12 | EventSub |
| Non-secure IRC WebSocket (`ws://`) | 2025-08-15 | `wss://` only (also TLS for raw IRC) |
| Get Users Follows | 2023-09-06 | Get Followed Channels / Get Channel Followers |
| `channel.follow` v1 | 2023-09-06 | `channel.follow` v2 (needs `moderator:read:followers`) |
| Create / Delete User Follows | 2021-07-27 | none (cannot follow via API) |
| Get Hype Train Events; `channel.hype_train.*` v1 | removed 2026-02-05 / v1 deprecated 2025-06-13 | Get Hype Train Status; `channel.hype_train.*` v2 |
| Get Banned Events, Get Moderator Events | 2022-03-18 | EventSub `channel.ban`, `channel.moderator.add/remove`, `channel.moderate` |
| Soundtrack endpoints | 2023-07-17 | none |
| Get Code Status / Redeem Code (Bits codes) | 2023-02-27 | none |
| Replace Stream Tags; `tag_ids` (always `[]`); Get All Stream Tags / Get Stream Tags → 410 | 2023 | `tags` field (free-form) in Get Streams, Search Channels, Get/Modify Channel Information |
| `is_mature` in `channel.update` | v2 2023-07-17 | `content_classification_labels` (CCLs); Get Content Classification Labels |
| Bits voting in polls (`bits_voting_enabled`, `bits_per_vote`, `bits_votes`) | 2022-09-01; fields gone from docs 2026-11-10 | — |
| `view_count` in Get Users | stale since 2022-06-27 | do not display |
| Developer Rig | unsupported 2023-03 | Twitch CLI |
| Chat rooms; IRC `MODE`/`NAMES` | 2020 | Get Chatters API |
| `/r9kbeta` | 2022 | `/uniquechat` |
| Combos in `channel.bits.use` | removed 2026-04-02 | — |
| `MODERATED` poll status | removed from docs 2026-11-10 | — |

## 2. Authentication

- OAuth is mandatory for every Helix call: headers `Client-Id: <id>` and `Authorization: Bearer <token>`; send `Content-Type: application/json` for bodies.
- Token types: app access token (client credentials) and user access token. **App access token support for several endpoints is GA (2026-05-29)** — check each endpoint's Authorization section.
- Device Code Flow (DCF) is GA (2023-12) — good for CLI tools/headless bots/TVs.
- Get Authorization By User (GA 2025-10-30): which scopes users granted your app.
- Revoking an invalid token returns `400`, not `200`.
- App names must not contain "Twitch" (exact or fuzzy) or registration fails.
- Notable scopes: `channel:bot`, `user:bot`, `user:read:chat`, `user:write:chat`, `moderator:read:followers`, `moderator:manage:warnings`, `moderator:read:warnings`, `moderator:manage:announcements`, `moderator:manage:chat_messages`, `moderator:manage:automod`, `user:read:moderated_channels`, `channel:read/manage:ads`, `channel:read/manage:guest_star`, `moderator:read/manage:guest_star`, `channel:read/manage:redemptions`, `channel:manage:broadcast`, `channel:read:subscriptions`, `user:read:email` (optional for Get Users), `user:read:follows`, `user:manage:whispers`, `user:manage:chat_color`, `channel:read/manage:vips`, `channel:manage:moderators`, `channel:read:editors`, `channel:manage:videos`, `user:read/manage:blocked_users`.
- Scopes page is split by API/EventSub, IRC, PubSub.

## 3. EventSub

### Transports
- `webhook` (needs `callback` + `secret`), `websocket` (needs `session_id`), `conduit` (needs `conduit_id`).
- WebSocket URL: `wss://eventsub.wss.twitch.tv/ws`. Flow: welcome → keepalive/ping → notifications; handle `reconnect` messages. Client may only send pong (in reply to ping); any other inbound traffic disables subs.
- Max **300 enabled subscriptions per WebSocket connection**.
- Transport response fields: `connected_at`, `disconnected_at` (websocket).
- Subscription limits are cost-based (no `limit` field).

### Conduits (GA 2024-01-25)
- Create/Get/Update/Delete Conduits; Get/Update Conduit Shards; event `conduit.shard.disabled`.
- `shard_count`: 1–20,000.
- Update Conduit Shards: max **100 shards per request**; transport validation errors are returned as a **400** response, not per-shard `errors`.
- Get Conduit Shards documents every `status` value.

### Create / Get EventSub Subscriptions
- Create returns `is_batching_enabled`. On **409 Conflict**, the existing subscription's ID is in `message` (not `id`).
- Get filters: `status`, `type`, `user_id`, `subscription_id` (returns matching sub or empty array), `conduit_id`.
- Get response: `conduit` transport, `conduit_id`, `is_batching_enabled`, `condition` is a JSON object.
- Disabled WebSocket subs are still returned for ≥1 minute; disabled webhook subs for ≥10 days.
- Disabled statuses include: `websocket_disconnected`, `websocket_failed_ping_pong`, `websocket_received_inbound_traffic`, `websocket_connection_unused`, `websocket_internal_error`, `websocket_network_timeout`, `websocket_network_error`, `moderator_removed`, `version_removed` (sent in revocation).

### Recent/important subscription types
- `channel.chat.message` — fragments include `gif` type (`gif.gif_id`, `gif.url`, plus `text`) since 2026-07-16; `is_source_only` in shared chat; `message_type` values `power_ups_message_effect`, `power_ups_gigantified_emote` (renamed 2024-06); `channel_points_animation_id`.
- `channel.chat.notification` — notice types include Watch Streaks, `modiversary`, `shared_chat_modiversary`, `gifted_drops_summary`, `shared_chat_gifted_drops_summary`, `unknown` (handle unknown gracefully!); `is_source_only` for announcements in shared chat; `sub_tier` (not `sub_plan`), `is_prime`, `badges`; no `sub.cumulative_months`.
- `channel.chat.clear`, `.clear_user_messages`, `.message_delete`, `.chat_settings.update`, `channel.chat.user_message_hold/update`.
- `channel.shared_chat.begin/update/end` + Get Shared Chat Session (GA 2024-09-16).
- `automod.message.hold` v2 / `automod.message.update` v2 — `reason` may be `blocked_link` (then `blocked_term` is populated); only public blocked terms trigger. Also `automod.settings.update`, `automod.terms.update` (private term changes not sent).
- `channel.moderate` v2 (includes warnings), `channel.warning.acknowledge/send`, `channel.suspicious_user.message/update` (`types`: `ban_evader`, `manually_added`, `banned_in_shared_channel`), `channel.unban_request.create/resolve`, `channel.vip.add/remove`, `channel.ban` (`banned_at`, reason, timeout duration), `channel.shield_mode.begin/end`.
- `channel.bits.use` v1 — includes Custom Power-ups; `emote_set_id`, `owner_id`, `format`; combos removed.
- `channel.custom_power_up_redemption.add` v1 (GA 2026-05-19, identical to beta) + Get Custom Power-Up API.
- `channel.channel_points_automatic_reward_redemption.add` v2 — `reward_type` values `message_effect`, `gigantify_an_emote`, `celebration`, …
- `channel.hype_train.begin/progress/end` v2 (v1 deprecated); Golden Kappa Train flag.
- `stream.online` (`started_at`, `id`), `stream.offline` v1 now includes `id` (2026-06-26).
- `channel.update` v2 (CCLs instead of `is_mature`).
- `channel.ad_break.begin` (`duration_seconds`).
- `channel.follow` v2, `channel.shoutout.create/receive`, `channel.charity_campaign.*`, `channel.goal.*` (types incl. `new_bit`, `new_cheerer`), polls/predictions, `channel.guest_star_*`, `user.whisper.message`, `user.update` (`email_verified`), `user.authorization.grant/revoke`, `drop.entitlement.grant`, `extension.bits_transaction.create`.

## 4. Chat & chatbots

- Preferred: **EventSub for reading** (`channel.chat.message`) + **Send Chat Message API** for writing. IRC still documented in "IRC Concepts" (`dev.twitch.tv/docs/chat/irc.md`).
- **Chat Bot badge / "Chat Bot" in user list:** send with an *app access token*, bot authorised with `user:bot`, and broadcaster granted `channel:bot` (or bot is a mod). See "Chatbot Badge and Chat Identity" in `chat.md`.
- **Send Chat Message:** `for_source_only` defaults to **true with an app access token** (shared chat); optional `pin` param sends and pins in one call; response may contain `drop_reason` (all codes documented).
- Pinned chat API (Get / Pin / Update / Unpin Chat Message) — GA 2026-05-29.
- Send Chat Announcement: supports app access tokens and `for_source_only`; can return 429.
- Update Chat Settings: `follower_mode` defaults to false; `follower_mode_duration` 0–129600; `slow_mode_wait_time` 3–120.
- Get Chatters (GA): returns `user_id`, `user_name`, `user_login`.
- Get User Emotes: follower emotes appear only if you pass `broadcaster_id` (subscribing doesn't add them). Get Emote Sets lists every `emote_type`. Emotes have `format`, `scale`, `theme_mode`, `template` (build URLs from the template).
- Warn Chat User (GA), Add/Remove Suspicious Status to Chat User (GA 2026-02-05).
- **IRC tags:** parse individually, accept any order, ignore unknown tags. PRIVMSG: `gif` (2026-07-17), `source-only`, `source-room-id`, `source-id`, `source-badges`, `source-badge-info`, `vip`, `reply-parent-*` (direct parent) + `reply-thread-parent-msg-id/user-login`, `badge-info`. USERNOTICE: `msg-id` values incl. `sharedchatnotice`, `viewermilestone` (watch streaks), `modiversary`; `source-msg-id`. Deprecated tags `subscriber`, `turbo`, `user_type` → use `badges`. `anonsubgift`/`anonsubmysterygift` removed.

## 5. Endpoint behaviours clarified 2026-11-10 (trust these over old code)

**Ads:** Get Ad Schedule and Snooze Next Ad timestamps are **Unix seconds**. Fields: `duration`, `preroll_free_time` (renamed from `*_seconds` at GA).
**CCLs:** Get Content Classification Labels — correct `locale` values listed.
**Channel Points:** Create/Update Custom Reward — no 200-char limit on `prompt`; `cost` max 1,000,000,000. Get Custom Reward Redemption — `status` not required (ignored when `id` given). `max_per_stream` required if `is_max_per_stream_enabled` set.
**Channels:** Modify Channel Information — unsupported `broadcaster_language` → **400** (not ignored); tags not case-sensitive. Get Channel Information — up to 100 `broadcaster_id`; `delay` needs user token (else 0); returns `content_classification_labels`, `is_branded_content`, `tags`.
**Clips:** Get Clips — results for `id` are **not** guaranteed in input order; no `before` param; max ~1000 results across pagination; `is_featured` filter/field; `vod_offset`, `duration`. Create Clip — optional `title`, `duration`, 2K and vertical (dual-format) output, new edit URL. Create Clip From VOD and Get Clips Download are GA.
**Conduits:** see §3.
**Entitlements:** Get Drops Entitlements — unless you pass `id` or both user and game, only last **6 months** returned; `first` up to 1000; `fulfillment_status`.
**Extensions:** secret `content` is base64. Get Extensions / Released Extensions `state` corrected, `views.live_config` added. Configuration Segment `segment` may repeat. Get Extension Live Channels → empty list (not 404) when none live. Send Extension PubSub Message — `broadcaster_id` required only when `is_global_broadcast` is false.
**Games:** Get Games — `id`/`name`/`igdb_id` each optional but at least one required; supports `igdb_id`. Get Top Games — no `before`.
**Guest Star:** Update Channel Guest Star Settings is **PATCH**; Get settings returns them under `data`; `group_layout` values corrected; session responses list guests under `guests`, use `audio`/`video` (not `audio_settings`/`video_settings`), no `user_login`/`user_display_name`; Update Slot requires `destination_slot_id`; Update Slot Settings has no `volume`; Delete Slot `should_reinvite_guest` is Boolean.
**Moderation:** Get Blocked Terms — `cursor` under `pagination`. Get Banned Users — no `before`. Get Moderators — `first` is Integer. Get Unban Requests — `broadcaster_name` top-level. Ban/Unban can return 409 and 429.
**Polls:** no Bits fields; `ended_at` omitted while active; no `MODERATED` status.
**Predictions:** `locked_at` null only if not locked; `prediction_window` min 1 second; up to 10 outcomes.
**Schedule:** segment `duration` is Integer; Get Channel Stream Schedule has top-level `pagination`.
**Search Channels:** no 6-month restriction; display names matched too.
**Streams:** Get Followed Streams — only streams from the **first 2,000 broadcasters** the user followed. Get Stream Markers — ordered by position in video; field is `URL`; no `before`. Get Streams `game_id` up to 10 IDs.
**Subscriptions:** Get Broadcaster Subscriptions — `points` and `total` **omitted** (not null) when `user_id` given.
**Users:** Get User Active Extensions — every slot always returned, inactive slot has only `active`. Get User Block List — new `user_id` param to check specific users. Get Users — `email` **omitted** (not empty) for other users; max 100 IDs+logins combined. Update User Extensions `data` is Object; Update User can 429.
**Videos:** Delete Videos — one invalid ID fails the whole request. Get Videos — types include `past_premiere`, `premiere_upload`; `duration` format corrected; no 404 for unknown IDs; `muted_segments`, `stream_id`; 500-result limit with `game_id`.
**General:** `first` is Integer in Get Polls/Predictions/Stream Markers/Broadcaster Subscriptions/Videos; status codes, error reasons and auth details corrected across the reference — re-read before relying on old assumptions. `first` is a maximum; fewer items may come back. Paginate with `cursor`.

## 6. Other areas

- **Hype Train:** Get Hype Train Status (GA 2025-08-19).
- **Goals:** Get Creator Goals types include `subscription`, `new_subscription`, `subscription_count`, `new_subscription_count`, `follower`, `new_bit`, `new_cheerer`.
- **Teams:** `background_image_url`, `banner` may be null.
- **Raids:** Start/Cancel a Raid (GA). **Whispers:** Send Whisper. **Shoutouts:** Send a Shoutout (GA).
- **Drops:** subscription-based Drops available to all game devs; overlapping campaign rules (watch minutes & subs split across simultaneous campaigns); Campaign Guide + Technical Guide; offline inventory C# sample; translations ja/zh/ko/ar.
- **Extensions:** CSP enforced (`connect-src` allows `blob:`; `img-src` allows Twitch image CDN; updated GA URLs as of 2026-07-31); Extension Helper must be the first script in every HTML file; mobile extensions work on iOS/Android with no extra allowlist; NFT and global-overlay policies (4.10, 4.11); `onPositionChanged` removed from SDK reference; monetization strategies guide for Bits-in-Extensions.
- **Embed:** `parent` parameter required (domain list); standardised minimum player size; autoplay requirements (default `autoplay=false`; features disabled if iframe obscured); captions API, `getPlaybackStats()`, `SEEK`/`PLAYING`/`PLAYBACK_BLOCKED` events.
- **Game Engine Plugins:** Unity, Unreal, C++, GameMaker (GA 2024).
- **Twitch CLI:** mock API, `event trigger`, WebSocket mock server (`start-websocket-server`, test reconnect, forward mock events).
- **Breaking changes policy:** see `api/guide.md#breaking-changes`; Product Lifecycle page tracks deprecations.

## 7. Checklist when writing Twitch code

1. Helix + EventSub only. No Kraken, PubSub or WebSub.
2. Pick the EventSub transport: WebSocket for a single small client (≤300 subs/connection), conduit for scale, webhook for a server with a public HTTPS endpoint.
3. Handle `reconnect`, keepalive timeouts, revocations (incl. `version_removed`, `moderator_removed`) and unknown enum values (`unknown` notice type, new fragment types like `gif`).
4. Use the latest version of each subscription type (`channel.follow` v2, `channel.update` v2, hype train v2, automod v2, `channel.moderate` v2).
5. Treat fields documented as "omitted" as possibly absent (not null): `email`, `points`/`total`, poll `ended_at`.
6. For bots: app token + `user:bot` + `channel:bot` → bot badge; mind `for_source_only` in shared chat.
7. Fetch the `.md` reference page for the exact endpoint before finalising request/response types.
