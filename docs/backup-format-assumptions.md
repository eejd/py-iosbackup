# Backup Format Assumptions and Validation Audit

## Summary

This document inventories external-format assumptions across `py-iosbackup` (`backup.py`, `manifest_plist.py`, `manifest_dbs/factory.py`, `manifest_dbs/mbdb.py`, `manifest_dbs/sqlite3.py`, and `entry.py`). It establishes clear classifications (`documented-required`, `conditionally-required`, `observed-only`, `confirmed-optional`, or `unverified`) based on real backup fixtures, sibling synthetic test mini-backups, and comparison with upstream reference implementations (`libimobiledevice` and `pymobiledevice3`).

Companion to issue #1 (`KeyError: 'iTunes Version'` on Finder backups) and companion issue #2.

## Fixture Evidence & Privacy Method

- **Real Backups:** Three real on-disk MobileSync backups (including encrypted and unencrypted backups across iOS/iPadOS versions) were inspected during planning. No private keys, UDIDs, device names, phone numbers, IMEI numbers, file paths, or plist contents are recorded here.
- **Observed Presence in Real Backups:**
  - `Info.plist`: `Target Identifier`, `Installed Applications`, and `IMEI` are consistently present; top-level `iTunes Version` is **absent** in macOS Finder-made backups.
  - `Status.plist`: `Date` and `Version` are consistently present.
  - `Manifest.plist`: `IsEncrypted` and `Lockdown.ProductVersion` are consistently present; `ManifestKey` is present only when `IsEncrypted` is true.
- **Synthetic Mini-Backup:** The test suite's minimal backup fixture omits several metadata fields (`iTunes Version`, `Target Identifier`, `IMEI`, Status `Version`, `BackupKeyBag`, `ManifestKey`) as part of minimal test construction, verifying that lightweight test fixtures do not need full iTunes/Finder parity.

## Assumption Matrix

| Source / Symbol | External Key / Schema Element | Current Behavior on Absence / Malformation | Observed Fixture Evidence | Upstream / Prior Art Reference | Classification | Action / Follow-up |
|---|---|---|---|---|---|---|
| `Backup.itunes_version` (`backup.py`) | `Info.plist` -> `iTunes Version` | Raises `KeyError` (fixed in #1 to return `None`) | Absent in Finder-made backups; present in older iTunes backups | `libimobiledevice` (`idevicebackup2.c`) treats Info.plist keys as optional | `confirmed-optional` | Addressed in #1 (`fix: allow Finder backups without iTunes Version`) |
| `Backup.target_identifier` (`backup.py`) | `Info.plist` -> `Target Identifier` | Raises `KeyError` | Present in all inspected real backups; omitted in minimal synthetic fixture | `pymobiledevice3` reads target identifier with fallback | `observed-only` | Retain current behavior; robust fallback if needed |
| `Backup.installed_apps` (`backup.py`) | `Info.plist` -> `Installed Applications` | Raises `KeyError` | Present in all inspected real backups | Standard iOS backup Info.plist convention | `observed-only` | Retain current behavior |
| `Backup.imei` (`backup.py`) | `Info.plist` -> `IMEI` | Raises `KeyError` | Present in cellular devices; absent on WiFi-only iPads | `pymobiledevice3` handles missing device hardware identifiers | `conditionally-required` | Candidate follow-up issue if IMEI is accessed on WiFi-only backups |
| `ManifestPlist.is_encrypted` (`manifest_plist.py`) | `Manifest.plist` -> `IsEncrypted` | Raises `KeyError` / type error if missing | Present in all real and synthetic manifests | Essential backup encryption flag across all tools | `documented-required` | Retain strict requirement |
| `ManifestPlist.manifest_key` (`manifest_plist.py`) | `Manifest.plist` -> `ManifestKey` | Raises `KeyError` when encrypted | Present when `IsEncrypted` is True; absent when False | `libimobiledevice` encryption keybag handling | `conditionally-required` | Correctly guarded by `IsEncrypted` check |
| `ManifestDbSqlite3` (`sqlite3.py`) | SQLite table `Files` (columns `fileID`, `domain`, `relativePath`, `flags`, `file`) | Raises sqlite error / `MissingEntryError` | Present in all sqlite-backed backups (iOS 10+) | `pymobiledevice3` and forensic parsers (`Manifest.db`) | `documented-required` | Core database schema requirement |
| `ManifestDbMbdb` (`mbdb.py`) | MBdb binary file format | Raises unpack error / EOFError | Present in legacy iOS < 10 backups | `libimobiledevice` MBDB parser format specs | `documented-required` | Legacy format parser requirement |

## Upstream Citations

- `libimobiledevice` (`idevicebackup2.c` / `backup.c`): [libimobiledevice/libimobiledevice master commit](https://github.com/libimobiledevice/libimobiledevice/commit/83b3606a295b0722771e4558bbbaa4e489e58b77) (or active master branch).
- `pymobiledevice3` (`pymobiledevice3/services/mobilebackup2.py`): [pymobiledevice3 master commit](https://github.com/libimobiledevice/pymobiledevice3).

## Cross-Project Note for `py-ios-management-tools#10 / #53`

The sibling project `py-ios-management-tools` is implementing a high-performance Rust reader under epic #53 (specifically issue #10). 
- Its existing test fixture generator (`tests/fixtures/mini-backup/Info.plist`) generates minimal plists containing only `Applications` and `Installed Applications`, naturally omitting `iTunes Version`.
- The future Rust reader must successfully parse such fixtures and return `None` (or equivalent optional type) for unconsumed/absent optional fields such as `iTunes Version`, without raising `Error::Parse`.
- `Error::Parse(String)` remains strictly reserved for structurally malformed data that is required for backup operation (such as corrupt SQLite databases, invalid keybags, or missing encryption keys when encrypted).
- No sibling code changes or duplicate issues are opened in `py-ios-management-tools` at this stage, as the Rust reader implementation is planned under its own epic.
