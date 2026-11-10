# AnyTalk-Server

Builds a patched Stoat backend (`stoatchat/stoatchat`) for AnyTalk. Nothing here is sent upstream.

## What the patch changes

`patches/0001-voice-allow-data-messages-and-own-attributes.patch` (two files in `crates/core/database/src/voice/`):

- `voice_client.rs`: the LiveKit join token now has `can_publish_data: true` and `can_update_own_metadata: true`.
- `mod.rs`: when permissions are re-synced during a call, the participant permission keeps `can_publish_data: true` and sets `can_update_metadata: true`. Without this the re-sync would revoke attribute updates mid-call.

Why: the AnyTalk client sends LiveKit data messages (topic `anytalk:watching`, stream viewers) and calls `setAttributes({ deafened })`. Stock Stoat rejects both. Both are set to true for everyone, including listen-only users, so the deafen state still works for them. Upstream tied data publishing to the speak permission, which we no longer do.

`patches/0002-voice-resync-publish-sources-and-target-user.patch` (security fixes, applies on top of 0001):

- `voice/mod.rs`: the permission re-sync during a call now sends `can_publish_sources` (the same list the join token uses, via `get_allowed_sources`) and `can_publish` only when that list is non-empty. Before, an empty list meant "all sources" to LiveKit, so a member without the Video permission could publish camera and screen share after any role edit mid-call.
- `crates/delta/src/routes/servers/member_edit.rs`: the re-sync after a server mute/deafen change now uses the edited member instead of the moderator, so the change reaches LiveKit. Removing a member from the voice channel (`FieldsMember::VoiceChannel`) now also removes the edited member, not the moderator.

`patches/0003-profile-banner-hue.patch` (profile banner colour, applies on top of 0002):

- Adds an optional `banner_hue` (0-359, `u16`) to the user profile (database `UserProfile` and v0 `UserProfile`, skipped when unset). Old documents without it still deserialize because it is optional.
- `PATCH /users/@me` accepts `profile.banner_hue`, validated to 0-359 (otherwise `FailedValidation`). It is merged into the existing profile, so setting `content`, `background` or `banner_hue` alone keeps the others.
- New `FieldsUser::ProfileBannerHue` (`"ProfileBannerHue"` in `remove`) clears it (Mongo path `profile.banner_hue`).
- The hue comes back in `GET /users/{id}/profile`, like `background`. The `UserUpdate` event carries no profile data (upstream's event user has no profile field), so clients reload the profile over the API.
- `ProfileBannerHue` is left out of the `clear` list of the `UserUpdate` event (`User::update` in `models/users/model.rs`). The official events service doesn't know the value and would drop the whole event, including other fields changed in the same edit and account deletions, which clear every profile field.

`patches/0004-qr-login.patch` (log in a new device by scanning a QR code, applies on top of 0003):

- New file `crates/delta/src/routes/session/qr.rs`, routes registered in `routes/session/mod.rs`. Pending logins live only in Valkey (hash `anytalk:qr:<id>`), expire after 120 s, and are deleted once collected. No Mongo model. At most 1000 pending logins at once (sorted set `anytalk:qr-index`).
- `POST /auth/session/qr` (no auth) starts a login and returns `id` (goes into the QR), `secret` (only for the new device, stored as SHA-256) and `expires_at`. The friendly name is cleaned (control and bidi characters removed, 64 characters at most). The coarse IP (last part hidden) and User-Agent are stored for the approver.
- `POST /auth/session/qr/{id}/poll` (no auth, body `{secret}`, constant-time compare) returns `pending`, `scanned`, `denied`, `expired` (also for a wrong secret) or `approved` with the new session, exactly once. Responses are `Cache-Control: no-store`.
- `POST /auth/session/qr/{id}/info`, `POST .../approve` and `POST .../deny` need a session. `info` marks the request scanned and says which method `approve` needs (`password` or `totp_or_recovery`). `approve` requires the account password (no MFA) or a TOTP/recovery code (MFA), refuses disabled accounts, applies the account lockout, claims the request atomically (Lua scripts) so concurrent approves create one session only, and denies the request after 3 wrong answers. A session that cannot be handed over (request expired meanwhile) is deleted again.
- `util/ratelimits.rs`: own buckets `auth_qr_start` (5 per 10 s per client IP) and `auth_qr_poll` (20 per 10 s, counted per client IP and per QR id, so polls keep their own budget even when a proxy hides the real IP). The api takes the client IP from `X-Forwarded-For` (`ip_header` in `main.rs`) or `CF-Connecting-IP` (`trust_cloudflare`), so the reverse proxy must set one of them, otherwise all clients share the proxy's IP in the start bucket and in the stored coarse IP. `crates/delta/Cargo.toml` and `Cargo.lock` add `sha2`.

`patches/0005-ratelimit-user-edit.patch` (applies on top of 0004):

- `util/ratelimits.rs`: the `user_edit` bucket (every `PATCH /users/...`: presence, status, profile, avatar, banner, username) is raised from 2 to 20 requests per 10 s, counted per session and user. The 10 s window is fixed and shared by all buckets, so this allows up to 20 in a burst, about 2 per second on average. With the stock limit, switching presence more than twice in 10 s returned 429.

Only the `api` service (`crates/delta`) creates tokens, updates participant permissions and serves the new routes. `voice-ingress`, `events` and the others don't, so **only the api image needs replacing**. The other services run official images and decode the shared models (`crates/core/database`, `crates/core/models`) with upstream's definitions, so a patch must not put new enum variants into anything the api publishes as an event. The events service can't decode them and drops the whole event. New fields on event types are ignored with the default JSON payload format, but break the `msgpack`/`bincode` formats (`REDIS_PAYLOAD_TYPE`). New optional fields that are only stored and served by the api are fine.

## Images

- `ghcr.io/<owner>/anytalk-server-api:<upstream tag>-anytalk` (and `:latest`): use this in Coolify instead of `ghcr.io/stoatchat/api`.
- `ghcr.io/<owner>/anytalk-server-base:<upstream tag>-anytalk`: intermediate build image, not deployed.

linux/amd64 only. For arm64 add `linux/arm64` to `platforms:` in both build steps of `.github/workflows/build.yml`. It runs under QEMU and takes hours, so use an arm64 runner (`ubuntu-24.04-arm`) if you need it.

Public visibility: GitHub creates new container packages as private for personal accounts. After the first push, open the package on GitHub (profile, Packages, the package, Package settings) and set visibility to public. Otherwise Coolify needs a registry login. This is a one-time step per package. The base package can stay private if the workflow runs with the same token, but the api package must be public for Coolify to pull it anonymously.

## How updates work

- There is no scheduled run: a new Stoat release is not built on its own. Build it by hand (below) when you want to update.
- Manual build: Actions, "Build patched Stoat server", "Run workflow". Enter an upstream tag (e.g. `v0.15.7`), or leave it empty for the latest release. Tick "force" to rebuild an existing tag.
- Pushing changes to `patches/` or the workflow on `main` rebuilds the latest release.
- When Stoat releases: update all Stoat service tags in Coolify to the new version, and use the `-anytalk` tag for the api service. Keep the patched api on the same version as the other Stoat services.
- A failed build usually means the patch no longer applies to the new upstream code. Fix the patch (check with `git apply --check` in an upstream checkout) and push.
- Watch releases on GitHub (stoatchat/stoatchat, Watch, Custom, Releases) to know when to bump Coolify.
