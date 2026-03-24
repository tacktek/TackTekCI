# TackTekCI

Public CI/CD repository for the Tack Tek iOS application. This repo exists as a public repository to take advantage of GitHub's free macOS runner minutes, avoiding the cost of macOS runners on private repositories.

## How It Works

The private [TackTek](https://github.com/tacktek/TackTek) repository triggers builds in this repo via `repository_dispatch` webhook events. When triggered, these workflows clone the private repo, build the iOS app, and deploy to TestFlight or the App Store.

```
TackTek (private) ── repository_dispatch ──> TackTekCI (public) ── macOS runner ──> TestFlight / App Store
```

## Workflows

### ci.yml — PR Staging

Runs CI checks on pull requests targeting the staging branch. Triggered by `pr-staging` dispatch event.

| Dispatch Payload | Description |
|-----------------|-------------|
| `source_repo` | Repository that triggered the build |
| `ref` | Git ref being built |
| `sha` | Commit SHA to checkout |

**Flow:** Checkout at specific SHA -> Ruby 3.2 -> `bundle install` -> `fastlane ci`

### deploy-to-testflight.yml — TestFlight Build

Builds from the `release` branch and uploads to TestFlight. Triggered by `testflight` dispatch event.

**Flow:** Clone `release` branch -> Xcode setup -> SSH key for Match -> Keychain setup -> Decode API key -> Ruby 3.2 + Fastlane -> `fastlane test_flight`

### deploy-to-appstore.yml — App Store Build

Builds from the `production` branch and uploads to the App Store. Triggered by `appstore` dispatch event.

**Flow:** Clone `production` branch -> Xcode setup -> SSH key for Match -> Keychain setup -> Decode API key -> Ruby 3.2 + Fastlane -> `fastlane app_store`

## Required Secrets

| Secret | Description |
|--------|-------------|
| `PRIVATE_REPO_TOKEN` | GitHub PAT with access to the private TackTek repo |
| `APP_STORE_CONNECT_API_KEY_ID` | App Store Connect API key ID |
| `APP_STORE_CONNECT_API_ISSUER_ID` | App Store Connect API issuer ID |
| `APP_STORE_CONNECT_API_KEY_BASE64` | Base64-encoded App Store Connect API key (.p8) |
| `MATCH_PASSWORD` | Password for decrypting Match certificates |
| `MATCH_SSH_PRIVATE_KEY` | SSH key for accessing the Match certificates repo |

## Triggering Builds

Builds are triggered from the TackTek repo using GitHub's `repository_dispatch` API:

```bash
# Trigger CI
gh api repos/tacktek/TackTekCI/dispatches \
  -f event_type=pr-staging \
  -f client_payload[source_repo]=tacktek/TackTek \
  -f client_payload[ref]=refs/heads/staging \
  -f client_payload[sha]=abc1234

# Trigger TestFlight build
gh api repos/tacktek/TackTekCI/dispatches \
  -f event_type=testflight

# Trigger App Store build
gh api repos/tacktek/TackTekCI/dispatches \
  -f event_type=appstore
```

## Security Note

This is a public repository. No source code, secrets, or sensitive configuration is stored here. The workflows clone the private repo at runtime using `PRIVATE_REPO_TOKEN`, and all signing credentials are injected via GitHub Secrets.
