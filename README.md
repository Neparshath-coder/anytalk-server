# AnyTalk-Server

Builds a patched Stoat backend (`stoatchat/stoatchat`) for AnyTalk. Nothing here is sent upstream.

## What the patch changes

`patches/0001-voice-allow-data-messages-and-own-attributes.patch` (two files in `crates/core/database/src/voice/`):

- `voice_client.rs`: the LiveKit join token now has `can_publish_data: true` and `can_update_own_metadata: true`.
- `mod.rs`: when permissions are re-synced during a call, the participant permission keeps `can_publish_data: true` and sets `can_update_metadata: true`. Without this the re-sync would revoke attribute updates mid-call.

Why: the AnyTalk client sends LiveKit data messages (topic `anytalk:watching`, stream viewers) and calls `setAttributes({ deafened })`. Stock Stoat rejects both. Both are set to true for everyone, including listen-only users, so the deafen state still works for them. Upstream tied data publishing to the speak permission, which we no longer do.

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
