# TreeWrite file format specification

## Goals of this document

This document describes a format, called **TreeWrite file format**, for storing bullet points in a `.jsonl` ([JSON Lines](https://jsonlines.org)) file

The purpose of this format is to propose an open standard by which [outliner applications](https://en.wikipedia.org/wiki/Outliner) store and process bullet points of various types (text, images, code, quotes, etc.) in a text file

The design goal is to have a transparent, human-readable format that is backward compatible with previous versions, so that notes written today can still be accessed by outliners 20 years from now. We prioritize simplicity and stability over abrupt changes

The TreeWrite file format was created specifically for the [TreeWrite text editor](https://treewrite.com), but since it's an open format, it can be used by any other tools. This document also serves as a guide for implementing a TreeWrite format parser. See the [official TreeWrite parser](https://github.com/treewrite/parser)

It's an open format, meaning any outliner or developer can use this format for any purpose. The specification license is [CC0 1.0 Universal](https://github.com/treewrite/spec/blob/main/LICENSE)

To understand why we created a new specification instead of using an existing one (such as [OPML 2.0](https://opml.org/spec2.opml)), see [this section](#why-jsonl)

## Conventions used in this document

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)

Only the capitalized forms of these words (e.g. `MUST`, not `must`) carry normative meaning in this document. Lowercase occurrences are used with their ordinary English meaning and impose no compliance requirement

## The `.jsonl` file

A `.jsonl` file (a.k.a. [JSON Lines](https://jsonlines.org)) is a known file format that stores one valid JSON per line of the file (separated by `\n`). Each line is treated as a valid and independent JSON

In TreeWrite format, each line is a JSON object. The first line contains file metadata, and each subsequent line represents a single bullet point

```jsonl
{ ... } // Metadata
{ ... } // First bullet point
{ ... } // Second bullet point
{ ... } // Third bullet point
{ ... } // ...
```

A parser MUST ignore empty lines and lines containing only whitespace. A parser MUST ignore a `\r` immediately before the `\n` line separator, so that files with `\r\n` line endings are accepted. A parser MUST ignore a UTF-8 byte order mark (BOM) at the start of the file. The last line of the file MAY omit the trailing `\n`

## Why `.jsonl`?

There are motivations behind the decision to use `.jsonl` and not other storage methods. Here they are:

1. **Text file:** The `.jsonl` file is a text file, not a binary one, which makes it more accessible for manual editing and integration with tools like [Git](https://git-scm.com)
1. **File diffs:** changing a bullet point is equivalent to changing a single line of the file, which makes it easier to view file diffs in tools like Git
1. **Merge by line:** Text merge tools operate on lines. Two devices editing different bullet points never produce conflicts. A conflict only exists when editing the same bullet point, and the resolution remains local to that line
1. **Isolated corruption:** A poorly formatted line (interrupted writing, incorrect manual editing) invalidates a single bullet point, not all of them
1. **Centralization:** All bullet points are stored in a single file, instead of being scattered across multiple files
1. **Predictable scalability:** The cost of any operation (write, read) is linear to the number and size of bullet points, with no operations affected by the depth of the tree (as would be the case in nested JSON recursion or a Markdown list)

## Metadata format

The first line of the file is a JSON object containing file metadata. The object has the following fields:

| Field     | Description                                                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version` | The current major version (ex. 1) of the specification that the file is following. See the [Versioning](#versioning) section for more information |

## Bullet point format

All lines in the file, except the first, represent bullet points. Each bullet point is a JSON object. All bullet points have fields in common, in addition to fields specific to each bullet point type. The following fields are common to all bullet points:

| Field        | Description                                                                                                                                                             |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`         | A [ULID](https://github.com/ulid/spec) to differentiate the bullet points                                                                                               |
| `updated_at` | Last bullet point update date. Represented by a number, the total milliseconds since the [Unix epoch](https://en.wikipedia.org/wiki/Unix_time)                          |
| `parent`     | The ULID of the parent bullet point (which allows for a tree structure). It MUST be null if it is not nested                                                            |
| `order`      | A sequential number, starting from zero, representing the bullet point index in the direct children of its parent. (ex. 2, meaning that it is the parent's third child) |
| `type`       | A string with the type that the bullet point represents (ex. text, code, etc.). See the complete [list of types](#bullet-point-types)                                   |

ULIDs, in both `id` and `parent`, MUST be written in uppercase, which is their canonical form. A parser MUST accept lowercase ULIDs when reading, and MUST compare ULIDs case-insensitively when resolving `parent`, detecting duplicate `id`, breaking cycles and ordering siblings

## Versioning

The specification follows [semantic versioning](https://semver.org). The `version` field in the metadata contains the current major version of the file

If the version is larger than what some parser currently supports (ex. metadata with version 2 with parser implementing major version 1), then the file MUST NOT be opened by the parser, since the parser is trying to read a version that is not compatible with its implementation

For example, if in a major version 2 we rename a [bullet point type](#bullet-point-types), a parser following major version 1 will not know about this renaming and will incorrectly ignore the bullet point

However, if the parser implements a major version larger than the file's version, it MUST be able to open and edit the older major version. In other words, a parser that supports a major version X MUST support all major versions older than X

When writing a file, a parser MUST write the major version it implements. Since a parser only opens files whose `version` is less than or equal to the one it implements, writing either keeps the file's `version` or upgrades it. When a file's `version` is greater than the one the parser implements, the parser MUST NOT open it, and the outliner application SHOULD tell the user that a newer version of the parser is required

## Bullet point types

### `text`

The `text` type represents any single-paragraph text (without new lines). New lines are not supported as they can easily be represented by new bullets. It has the following fields:

| Field  | Description                                                                             |
| ------ | --------------------------------------------------------------------------------------- |
| `text` | A string without newlines (`\n`). Accepts [Inline markdown text](#inline-markdown-text) |

A parser MUST replace all `\n` with a single space if it encounters them in the `text` field

### `quote`

The `quote` type represents a quote from an author. It has the following fields:

| Field    | Description                                                                                           |
| -------- | ----------------------------------------------------------------------------------------------------- |
| `text`   | The quotation itself. Accepts [Inline markdown text](#inline-markdown-text)                           |
| `author` | The name of the author who quoted this quote. Semantically, it can be any source (person, book, etc.) |

### `divider`

The `divider` type does not have any custom fields. It represents a line that visually divides the user interface

## Inline markdown text

The content of some bullet fields may support inline markdown. This means that the outliner application MUST render/style this markdown in the user interface

The following markdown syntax MUST be supported:

- `**bold**`
- `*italic*`
- `***italic & bold***`
- `[Some link](https://example.com)`

## Handling inconsistencies

This section lists and details some inconsistency scenarios that could (but should not) eventually be found in a TreeWrite file, and how a parser should behave in each scenario

### Order of application

The result of loading a file depends on the order in which the rules below are applied. A parser MUST apply them in the following order:

1. Discard malformed lines (invalid JSON, JSON that is not an object, invalid `id`, required fields missing or with the wrong JSON type)
1. Resolve duplicate `id`
1. Resolve `parent` values that are absent, invalid, or point to an `id` that does not exist
1. Break cycles
1. Sort siblings by `order`, using `id` as the tiebreaker

### `parent` points to an `id` that does not exist in the file

A parser MUST treat it as `parent: null`, promoting the bullet to the root. A parser MUST NOT discard the bullet, and MUST NOT fail to load the rest of the file because of it

### `parent` field absent

A parser MUST treat it as `parent: null`. This is the only required field whose absence does not invalidate the line

### `parent` with an invalid value

If `parent` is present but is neither `null` nor a valid ULID (for example a string that is not a ULID, or a number), a parser MUST treat it as a `parent` that points to an `id` that does not exist in the file, promoting the bullet to the root

### Cycle in the tree (A is the parent of B, B is the parent of C, C is the parent of A)

It can be detected during tree loading. Upon finding a cycle, a parser MUST break it by taking the bullet with the highest `id` among the bullets that form the cycle and treating it as `parent: null`. The highest `id` is the lexicographically greatest ULID in its uppercase form. Bullets that descend from the cycle but do not belong to it keep their `parent`. If the file contains more than one cycle, this rule applies to each of them independently

### Duplicate `id`

Two or more bullets with the same `id`. The bullet on the last line of the file MUST be considered, discarding the other repeated bullets

### Line with syntactically invalid/malformed JSON

A parser MUST discard the poorly formatted bullet line. A parser MUST NOT fail to load the rest of the file because of it

A line containing valid JSON that is not a JSON object (for example an array, a string, a number or `null`) MUST also be treated as malformed JSON

### `id` that is not a valid ULID

The entire bullet point is invalid. The parser MUST treat it the same way as a malformed JSON

### Required bullet field missing or with the wrong JSON type

Invalid line, treated as malformed. Unlike an unknown field (which is preserved), a missing required field prevents the reconstruction of a minimally coherent bullet point, so the line cannot be promoted to a valid bullet point

A required field whose value has a JSON type different from the one specified MUST be treated as a missing required field. The only exception is `parent`, see [`parent` field absent](#parent-field-absent) and [`parent` with an invalid value](#parent-with-an-invalid-value)

### Unknown `type`

A bullet point indicating an unknown type (not mentioned in this specification) MUST be preserved by the parser, keeping its required fields intact within the tree. The parser MUST NOT discard the bullet or remove it from the tree. The outliner application SHOULD render it with a generic fallback (e.g. displaying the raw content or a "type not supported" notice). Fields specific to the unknown type are subject to the [Preservation of unknown fields](#preservation-of-unknown-fields) rule below

### `type` recognized but wrong schema for that type

For example: type `image` but without the `src` field. The parser MUST preserve the bullet's state, and the outliner application SHOULD provide a visual fallback

### Metadata with invalid schema/Missing metadata

Treat this as an invalid file. The parser MUST NOT open the TreeWrite file

### `version` with a major greater than what the parser supports

As discussed in the [Versioning](#versioning) section, the parser MUST NOT open the file

### Duplicate `order` for the same `parent`

If two or more bullets have the same `order` and the same `parent`, then the tiebreaker MUST be the `id` (ULID is sortable). The bullet with the lexicographically smaller ULID MUST come first. The relative order of the siblings that are not involved in the conflict MUST be preserved

### Gaps or negative values in `order`

Gaps (for example 0, 2, 5) and negative values in `order` MUST be tolerated. Only the relative order among siblings is meaningful. A parser MAY renumber `order` sequentially from zero when writing

### Preservation of unknown fields

Any additional fields in JSON objects beyond those defined in this specification MUST NOT be discarded during a read/rewrite to avoid data loss
