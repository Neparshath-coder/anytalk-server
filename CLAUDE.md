# AnyTalk-Server

Builds a patched Stoat backend api image for AnyTalk (the web client lives in `../AnyTalk`). Nothing here is sent upstream. `README.md` explains what the patch does, the image names and how updates work. Read it first.

## How this repo works

- The Stoat source (`stoatchat/stoatchat`, Rust) is **not** in this repo. The workflow clones it at a release tag, applies `patches/*.patch` with `git apply`, and builds the images.
- Only the `api` service (`crates/delta`) is patched and replaced in Coolify. All other Stoat services keep the official images. The patched api must stay on the same version as the other services.
- Images: `ghcr.io/<owner>/anytalk-server-api:<tag>-anytalk`, linux/amd64 only (the server is an old x86 PC). The owner is derived from the repo, so don't hardcode it (the repo may move to another account later).

## Changing the patch

1. Clone upstream at the tag that's currently deployed into a temp folder, apply the existing patches, and make the edit there.
2. Regenerate the patch with `git diff > patches/<file>.patch` from the upstream root. Keep one patch per logical change, numbered (`0002-...patch`).
3. Check that it applies to the deployed tag and the latest release: `git apply --check`.
4. Check that it compiles: `cargo check -p revolt-database`, or `-p revolt-delta` if the change touches `crates/delta`.
5. Update the "What the patch changes" section in `README.md`.

Pushing a change to `patches/**` or the workflow on `main` rebuilds the latest release. Other pushes don't.

## Where things live upstream

- LiveKit token grants: `crates/core/database/src/voice/voice_client.rs` (`create_token`).
- Permission re-sync during calls: `crates/core/database/src/voice/mod.rs` (`sync_user_voice_permissions`).
- "Call started" system messages: `crates/daemons/voice-ingress/src/api.rs`. Patching these would need a second image, so the AnyTalk client marks them as read itself.
- Image build: root `Dockerfile` (base image, compiles everything) plus `crates/delta/Dockerfile`. Its `FROM ghcr.io/stoatchat/base:latest` is hardcoded, so the workflow rewrites it with `sed`.

## Client features that depend on this

- Stream viewers: LiveKit data messages, topic `anytalk:watching` (`components/rtc/components/StreamViewers.tsx` in AnyTalk).
- Deafen indicator: `setAttributes({ deafened })` (`components/rtc/state.tsx` in AnyTalk).

If a patch stops applying, these features break silently. The client gets permission errors and the call itself keeps working.

## Git

- Conventional commits, present tense, lower case, e.g. `fix: patch applies to v0.16.0`.
- A full image build compiles the whole backend and takes a long time, so batch changes to `patches/` before pushing.
