# Moodle File Extractor

Extract, verify and organise files from Moodle course backups (`.mbz`) without
having to restore the course in Moodle.

Moodle stores uploaded content in a backup under generated SHA-1 hashes rather
than recognisable filenames. This Python tool reads Moodle's metadata, locates
the stored content and reconstructs accessible copies using the original names
and available course context.

It works locally, uses only the Python standard library and never modifies the
source backup.

## Example output

Running the extractor with `--mode all` generates three complementary views of the recovered files:

- `files_by_moodle_context/` — organised by Moodle’s technical component and file-area metadata;
- `resource_bundle/` — organised by course section and activity;
- `resource_bundle_by_type/` — organised by file type.

It also generates:

- `resource_manifest.csv` — a searchable inventory of recovered files;
- `extraction_report.md` — a summary of extraction results, integrity checks and warnings.

![Example of one Moodle file extractor output](docs/images/screenshot.png)

*The screenshot shows one example output view. Additional course-organised and file-type views, together with the CSV manifest and extraction report, are also available when using `--mode all`.*


## Why use this tool?

Restoring a complete course is not always possible or necessary. You may only
need to inspect its resources, recover teaching files, prepare material for a
redesign, or create an inventory for audit and quality-assurance work.

The extractor makes the contents of a backup usable outside Moodle while
retaining technical provenance. It can help with:

- recovering resources from archived or inherited courses;
- reviewing material without access to a Moodle restore area;
- course redesign, migration and content rationalisation;
- locating PDFs, presentations, datasets, images, code and media;
- accessibility, copyright and quality-assurance reviews;
- identifying missing content, duplicates and filename collisions;
- feeding structured file information into audits or dashboards.

This complements Moodle restore rather than replacing it. It recovers stored
files and metadata, but it does not recreate the complete behaviour of Moodle
activities.

### Typical use cases

| Scenario | How the extractor helps |
|---|---|
| Reviewing an archived course | Makes its documents, images, datasets and other uploaded resources accessible without restoring the course |
| Inheriting an unfamiliar course | Produces a readable course-organised bundle and a searchable inventory of its files |
| Redesigning or rebuilding a course | Collects the existing teaching resources so they can be reviewed, rationalised and reused |
| Migrating to another course or platform | Recovers files with recognisable names rather than Moodle's hash-based storage names |
| Accessibility or copyright review | Groups resources into convenient file-type folders and records them in a CSV manifest |
| Investigating an incomplete backup | Reports missing content, hash failures, collisions and unresolved mappings rather than silently omitting them |
| Course auditing or analysis | Supplies structured metadata that can be filtered in a spreadsheet or linked to other audit and dashboard tools |
| Preserving course materials | Creates independent, verifiable resource copies while leaving the original `.mbz` unchanged |

For example, an academic reviewing an inherited course may prefer
`resource_bundle/`, which uses the available section, activity and Book chapter
names. A technical reviewer can use `files_by_moodle_context/` and
`resource_manifest.csv` to trace the same resources back to Moodle components
and identifiers.

## Key features

- Accepts a Moodle `.mbz` file or an already extracted backup directory.
- Supports common ZIP- and TAR-based Moodle backups.
- Restores original filenames and paths.
- Produces three complementary views of the recovered resources.
- Generates a searchable CSV manifest and Markdown extraction report.
- Optionally verifies recovered content against Moodle's SHA-1 hashes.
- Preserves files with duplicate names using collision-safe filenames.
- Retains uncertain course mappings under `Unresolved/`.
- Supports normal copies or space-saving hard links.
- Applies archive path and special-entry safety checks.
- Contains no course-specific names, IDs or folder mappings.

## Requirements

- Python 3.9 or later
- No third-party Python packages

## Quick start

From the root of the Learning Publisher project:

```bash
python3 src/course_generator/tools/extract_moodle_files.py \
  "imports/moodle_courses/source_mbz/backup-moodle2-course-example.mbz" \
  --output "output/moodle_courses/example_course_resources" \
  --mode all \
  --verify-hashes
```

If the extractor is being used as a standalone script:

```bash
python3 extract_moodle_files.py \
  "path/to/course-backup.mbz" \
  --output "output/course-resources" \
  --mode all \
  --verify-hashes
```

For routine use, `--mode all --verify-hashes` is recommended.

Replace the example backup and output names with the appropriate names for the
course being processed.

If the output already contains an earlier extraction made by this tool and you
intend to regenerate it, add:

```bash
--overwrite-output
```

The extractor will replace its recognised outputs but refuse to clean a
destination containing unrelated items.

## Outputs

Using `--mode all` creates:

```text
course-resources/
├── files_by_moodle_context/
├── resource_bundle/
├── resource_bundle_by_type/
├── resource_manifest.csv
└── extraction_report.md
```

| Output | Purpose |
|---|---|
| `files_by_moodle_context/` | Authoritative technical view organised by Moodle component, file area and item ID |
| `resource_bundle/` | Best-effort course view organised by readable sections, activities and Book chapters |
| `resource_bundle_by_type/` | Convenience view grouping files into categories such as PDFs, Word, data, code and images |
| `resource_manifest.csv` | Searchable inventory of Moodle records, extracted paths, hashes, statuses and warnings |
| `extraction_report.md` | Summary of recovered, verified, missing, duplicate, renamed and unresolved records |

The three folders are alternative organisational views of the same recovered
file records. They are not different sets of source content. Because the course
view depends on the metadata available in the backup, unresolved files are
retained rather than discarded.

Normal copies are the safest and most portable default. As three views of the
same files are generated by `--mode all`, the output can use roughly two or
three times the storage occupied by one extracted view. On a compatible local
filesystem, `--link-mode hardlink` can reduce this duplication, but linked
outputs are less portable if folders are moved or copied independently.

## Output modes

| Mode | View generated | Best suited to |
|---|---|---|
| `context` | `files_by_moodle_context/` | Technical audit and provenance |
| `course` | `resource_bundle/` | Academic review, redesign and migration |
| `type` | `resource_bundle_by_type/` | Format-specific review and conversion |
| `all` | All three views | Complete extraction |

Every mode also generates `resource_manifest.csv` and
`extraction_report.md`. If no mode is specified, `context` is used.

## Common options

```text
--output, -o PATH              Output directory
--mode {context,course,type,all}
--verify-hashes                Verify recovered files against Moodle SHA-1 hashes
--include-directories          Include directory metadata records in the manifest
--link-mode {copy,hardlink}    Use portable copies or space-saving hard links
--overwrite-output             Regenerate recognised extractor outputs safely
```

Display the complete current interface with:

```bash
python3 extract_moodle_files.py --help
```

## Interpreting the result

Check `extraction_report.md` first. Recovery and verification are separate:

- **Recovered** means the stored content was found and written.
- **Hash verified** means the recovered content also matched the SHA-1 value
  recorded by Moodle.
- **Missing** means the metadata record existed but its stored content could not
  be found.
- **Unresolved** means the file was recovered but could not be assigned
  confidently to a readable course location.

Directory records are skipped by default because they do not contain
downloadable file content. Add `--include-directories` only when their metadata
is useful for technical analysis.

A normal end user does not need to run a separate chain of shell tests after
every extraction. The important production check is `--verify-hashes`, followed
by reviewing `extraction_report.md`. Additional `test -d` and `test -f` commands
are mainly useful during development or deployment testing.

## Safe operation

The extractor:

- reads the source backup without changing it;
- does not connect to a live Moodle site;
- does not restore, edit or delete a Moodle course;
- refuses to reuse a non-empty output folder unless replacement is explicitly
  requested;
- preserves uncertain records and reports them instead of silently discarding
  them.

To regenerate recognised outputs intentionally, add `--overwrite-output`.
Using a new output directory remains the safest option for a first run.

## Limitations

- External links such as YouTube, Panopto and SharePoint are not downloadable
  files in the backup.
- A backup cannot supply content that was excluded when it was created.
- Human-readable course mapping is best-effort and varies with Moodle version,
  backup settings and plugins.
- Embedded images may have limited meaning without their surrounding activity
  or Book HTML.
- Recovering resources does not reconstruct Moodle activity behaviour.

Treat `files_by_moodle_context/` and `resource_manifest.csv` as the
authoritative provenance record. The course and file-type folders are
additional views designed for easier review and reuse.

## Technical documentation

For the complete command reference, output schema, mapping behaviour, archive
safeguards, duplicate handling, testing and troubleshooting, see the
[technical guide](docs/technical-guide.md).

## Licence

MIT
