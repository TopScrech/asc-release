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
- Some quick-edit commands do not support `--dry-run`; validate after targeted writes when no dry run is available
- Use `--output json` for parsing and `--output table` for user-facing checks
- Use structured parsers such as `jq` for JSON output; do not parse table output
- Keep user-facing summaries short: app, version, platform, IDs, state, validation, blockers, and next action

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
   - If App Store Connect has no version history for a platform, report that and continue only with resolved App Store Connect platforms
   - Common discovery commands:

```bash
asc apps list --bundle-id "BUNDLE_ID" --output json --pretty
asc versions list --app "APP_ID" --output table --paginate
asc versions list --app "APP_ID" --output json --pretty --paginate
rg "MARKETING_VERSION|PRODUCT_BUNDLE_IDENTIFIER" "*.xcodeproj/project.pbxproj"
```

2. Open the new version for every included platform
   - Discover the current `asc` version-create command with `asc versions --help` and related subcommand help
   - Create the new app-store version only when it does not already exist
   - If the target version already exists, reuse its `VERSION_ID` and continue with localization updates
   - Store every resolved target `VERSION_ID`

3. Resolve localizations for every included platform
   - Use all existing localizations by default
   - Narrow localizations only when the user asks
   - Create missing target-version localizations when needed
   - Inspect source and target localizations before copying fields:

```bash
asc localizations list --version "VERSION_ID" --output json --pretty
asc localizations list --version "VERSION_ID" --output table
```

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

   - For small targeted edits, use the current quick-edit command discovered with `asc localizations update --help`, `asc versions update --help`, or related help
   - Version-localization quick edits commonly look like this:

```bash
asc localizations update --version "VERSION_ID" --locale "LOCALE" --promotional-text "PROMOTIONAL_TEXT" --whats-new "CHANGELOG_TEXT" --output table
```

   - After quick edits, pull remote metadata to a temporary directory and validate the current App Store Connect state:

```bash
TMP_DIR="$(mktemp -d)"
asc metadata pull --app "APP_ID" --version "NEW_VERSION" --platform PLATFORM --dir "$TMP_DIR" --force --output table
asc metadata validate --dir "$TMP_DIR" --output table
```

   - Always validate after editing and before submission

7. Stop
   - Report the prepared platforms, version IDs, localizations, and validation state
   - If the user has not provided a build number yet, also report the latest Xcode Cloud build number and commit message
   - Ask the user for the build number
   - End the turn without attaching a build or submitting for review

   Discover the latest Xcode Cloud build with:

```bash
asc xcode-cloud --help
asc xcode-cloud workflows --help
asc xcode-cloud build-runs --help
asc xcode-cloud workflows list --app "APP_ID" --output json --pretty
asc xcode-cloud build-runs list --workflow-id "WORKFLOW_ID" --sort "-number" --limit 1 --output json --pretty
```

   Report `attributes.number`, `attributes.executionProgress`, `attributes.sourceCommit.commitSha`, and `attributes.sourceCommit.message` for the latest run

## Phase 2: Attach build and submit

Use this phase only when the user has provided the build number after Phase 1

1. Resolve the build for each relevant platform
   - Match the user-provided build number to the included platforms
   - Resolve the user-provided build number to a unique build ID before submitting
   - Confirm each build is processed and eligible for App Store release
   - If more than one build matches a platform, disambiguate before submitting
   - Discover current build command syntax before relying on it:

```bash
asc builds --help
asc builds list --help
```

   - Inspect the matching build with:

```bash
asc builds list --app "APP_ID" --version "NEW_VERSION" --build-number "BUILD_NUMBER" --platform PLATFORM --output json --pretty --paginate
asc builds list --app "APP_ID" --version "NEW_VERSION" --build-number "BUILD_NUMBER" --platform PLATFORM --output table --paginate
```

2. Select the build for release
   - Attach the resolved build to each target version
   - Run `asc review doctor` before submitting to check for blockers
   - Prefer `asc review submit` when it covers the flow:

```bash
asc review --help
asc review submit --help
asc review doctor --app "APP_ID" --output table
asc review submit --app "APP_ID" --version-id "VERSION_ID" --build "BUILD_ID" --platform PLATFORM --dry-run --output table
asc review submit --app "APP_ID" --version-id "VERSION_ID" --build "BUILD_ID" --platform PLATFORM --confirm --output table
```

   - Use `--version-id "VERSION_ID"` instead of `--version` when that is more deterministic

3. Submit for App Review
   - Submit every included platform version after the dry run is clean
   - Capture submission IDs and review status
   - Report submitted platforms, selected builds, and submission IDs
   - Verify the final state with:

```bash
asc versions list --app "APP_ID" --version "NEW_VERSION" --platform PLATFORM --output table
asc versions view --version-id "VERSION_ID" --include-build --include-submission --output table
asc review submissions-get --id "SUBMISSION_ID" --output table
asc review status --app "APP_ID" --output table
```

## Fallbacks

- If the public API does not support a required release step, identify whether an `asc web ...` command exists and say that it is an experimental web-session fallback before using it
- If no CLI path exists for a required App Store Connect action, stop and give the exact manual step needed
- If validation reports blockers, fix API-addressable metadata issues, then re-run validation before continuing
