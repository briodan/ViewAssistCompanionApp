# Fork maintenance notes

This repository is briodan's fork of [msp1974/ViewAssistCompanionApp](https://github.com/msp1974/ViewAssistCompanionApp).
This file is **fork-only**: it lives on the `fork/version` branch and must never appear in a PR to upstream.

## Remotes

| Remote     | Repo                                 | Use                    |
|------------|--------------------------------------|------------------------|
| `origin`   | `briodan/ViewAssistCompanionApp`     | push target (the fork) |
| `upstream` | `msp1974/ViewAssistCompanionApp`     | fetch only             |

```
git remote add upstream https://github.com/msp1974/ViewAssistCompanionApp.git   # if missing
```

## Branch layout

```
upstream/dev ──► feature/* (one per feature) ──► fork/version ══ dev
```

- **`feature/<name>`**: based on `upstream/dev`, containing only that feature's commits.
  These are the branches PRs go to upstream from. They must never contain a version bump or fork-only files.
  - Current: `feature/rtsp-camera-streaming`.
- **`fork/version`**: `upstream/dev` plus the fork-only commits (the version bump and this file).
  It is always applied **last**.
- **`dev`**: the fork's build branch. It is **assembled** from the branches above, not edited by hand.
  Because it is rebuilt on every sync, pushing it needs `--force-with-lease`.

## Version rule

The fork's `versionName` (`app/build.gradle.kts`) is **one patch above upstream's latest released version**.

Upstream publishes APK releases from the integration repo
[msp1974/ViewAssist_Companion_App](https://github.com/msp1974/ViewAssist_Companion_App/releases),
and the app repo's `versionName` often lags behind. Use the **higher** of:

```
gh release list --repo msp1974/ViewAssist_Companion_App --limit 1                # latest release tag
git show upstream/dev:app/build.gradle.kts | grep 'versionName = '               # app repo version
```

Example: upstream released 0.13.5, so the fork is 0.13.6.

`versionCode` comes from the local, auto-incremented `version.properties`. If it is below the
code already installed on a device, `adb install -r` fails with `INSTALL_FAILED_VERSION_DOWNGRADE`.
Raise `VERSION_CODE` above the installed code and rebuild:

```
adb shell dumpsys package com.msp1974.vacompanion | grep versionCode
```

## Syncing from upstream

```bash
git fetch upstream && git fetch origin

# 0. Safety net
git tag backup/dev-pre-sync-$(date +%F) dev

# 1. Rebase each feature branch onto upstream
git rebase upstream/dev feature/rtsp-camera-streaming      # repeat per feature/*
#    Conflicts: keep upstream's new APIs and re-apply the feature's intent on top.
#    Always take upstream's version.properties.

# 2. Rebase fork-only commits
git rebase upstream/dev fork/version
#    Expected conflict: the versionName line whenever upstream bumped it.
#    Resolve by setting upstream-release + 1 (see Version rule).
#    If the number didn't change but upstream released, amend the bump commit.

# 3. Reassemble dev: upstream, then features, then fork/version last
git checkout dev
git reset --hard upstream/dev
git merge --ff-only feature/rtsp-camera-streaming   # first feature fast-forwards
# git cherry-pick upstream/dev..feature/<other>     # any further features
git cherry-pick upstream/dev..fork/version

# 4. Verify
./gradlew assembleDebug lint
git diff upstream/dev...feature/rtsp-camera-streaming -- app/build.gradle.kts FORK.md   # must show no fork-only changes

# 5. Publish
git push --force-with-lease origin dev fork/version feature/rtsp-camera-streaming
```

## Opening a PR to upstream

1. Branch from `upstream/dev` (never from `dev`): `git checkout -b feature/<name> upstream/dev`.
2. Commit only the feature's changes. Leave `versionName`, `version.properties` and `FORK.md` untouched.
3. Before opening the PR, confirm `git diff upstream/dev...feature/<name> --stat` lists only feature files.
4. Add the branch to step 1 of the sync procedure above.

## Building and installing

- `app/google-services.json` is gitignored. Copy it into any fresh clone or worktree, or the build fails.
- Test tablet: `adb connect 10.10.10.37` (adb is at `/opt/android-sdk/platform-tools/adb`), then
  `adb install -r app/build/outputs/apk/debug/vaca-<version>-debug.apk`.
- The build host's disk is small (12G). Prefer building in the main checkout over fresh worktrees.
