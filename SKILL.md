---
name: asc-release
description: Use the asc CLI to prepare and submit a new App Store Connect app version, including release requests, opening a new version, copying promotional text from the currently released version, applying changelogs, attaching a build number, or submitting an app version for App Review across one or more platforms and localizations
---

# asc release

Use this skill to prepare a new app version with `asc`, pause for the build number, then attach the build and submit for App Review

## Core rules

- Use `asc --help` and subcommand `--help` before relying on command syntax
- Include all platforms unless the user explicitly excludes platforms
- Include all localizations unless the user explicitly narrows localizations
- Do not localize changelogs unless the user explicitly asks for localization
- Use the same provided changelog text across every included localization by default
- Stop after metadata updates and wait for the user to provide a build number in a later prompt
- Do not select a build or submit for review before the user provides the build number
- Prefer explicit app, version, platform, locale, version ID, and build selectors
- Preview remote writes with `--dry-run` when supported
- Use `--output json` for parsing and `--output table` for user-facing checks

## Inputs to resolve

Resolve these before changing App Store Connect:

- App ID or app name
- New version string
- Platforms to include
- Localizations to include
- Currently released version for each platform
- Changelog text provided by the user
- Whether changelogs should be localized

If the user has not supplied a changelog, ask for it before creating or updating version metadata

## Phase 1: Prepare metadata

1. Resolve the app and included platforms
   - Use all platforms by default
   - Exclude only platforms the user names
   - Resolve platform-specific current released versions and target new versions

2. Open the new version for every included platform
   - Discover the current `asc` version-create command with `asc versions --help` and related subcommand help
   - Create the new app-store version only when it does not already exist
   - Store every resolved target `VERSION_ID`

3. Resolve localizations for every included platform
   - Use all existing localizations by default
   - Narrow localizations only when the user asks
   - Create missing target-version localizations when needed

4. Copy promotional text from the currently released version
   - For each included platform and locale, read the released version localization
   - Copy only the promotional text from the released version to the target new version
   - Do not copy unrelated version metadata unless needed by `asc` to preserve existing values

5. Add changelog text to the new version
   - Set `whatsNew` or the current `asc` equivalent on every included target-version localization
   - Use the exact changelog text provided by the user across all included localizations
   - Localize changelogs only when the user explicitly requests localization

6. Validate and apply
   - Prefer the canonical metadata workflow when practical:

```bash
asc metadata pull --app "APP_ID" --version "NEW_VERSION" --platform PLATFORM --dir "./metadata"
asc metadata validate --dir "./metadata" --output table
asc metadata push --app "APP_ID" --version "NEW_VERSION" --platform PLATFORM --dir "./metadata" --dry-run --output table
asc metadata push --app "APP_ID" --version "NEW_VERSION" --platform PLATFORM --dir "./metadata"
```

   - For small targeted edits, use the current quick-edit command discovered with `asc apps info edit --help`, `asc versions --help`, or related help
   - Always validate after editing and before submission

7. Stop
   - Report the prepared platforms, version IDs, localizations, and validation state
   - Ask the user for the build number
   - End the turn without attaching a build or submitting for review

## Phase 2: Attach build and submit

Use this phase only when the user has provided the build number after Phase 1

1. Resolve the build for each relevant platform
   - Match the user-provided build number to the included platforms
   - Confirm each build is processed and eligible for App Store release
   - If more than one build matches a platform, disambiguate before submitting

2. Select the build for release
   - Attach the resolved build to each target version
   - Prefer `asc review submit` when it covers the flow:

```bash
asc review submit --app "APP_ID" --version "NEW_VERSION" --build "BUILD_NUMBER_OR_ID" --dry-run --output table
asc review submit --app "APP_ID" --version "NEW_VERSION" --build "BUILD_NUMBER_OR_ID" --confirm
```

   - Use `--version-id "VERSION_ID"` instead of `--version` when that is more deterministic

3. Submit for App Review
   - Submit every included platform version after the dry run is clean
   - Capture submission IDs and review status
   - Report submitted platforms, selected builds, and submission IDs

## Fallbacks

- If the public API does not support a required release step, identify whether an `asc web ...` command exists and say that it is an experimental web-session fallback before using it
- If no CLI path exists for a required App Store Connect action, stop and give the exact manual step needed
- If validation reports blockers, fix API-addressable metadata issues, then re-run validation before continuing
