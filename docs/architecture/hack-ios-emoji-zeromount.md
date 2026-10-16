# Hack + iOS Emoji + ZeroMount Integration Plan

Status: design contract only; no runtime behavior changed.

## Goals

- Make Hack the selected primary text family while retaining Android's fallback coverage.
- Make one user-selected iOS emoji font the authoritative emoji source.
- Avoid having the standalone iOS Emoji module, MFGA, and ZeroMount all compete to replace the same font paths.
- Keep changes reversible and preserve the stock font/emoji files.
- Keep Google Play Services (GMS) font suppression separate and disabled by default.

## Responsibility boundaries

### aOpenFont (font configuration and policy)

Owns:
- Hack Regular, Bold, Italic, and Bold Italic assets and license notices.
- Validation of the four files and font metadata before packaging.
- A conservative system-font configuration overlay that changes the primary sans family only; it must not remove Noto fallback, emoji fallback, CJK, symbols, or accessibility coverage.
- An explicit, reversible policy manifest describing optional font exclusions. It must not delete stock files.

Does not own:
- A competing emoji mount engine.
- Direct edits to /system or persistent changes to stock partitions.
- Automatic GMS font blocking.

### ZeroMount integration (mount ownership)

ZeroMount already has an emoji staging/apply path, an emoji enabled setting, target discovery, conflict reporting, and optional per-app overrides. Prefer extending that existing subsystem over adding a second emoji mount implementation to aOpenFont.

The current upstream implementation stages a file named `NotoColorEmoji.ttf` and its target discovery includes multiple emoji-font variants. A port should not blindly map one iOS font file onto every legacy/vendor emoji filename: the files may use different glyph sets or font formats.

Proposed ZeroMount changes, to be implemented in a dedicated fork/branch:
1. Add a validated emoji-source manifest (source filename, SHA-256, license/source, intended target paths, and whether app-level overrides are allowed).
2. Support the iOS emoji payload as an explicit source option, not as an implicit replacement for every discovered variant.
3. Default to the canonical active emoji target discovered from the device's font configuration. Treat legacy/vendor variants as opt-in targets only after compatibility checks.
4. Detect active modules that provide any overlapping emoji target and show a conflict before applying. Never uninstall, disable, or mark another module for removal automatically.
5. Keep the existing enable/disable setting, and add an apply/restore status that reports exact targets and source hash.
6. Before applying, record the original target mapping and active mount strategy. Restore the previous state on disable, failure, or rollback.
7. Keep app-level GBoard/Facebook/GMS overrides separate from the global system emoji replacement. They must be opt-in and independently reversible.
8. Do not modify Play Integrity Fix, GMS packages, Advanced Account Protection, bootloader state, or system partitions.

## Migration plan for this device

1. Read-only inventory of the active ZeroMount version/config, iOS Emoji module payload/hash, MFGA state, and actual Android 16 emoji-family XML references.
2. Compare the selected iOS font's file format, glyph coverage, and license with the device's active font configuration.
3. Implement and test the ZeroMount fork off-device, including malformed manifest, missing font, conflict, disable/restore, and failed-apply cases.
4. Build an aOpenFont package that owns Hack only; document ZeroMount as the optional emoji backend.
5. Test on a copied config or emulator/test device before enabling either replacement on the Moto G.
6. Only after validation, request explicit approval before changing the phone's module state.

## Acceptance checks

- Only one module is authoritative for each target path.
- No stock font file is deleted or overwritten on the read-only system partition.
- No target is changed unless it exists in the generated, reviewed plan.
- Every apply operation has a recorded rollback path.
- A conflict never triggers automatic module removal/disable.
- Emoji fallback and non-emoji glyph coverage remain intact.
- GMS blocking is off by default and cannot silently change PIF-related settings.
- No device changes occur as part of building or testing this repository.
