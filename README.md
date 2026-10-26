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
- The hue comes back in `GET /users/{id}/profile`, the edit response and the `UserUpdate` event, like `background`.

Only the `api` service (`crates/delta`) creates tokens and updates participant permissions. `voice-ingress`, `events` and the others don't, so **only the api image needs replacing**.

## Images

- `ghcr.io/<owner>/anytalk-server-api:<upstream tag>-anytalk` (and `:latest`): use this in Coolify instead of `ghcr.io/stoatchat/api`.
- `ghcr.io/<owner>/anytalk-server-base:<upstream tag>-anytalk`: intermediate build image, not deployed.

linux/amd64 only. For arm64 add `linux/arm64` to `platforms:` in both build steps of `.github/workflows/build.yml`. It runs under QEMU and takes hours, so use an arm64 runner (`ubuntu-24.04-arm`) if you need it.

Public visibility: GitHub creates new container packages as private for personal accounts. After the first push, open the package on GitHub (profile, Packages, the package, Package settings) and set visibility to public. Otherwise Coolify needs a registry login. This is a one-time step per package. The base package can stay private if the workflow runs with the same token, but the api package must be public for Coolify to pull it anonymously.

## How updates work

- A daily workflow looks up the latest upstream release and builds it if `<tag>-anytalk` doesn't exist yet.
- Manual build: Actions, "Build patched Stoat server", "Run workflow". Enter an upstream tag (e.g. `v0.15.7`), or leave it empty for the latest release. Tick "force" to rebuild an existing tag.
- Pushing changes to `patches/` or the workflow on `main` rebuilds the latest release.
- When Stoat releases: update all Stoat service tags in Coolify to the new version, and use the `-anytalk` tag for the api service. Keep the patched api on the same version as the other Stoat services.
- A failed build usually means the patch no longer applies to the new upstream code. Fix the patch (check with `git apply --check` in an upstream checkout) and push.
- Watch releases on GitHub (stoatchat/stoatchat, Watch, Custom, Releases) to know when to bump Coolify.
