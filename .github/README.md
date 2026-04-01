# CI/CD Workflows

## Overview

CI/CD pipelines for mobile app (iOS + Android) using GitHub Actions with **reusable workflows** + **composite actions** architecture.

```
.github/
├── actions/                    # Shared composite actions
│   ├── checkout-private-repo/  # Clone private repo via SSH
│   ├── notify-on-failure/      # Send failure notification via webhook
│   ├── prepare-mobile-build/   # Setup Ruby, Fastlane, gems
│   ├── setup-env/              # Generate .env from Passbolt
│   └── setup-ssh/              # Configure SSH for private repo
│
└── workflows/
    ├── internal-test-build.yml       # Entry: Build iOS AdHoc + APK
    ├── testflight-upload.yml         # Entry: Upload iOS to TestFlight
    ├── reusable-notify-start.yml     # Reusable: Build start notification
    ├── reusable-ios-adhoc.yml        # Reusable: iOS AdHoc build
    ├── reusable-ios-testflight.yml   # Reusable: iOS TestFlight build & upload
    └── reusable-build-apk.yml        # Reusable: Android APK build
```

## Entry Workflows

Manually triggered from GitHub UI (`workflow_dispatch`).

### internal-test-build.yml

Builds iOS AdHoc + Android APK in parallel for internal testing.

| Input | Default | Description |
|-------|---------|-------------|
| `branch` | `develop` | Branch to build |
| `environment` | `staging` | `staging` or `live` |
| `xcode_version` | `16.4` | `16.4` or `26.3` |

```
notify-start ──┐
ios-adhoc ─────┤  (parallel)
build-apk ─────┘
```

### testflight-upload.yml

Builds iOS and uploads to TestFlight.

| Input | Default | Description |
|-------|---------|-------------|
| `branch` | `develop` | Branch to build |
| `environment` | `staging` | `staging` or `live` |
| `xcode_version` | `16.4` | `16.4` or `26.3` |

```
notify-start ──┐
ios-testflight ┘
```

## Reusable Workflows

Called via `workflow_call`, can be reused by any workflow.

| Workflow | Runner | Timeout | Description |
|----------|--------|---------|-------------|
| `reusable-notify-start.yml` | `ubuntu-latest` | - | POST webhook for build start notification |
| `reusable-ios-adhoc.yml` | `macos-15` / `macos-26` | 30m | Build iOS AdHoc, upload to Diawi |
| `reusable-ios-testflight.yml` | `macos-15` / `macos-26` | 30m | Build iOS, upload to TestFlight |
| `reusable-build-apk.yml` | `ubuntu-latest` | - | Build Android APK |

> macOS runner is selected automatically: `macos-26` for Xcode `26.3`, otherwise `macos-15`.

## Composite Actions

### setup-ssh

Configures SSH access to private repo (port 2222).

| Input | Description |
|-------|-------------|
| `ssh_private_key` | SSH private key |
| `private_host` | Private repo host |

### checkout-private-repo

Clones private repo into `real-code` directory.

| Input | Description |
|-------|-------------|
| `branch` | Branch to checkout |
| `private_repo_full_name` | Full SSH URL of the repo |

### setup-env

Generates `.env.stg` or `.env.live` from Passbolt.

| Input | Description |
|-------|-------------|
| `environment` | `staging` or `live` |
| `passbolt_env_file` | Content of `.env.passbolt` file |
| `passbolt_private_key` | Passbolt bot PGP private key |

Flow: write key + env file to disk -> run `npx tsx script/passbolt-env.ts` -> cleanup sensitive files.

### prepare-mobile-build

Sets up Ruby 3.4.4, caches gems, installs Fastlane dependencies.

### notify-on-failure

Sends build failure notification via webhook. Only runs when `if: failure()`.

| Input | Description |
|-------|-------------|
| `project_name` | Project name |
| `webhook_url` | Webhook URL |
| `job_name` | Name of the failed job |
| `branch` | Branch being built |
| `run_url` | Link to GitHub Actions run |

## Secrets & Variables

### Secrets (configured per GitHub Environment)

| Secret | Used by |
|--------|---------|
| `SSH_PRIVATE_KEY` | setup-ssh |
| `PRIVATE_HOST` | setup-ssh |
| `PRIVATE_REPO_FULL_NAME` | checkout-private-repo |
| `PASSBOLT_ENV_FILE` | setup-env |
| `PASSBOLT_PRIVATE_KEY` | setup-env |
| `ANDROID_KEYSTORE_BASE64` | build-apk |
| `APP_STORE_CONNECT_API_KEY` | ios-testflight |
| `FASTLANE_USER` | All build jobs |
| `MATCH_GIT_URL` | iOS builds |
| `MATCH_PASSWORD` | iOS builds |
| `TEAM_ID` | iOS builds |
| `BUNDLE_ID_STAGING` | All build jobs |
| `BUNDLE_ID_LIVE` | All build jobs |
| `DIAWI_API_TOKEN` | iOS AdHoc |
| `DRIVER_KEY_JSON` | Google Drive upload |
| `NOTIFICATION_WEBHOOK_URL` | Notifications |

### Variables

| Variable | Description |
|----------|-------------|
| `PROJECT_NAME` | Project name displayed in notifications |
| `GG_DRIVE_FOLDER_ID` | Google Drive folder for build uploads |
| `KEYCHAIN_NAME` | Keychain name for iOS builds |

## Caching

| Cache | Key | Used by |
|-------|-----|---------|
| Ruby gems | `Gemfile.lock` | All build jobs |
| CocoaPods | `Podfile.lock` | iOS builds |
| Gradle | `gradle-wrapper.properties` + `build.gradle` | Android build |
| Yarn | `yarn.lock` | TestFlight build |

## Adding a New Workflow

1. Create a reusable workflow at `.github/workflows/reusable-*.yml` with `workflow_call` trigger
2. Reuse existing actions (setup-ssh, checkout-private-repo, setup-env, prepare-mobile-build)
3. Create an entry workflow with `workflow_dispatch` that calls the reusable workflows
4. Add `notify-on-failure` action at the end of each job
