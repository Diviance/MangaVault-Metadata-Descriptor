# MVMD Concept: External Media Metadata Descriptor

## Purpose

This document captures an early concept for an `mvmd` file type: a MangaVault metadata descriptor that lives beside image files instead of containing the image bytes directly.

The broad idea is to create a richer, MangaVault-native sidecar format: something in the spirit of `ComicInfo.xml`, but capable of describing series, volumes, chapters, pages, labels, hashes, dimensions, and analysis results in a way that MangaVault and external tools can refresh and update over time.

This is not a final spec. It is a design note for future exploration.

## Core Idea

An `mvmd` file would act as the portable source of truth for a folder of manga images.

The image files would remain normal files on disk, usually in the same directory tree as the `mvmd` file. The `mvmd` file would store metadata and structure for those images, including:

- Series metadata
- Volume structure
- Chapter structure
- Page order
- Relative image paths
- Content hashes
- Image dimensions
- File sizes
- Image format or codec
- Labels such as `Uncensored`, `Upscaled`, `Color`, or other user-defined markers
- Analysis results that MangaVault would otherwise store only in the database

The database would still exist, but it would behave more like a cache/index of the descriptor and media files. A refresh operation would reconcile the folder, `mvmd`, and database state.

## Relationship To MVARC

There are two possible paths:

- Extend `mvarc` with an external-media mode
- Create a separate `mvmd` sidecar/project format

The separate `mvmd` file type currently feels cleaner.

`mvarc` can remain the self-contained archive format: portable, sealed, and responsible for storing its own image bytes.

`mvmd` can become the editable sidecar/project format: useful for folders of images that may be renamed, upscaled, replaced, repaired, or edited by external tools.

That split would allow both workflows without making `mvarc` ambiguous.

## Intended Workflows

### Upscaling Or Replacement

A user could keep a folder of source images with an `mvmd` file. If the images are later upscaled or replaced, MangaVault could refresh the folder, detect changed files, update hashes and dimensions, and preserve the volume/chapter/page metadata.

Useful cases:

- Upscaling existing pages
- Replacing damaged or missing pages
- Swapping image formats, such as PNG/JPEG to AVIF/JXL
- Re-running page analysis after image changes
- Preserving curation metadata while media bytes evolve

### Plex-Style Refresh

MangaVault or a desktop editor could expose a refresh operation similar to Plex library refresh:

1. Open the `mvmd` file.
2. Scan referenced folders and image files.
3. Compare known metadata against the current filesystem.
4. Re-hash and re-analyze only changed or unknown files.
5. Update the `mvmd` file.
6. Update the MangaVault database from the refreshed descriptor.

The important design point is that refresh should be incremental. It should not fully re-analyze every image unless explicitly requested.

### External Desktop Editor

A custom desktop tool could open and edit `mvmd` directly.

Possible editor features:

- Edit series, volume, and chapter metadata
- Reorder pages
- Assign pages to volumes and chapters
- Add or remove labels
- Update titles and numbering
- Validate missing or changed files
- Trigger MangaVault refresh after saving

This would make `mvmd` a useful interchange/project format, not just an internal backend artifact.

## Filesystem Organization

`mvmd` should support both automatic structure inference and manual metadata editing.

### Folder-Based Structure

One possible layout:

```text
Manga Series Main Folder/
  series.mvmd
  Volume 01 - The Journey Begins Arc/
    Chapter 01 - The Town of Beginnings [Uncensored] [Upscaled]/
      chapter_images/
        001.avif
        002.avif
        003.avif
```

In this model, MangaVault could infer:

- Volume number: `01`
- Volume title: `The Journey Begins Arc`
- Chapter number: `01`
- Chapter title: `The Town of Beginnings`
- Labels: `Uncensored`, `Upscaled`
- Page order: image filename order inside `chapter_images`

### Filename-Based Structure

Another possible layout:

```text
Manga Series Main Folder/
  series.mvmd
  Volume 01 - The Journey Begins Arc - Chapter 01 - The Town of Beginnings [Uncensored] [Upscaled] - 001.avif
  Volume 01 - The Journey Begins Arc - Chapter 01 - The Town of Beginnings [Uncensored] [Upscaled] - 002.avif
  Volume 01 - The Journey Begins Arc - Chapter 01 - The Town of Beginnings [Uncensored] [Upscaled] - 003.avif
```

In this model, MangaVault could infer the same structure from filenames instead of folders.

### Manual Structure

Users should not be required to follow a naming convention.

A folder could contain simply numbered images:

```text
Manga Series Main Folder/
  series.mvmd
  0001.avif
  0002.avif
  0003.avif
```

The user could then manually set volume numbers, chapter names, labels, and page assignments in MangaVault's web metadata editor or a dedicated `mvmd` editor.

## Parsing Philosophy

Path and filename parsing should be treated as an import convenience, not the permanent source of truth.

Once imported, the `mvmd` file should store normalized structure explicitly. This avoids making important metadata fragile just because a folder was renamed.

For example, the descriptor should store:

- `volumeNumber`
- `volumeTitle`
- `chapterNumber`
- `chapterTitle`
- `labels`
- `pageIndex`
- `relativePath`
- `contentHash`
- `width`
- `height`
- `format`

If a path changes later, the refresh process can attempt to reconnect the file by hash, partial metadata match, or user confirmation.

## Performance Notes

Keeping images outside the descriptor should not create a major performance problem if the refresh system is designed well.

The expensive operations are mostly the same whether images are embedded in an archive or stored externally:

- Directory scanning
- File stat calls
- Hashing image bytes
- Reading image dimensions
- Decoding images for thumbnailing or analysis

A self-contained archive may be simpler to move around and can be faster to enumerate in some cases, but an external image tree is still practical.

The main performance rule is to avoid unnecessary work:

- Store relative paths, file sizes, modification times, hashes, and analysis timestamps.
- Use cheap checks first.
- Re-hash only when size or modified time suggests a file changed, or when validation mode requires it.
- Re-analyze only files whose bytes or relevant metadata changed.
- Allow a full verification mode for users who want maximum certainty.

## Validation States

Because external media can drift from the descriptor, `mvmd` should support clear validation states.

Possible states:

- File exists and hash matches
- File exists but hash changed
- File is missing
- File appears to have moved
- File is new and untracked
- File dimensions changed
- File format changed
- Page order changed
- Duplicate file detected
- Possible replacement candidate found

The refresh UI should surface these states in a way that allows the user to accept, reject, or manually resolve changes.

## Portability Tradeoffs

### Self-Contained MVARC

Advantages:

- Easy to move as one file
- Harder to accidentally break
- Stronger integrity guarantees
- Cleaner long-term preservation story

Disadvantages:

- Less convenient for image editors, upscalers, and external scripts
- Replacing images may require archive-specific tooling
- Less transparent to users browsing files directly

### MVMD Plus External Images

Advantages:

- Easy to edit with normal tools
- Friendly to upscaling and repair workflows
- Human-inspectable folder structure
- Better fit for a desktop metadata editor
- Can act as a project format before producing final archives

Disadvantages:

- Easier to break by moving or deleting files
- Needs validation and reconciliation logic
- Requires careful relative path handling
- More sensitive to sync tools and partial copies

## Possible Descriptor Contents

A future `mvmd` schema could include:

- Format/version information
- Series metadata
- Contributor metadata
- Volumes
- Chapters
- Pages
- Relative file references
- Stable page IDs
- Cryptographic content hashes
- Optional perceptual hashes
- Image dimensions
- Image format and file size
- User labels and tags
- Analysis metadata
- Cover references
- Import/parser hints
- Last refresh information
- Unknown/custom extension fields

Stable IDs are important. A page should be able to keep its identity even if its path or image bytes change, as long as the user or refresh process confirms it is the same logical page.

## Open Questions

- Should `mvmd` be JSON, MessagePack, SQLite, or another format?
- Should it be fully human-editable, or optimized for tool safety and compactness?
- Should MangaVault support both a human-readable `.mvmd.json` and a compact `.mvmd`?
- How much path parsing should be standardized versus user-configurable?
- How should conflicting metadata be resolved between folder names, filenames, `mvmd`, and the database?
- Should refresh update `mvmd` automatically, or stage changes for user approval?
- Should `mvmd` be allowed to reference files outside its own directory tree?
- Should it support multiple image variants for the same logical page?
- Should it support generating a sealed `mvarc` from an `mvmd` project?

## Current Lean

The current preferred direction is:

- Keep `mvarc` as the self-contained archive format.
- Introduce `mvmd` as an editable external-media descriptor format.
- Treat path and filename parsing as import helpers.
- Treat the normalized `mvmd` structure as the portable source of truth.
- Treat the MangaVault database as a searchable/cacheable index of the descriptor.
- Use incremental refresh to keep performance reasonable.

This would let MangaVault support both preservation-style archives and flexible editing/project workflows without forcing one model to do both jobs.
