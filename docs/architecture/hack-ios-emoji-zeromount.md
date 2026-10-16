# Hack + FancyMoji + ZeroMount Integration Plan

Status: architecture decision recorded; no runtime behavior or phone settings changed.

## Decision

Port the useful FancyMoji feature set into the ZeroMount integration rather than requiring a second emoji replacement module. FancyMoji is the preferred feature reference because it supports selecting among iOS, One UI, AOSP/Noto, and custom TTF emoji sources. The iOS 26.4 / Unicode 17 source is the initial target option.

Upstream references:
- FancyMoji: https://github.com/newhertz/FancyMoji
- FancyMoji README and feature/changelog documentation: https://github.com/newhertz/FancyMoji/blob/main/README.md
- ZeroMount: https://github.com/Enginex0/zeromount
- Hack: https://github.com/source-foundry/hack

FancyMoji's repository is MIT-licensed, but that does not automatically establish the redistribution license for every bundled emoji font. Audit the actual selected font asset and its upstream attribution/license before redistributing it.

## Goals

- Make Hack the selected primary text family while retaining Android's fallback coverage.
- Offer user-selectable iOS 26.4, One UI, AOSP/Noto, and custom emoji font sources through one ZeroMount-owned integration.
- Avoid having a standalone iOS Emoji module, MFGA, FancyMoji, and ZeroMount compete to replace the same font paths.
- Keep changes reversible and preserve stock font/emoji files.
- Keep Google Play Services (GMS) font suppression separate, opt-in, and off by default.

## Responsibility boundaries

### aOpenFont (font configuration and policy)

Owns:
- Hack Regular, Bold, Italic, and Bold Italic assets and license notices.
- Validation of the four Hack files and font metadata before packaging.
- A conservative system-font configuration overlay that changes the primary sans family only; it must not remove Noto fallback, emoji fallback, CJK, symbols, or accessibility coverage.
- An explicit, reversible policy manifest describing optional font exclusions. It must not delete stock files.

Does not own:
- A competing emoji mount engine.
- Direct edits to /system or persistent changes to stock partitions.
- Automatic GMS font blocking.

### ZeroMount integration (single emoji mount owner)

ZeroMount already has an emoji staging/apply path, an emoji-enabled setting, target discovery, conflict reporting, and optional per-app overrides. Extend that subsystem rather than adding a second emoji mount implementation to aOpenFont.

Implement the following FancyMoji-inspired features in a dedicated ZeroMount fork/branch:

1. **Source selector:** choices for iOS 26.4, One UI, AOSP/Noto, and custom TTF. Keep the selected source in an explicit config value; do not require a volume-key-only installer.
2. **Validated custom sources:** accept a custom font only after checking it is a regular file, readable, has an allowed extension, passes font parsing/validation, and matches a recorded SHA-256. Reject malformed or truncated files with a clear error.
3. **Source manifest:** record source filename, SHA-256, attribution/license, Unicode coverage metadata when available, intended target paths, and whether app-level overrides are allowed.
4. **Target discovery:** derive active emoji font targets from the device's actual font configuration and ZeroMount's supported target list. Do not blindly map one iOS font onto every legacy/vendor filename. Treat legacy/vendor variants as opt-in targets only after compatibility checks.
5. **Conflict handling:** identify enabled modules that overlap a planned target and show the exact conflicting module/path. Report only; never uninstall, disable, or mark another module for removal automatically.
6. **Apply/restore transaction:** generate and display a target plan before applying. Record original target mappings, selected source/hash, and active mount strategy. Apply only listed targets. Restore on disable, failed apply, or rollback.
7. **App-specific overrides:** preserve the useful FancyMoji concept for Facebook/Messenger and Gboard, but make every app override independently opt-in and reversible. Do not recursively replace every file matching *emoji*.ttf in /data/data or /data/user/0.
8. **Cache handling:** prefer explicit, app-scoped force-stop/cache refresh guidance. Do not recursively delete broad application cache/data paths as an automatic boot action.
9. **GMS behavior:** never disable GMS font provider/updater components, remove /data/fonts, or delete GMS font directories automatically. If later implemented, GMS suppression must be a separate opt-in experiment with a clear warning and a recorded restore plan; default off.
10. **Status and diagnostics:** expose the selected source, source hash, discovered/planned/applied targets, conflicts, current strategy, and last apply/restore error through CLI/WebUI.
11. **Failure safety:** missing source, invalid manifest, unresolved conflict, mount failure, or unsupported target must fail closed and restore the previous state where possible.

### Porting policy

Port features and the selection/configuration model, not unsafe side effects wholesale. In particular, do not port FancyMoji's broad recursive app-font replacement, immutable Facebook font locks, permission-000 font directories, automatic GMS component disabling, or deletion of /data/fonts and GMS font directories as default behavior. These behaviors can interfere with app updates, Play Integrity Fix, or system font management.

Do not copy or redistribute a font asset until its actual source and redistribution terms have been verified independently of the module's source-code license.

## Migration plan

1. Read-only inventory of ZeroMount's active emoji implementation, UI/config types, module packaging, and tests.
2. Inspect FancyMoji's selector, custom-font path, bundled font filenames, and scripts; separate useful selection logic from destructive service behavior.
3. Create/use a dedicated fork and feature branch of ZeroMount. Keep all runtime changes off the upstream repository and off the phone during development.
4. Implement source selection, validation, target planning, conflict reporting, transactional apply/restore, and opt-in app overrides.
5. Add tests for malformed manifest, missing/corrupt custom font, hash mismatch, conflicting module, absent target, partial apply failure, disable/restore, and rollback.
6. Validate the selected iOS font's actual format, glyph coverage, attribution, and redistribution terms.
7. Keep aOpenFont responsible for Hack only; document ZeroMount as the optional emoji backend.
8. Test on a disposable emulator/test device or copied configuration before any Moto G rollout.
9. Phone changes require a separate explicit approval after validation.

## Acceptance checks

- Only one module is authoritative for each target path.
- No stock font file is deleted or overwritten on the read-only system partition.
- No target changes unless it appears in a generated, reviewable plan.
- Every apply operation has a recorded rollback path.
- Conflicts never trigger automatic module removal or disabling.
- Emoji fallback and non-emoji glyph coverage remain intact.
- GMS blocking is off by default and cannot silently change Play Integrity Fix-related settings.
- Custom font inputs are validated and hash-recorded.
- No phone changes occur as part of repository development or testing.
