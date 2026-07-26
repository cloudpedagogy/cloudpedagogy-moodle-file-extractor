# Technical guide

This guide documents the detailed behaviour of `extract_moodle_files.py`.
For an introduction and quick start, see the [README](../README.md).

## How extraction works

A Moodle backup commonly contains:

- `files.xml`, which records filenames, paths, MIME types, sizes, content
  hashes and Moodle identifiers;
- content stored under hash-based paths such as `files/<contenthash>`;
- course, section and activity metadata;
- Moodle Book chapter and question metadata, when included.

The extractor parses this metadata, resolves each non-directory file record to
its stored content and writes one or more organisational views.

It supports:

- ZIP-based `.mbz` archives;
- TAR-based `.mbz` archives;
- already extracted Moodle backup directories;
- common flat and two-level Moodle hash-storage layouts.

No course-specific identifiers or names are embedded in the implementation.

## Command-line interface

```text
usage: extract_moodle_files.py [-h] [--output OUTPUT]
                               [--mode {context,course,type,all}]
                               [--by-context] [--by-course] [--by-type]
                               [--flat] [--include-directories]
                               [--verify-hashes]
                               [--link-mode {copy,hardlink}]
                               [--overwrite-output]
                               backup
```

| Argument | Behaviour |
|---|---|
| `backup` | Moodle `.mbz` archive or extracted backup directory |
| `--output`, `-o` | Destination directory |
| `--mode context` | Generate `files_by_moodle_context/` |
| `--mode course` | Generate `resource_bundle/` |
| `--mode type` | Generate `resource_bundle_by_type/` |
| `--mode all` | Generate all three views |
| `--by-context` | Add the context view to the selected mode |
| `--by-course` | Add the course view to the selected mode |
| `--by-type` | Add the type view to the selected mode |
| `--flat` | Deprecated alias that adds the type view; use `--mode type` in new commands |
| `--include-directories` | Retain Moodle directory records in the CSV manifest |
| `--verify-hashes` | Recalculate and compare SHA-1 hashes |
| `--link-mode copy` | Write portable copies; the default |
| `--link-mode hardlink` | Use hard links where the filesystem permits |
| `--overwrite-output` | Replace recognised outputs from an earlier extraction |

The default mode is `context`. Every mode writes `resource_manifest.csv` and
`extraction_report.md`.

### Recommended command

```bash
python3 extract_moodle_files.py \
  "path/to/course-backup.mbz" \
  --output "output/course-resources" \
  --mode all \
  --verify-hashes
```

### Running from a larger project

```bash
python3 src/course_generator/tools/extract_moodle_files.py \
  "imports/moodle_courses/source_mbz/backup-moodle2-course-example.mbz" \
  --output "output/moodle_courses/example-resources" \
  --mode all \
  --verify-hashes
```

### Using an extracted backup directory

```bash
python3 extract_moodle_files.py \
  "path/to/extracted-backup" \
  --output "output/course-resources" \
  --mode all \
  --verify-hashes
```

## Resource views

### Moodle-context view

`files_by_moodle_context/` preserves technical provenance:

```text
files_by_moodle_context/
â””â”€â”€ component/
    â””â”€â”€ filearea/
        â””â”€â”€ itemid/
            â””â”€â”€ original_filename.pdf
```

Common examples include:

| Context path | Typical meaning |
|---|---|
| `course/section/â€¦` | File embedded in a section description |
| `mod_resource/content/â€¦` | Individual Moodle resource |
| `mod_folder/content/â€¦` | File in a Folder activity |
| `mod_book/chapter/â€¦` | File embedded in a Book chapter |
| `question/questiontext/â€¦` | File embedded in question text |

This is the authoritative file view when course metadata is incomplete or an
activity plugin is not recognised.

### Course-organised view

`resource_bundle/` attempts to produce readable paths:

```text
resource_bundle/
â””â”€â”€ Section name/
    â””â”€â”€ Activity type - Activity name/
        â””â”€â”€ Chapter name/
            â””â”€â”€ original_filename.pdf
```

Mapping is best-effort. It uses recognised section, activity, Moodle Book
chapter and question metadata. The extractor does not generically interpret
every third-party Moodle plugin.

When a readable course mapping cannot be determined, the recovered file is
placed under `Unresolved/`. This indicates incomplete classification, not
necessarily a missing or damaged file.

### File-type view

`resource_bundle_by_type/` groups files by extension and MIME type. Supported
category names are:

```text
PDFs/
Word/
PowerPoint/
Spreadsheets/
Data/
R_scripts/
Code/
Images/
Audio/
Video/
Archives/
Text/
Other/
```

`Other/` retains content that cannot be classified reliably.

## Manifest

`resource_manifest.csv` is written with UTF-8 BOM encoding for convenient use
in spreadsheet software.

Its columns include:

| Field group | Examples |
|---|---|
| Moodle record | `file_id`, `contenthash`, `component`, `filearea`, `itemid`, `contextid` |
| Original file | `filename`, `filepath`, `mimetype`, `filesize` |
| Descriptive metadata | `author`, `license`, `source`, timestamps |
| Course context | section, activity and subcontext labels where available |
| Generated paths | context, course and type output paths for selected modes |
| Extraction result | status, hash result and notes |

The course-organised path contains an `Unresolved` location when contextual
mapping was not possible; there is no separate mapping-status field.

Directory records are excluded by default. With `--include-directories`, their
metadata is retained with a `directory_record` status, but no downloadable file
is created.

## Extraction report

`extraction_report.md` summarises:

- Moodle file records processed;
- directory records encountered;
- recovered files;
- missing stored content;
- SHA-1 hashes verified;
- hash mismatches;
- unresolved course mappings;
- duplicate content references;
- duplicate filenames;
- collision-renamed files;
- recovered file categories;
- Moodle components represented in the records;
- extraction warnings.

The report is the first output to review after a run.

## Recovery and verification

Recovery and hash verification are distinct:

1. A file is **recovered** when its stored content is found and written.
2. With `--verify-hashes`, its SHA-1 value is recalculated.
3. A matching value is counted as **verified**.
4. A mismatch is retained and reported as `hash_mismatch` for investigation.

The terminal summary follows this form:

```text
Recovered 147 files; 0 missing; 147 hashes verified; 0 hash mismatches; 8 course mappings unresolved.
Manifest: /path/to/resource_manifest.csv
Report:   /path/to/extraction_report.md
```

The actual figures depend on the backup and options.

Hash verification is recommended for archive recovery, migration and
quality-assurance work.

## Duplicate and collision handling

The extractor distinguishes several related conditions:

- The same content hash can be referenced by multiple Moodle records.
- Different files can share the same original filename.
- The same name can occur at the same generated destination.

No source record is intentionally overwritten. When generated paths collide,
the extractor appends a stable Moodle file identifier, for example:

```text
lecture-slides.pdf
lecture-slides__file-21016169.pdf
```

Do not deduplicate by filename alone. Compare content hashes and Moodle context
in the manifest. Empty filenames and missing content hashes are excluded from
duplicate calculations so they do not inflate the totals.

## Copy and hard-link modes

Copy mode is the default:

```bash
--link-mode copy
```

It produces portable, independent directory trees and is recommended when
outputs will be moved, archived or shared.

Hard-link mode can reduce additional disk use when the same recovered content
is represented in several views:

```bash
--link-mode hardlink
```

Hard links require compatible source and destination filesystems. The linked
paths refer to the same underlying file data, so this mode is best reserved for
local, space-conscious workflows where that behaviour is understood.

## Safe repeat runs

By default, extraction stops if the output directory is non-empty. Use a new
destination or intentionally regenerate an earlier output with:

```bash
--overwrite-output
```

The extractor recognises and removes only:

- `files_by_moodle_context/`;
- `resource_bundle/`;
- `resource_bundle_by_type/`;
- `resource_manifest.csv`;
- `extraction_report.md`.

It refuses to clean an output directory containing unrecognised items. The
source backup is never modified.

After extraction, the script checks automatically that the manifest, report
and all views requested by the selected modes exist before reporting success.

## Archive safeguards

For supported ZIP and TAR inputs, the extractor checks paths before extraction
to prevent content being written outside its temporary directory.

It also rejects unsafe link or special-file entries where they can be
identified, including ZIP symbolic links and TAR symbolic links, hard links and
device entries.

These safeguards protect the extraction process. They do not establish that
the educational content itself is safe or appropriate to open. Process only
backups obtained from trusted, authorised sources.

## Testing

### Compilation and interface

```bash
python3 -m py_compile extract_moodle_files.py
python3 extract_moodle_files.py --help
```

### Test all modes independently

```bash
MBZ_SOURCE="path/to/course-backup.mbz"

python3 extract_moodle_files.py "$MBZ_SOURCE" \
  --output "output/context-test" --mode context --verify-hashes \
&& python3 extract_moodle_files.py "$MBZ_SOURCE" \
  --output "output/course-test" --mode course --verify-hashes \
&& python3 extract_moodle_files.py "$MBZ_SOURCE" \
  --output "output/type-test" --mode type --verify-hashes \
&& python3 extract_moodle_files.py "$MBZ_SOURCE" \
  --output "output/all-test" --mode all --verify-hashes \
&& echo "All extraction-mode tests completed successfully"
```

For normal end-user use, a single `--mode all` run is sufficient.

### Inspect outputs

```bash
cat "output/all-test/extraction_report.md"
tree -L 3 "output/all-test"
```

If `tree` is unavailable:

```bash
find "output/all-test" -maxdepth 3 -print
```

## Troubleshooting

### Backup not found

List available backups and copy the exact path:

```bash
ls -lh path/to/backups/*.mbz
```

### Script not found

Run from the repository or project root, or locate the script:

```bash
find . -name "extract_moodle_files.py"
```

### Output directory is not empty

Choose a new output directory. Use `--overwrite-output` only when the
destination contains a previous extraction created by this tool. The safety
check will refuse a directory containing unrelated items.

### Files are missing

Review rows with a missing status in `resource_manifest.csv`, then inspect
`extraction_report.md`.

To inspect a TAR-based archive:

```bash
tar -tf "course-backup.mbz" | grep -E '(^|/)files/' | head -20
```

For a ZIP-based archive:

```bash
unzip -l "course-backup.mbz" | grep -E '(^|/)files/' | head -20
```

If content is genuinely absent from the archive, the extractor cannot recreate
it.

### Hash mismatch

A `hash_mismatch` means content was recovered but did not match Moodle's
recorded SHA-1 value. Check the source backup or recreate it before relying on
the affected resource.

### Unresolved course mapping

Use the file's context path, component, file area, item ID and content hash in
the manifest to investigate its origin. The underlying file may still have
been recovered and verified successfully.

### Directory records are absent

This is the default. Add `--include-directories` when their metadata is needed.

### Old commands using `--flat`

`--flat` remains as a deprecated alias that adds the type view. New commands
should use:

```bash
--mode type
```

## Technical limitations

- External resources referenced only by URL are not stored file content.
- Excluded backup content cannot be recovered.
- Moodle versions and third-party plugins may use metadata structures that are
  not mapped into a readable course path.
- Embedded Book and question images can lose meaning outside their surrounding
  HTML.
- Historical backups may contain repeated records or several references to the
  same stored content.
- File extraction does not reproduce Moodle activity logic, configuration,
  completion rules or interactivity.

The extractor prioritises recovery, provenance and explicit reporting over
guessing uncertain course relationships.