# TreeWrite file format specification

## Goals of this document

This document describes a format, called **TreeWrite file format**, for storing bullet points in a `.jsonl` ([JSON Lines](https://jsonlines.org)) file

The purpose of this format is to propose an open standard by which [outliner applications](https://en.wikipedia.org/wiki/Outliner) store and process bullet points of various types (text, images, code, quotes, etc.) in a text file

The design goal is to have a transparent, human-readable format that is backward compatible with previous versions, so that notes written today can still be accessed by outliners 20 years from now. We prioritize simplicity and stability over abrupt changes

The TreeWrite file format was created specifically for the [TreeWrite text editor](https://treewrite.com), but since it's an open format, it can be used by any other tools. This document also serves as a guide for implementing a TreeWrite format parser. See the [official TreeWrite parser]()

It's an open format, meaning any outliner or developer can use this format for any purpose. The specification license is [CC0 1.0 Universal](https://github.com/treewrite/spec/blob/main/LICENSE)

To understand why we created a new specification instead of using an existing one (such as [OPML 2.0](https://opml.org/spec2.opml)), see [this section]()

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

| Field     | Description                                                                                                                            |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `version` | The current major version (ex. 1) of the specification that the file is following. See the [Versioning]() section for more information |

## Bullet point format

All lines in the file, except the first, represent bullet points. Each bullet point is a JSON object. All bullet points have fields in common, in addition to fields specific to each bullet point type. The following fields are common to all bullet points:

| Field        | Description                                                                                                                                                             |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`         | A [ULID](https://github.com/ulid/spec) to differentiate the bullet points                                                                                               |
| `updated_at` | Last bullet point update date. Represented by a number, the total milliseconds since the [Unix epoch](https://en.wikipedia.org/wiki/Unix_time)                          |
| `parent`     | The ULID of the parent bullet point (which allows for a tree structure). It must be null if it is not nested                                                            |
| `order`      | A sequential number, starting from zero, representing the bullet point index in the direct children of its parent. (ex. 2, meaning that it is the parent's third child) |
| `type`       | The type that the bullet point represents (ex. text, code, etc.). See the complete [list of types]()                                                                    |

## Versioning

The specification follows [semantic versioning](https://semver.org). The `version` field in the metadata contains the current major version of the file

If the version is larger than what some parser currently supports (ex. metadata with version 2 with parser implementing major version 1), then the file should not be opened by the parser, since the parser is trying to read a version that is not compatible with its implementation

For example, if in a major version 2 we rename a [bullet point type](), a parser following major version 1 will not know about this renaming and will incorrectly ignore the bullet point

However, if the parser implements a major version larger than the file's version, it should be able to open and edit the older major version. In other words, a parser that supports a major version X should support all major versions older than X

## Bullet point types

### `text`

The `text` type represents any single-paragraph text (without new lines). New lines are not supported as they can easily be represented by new bullets. It has the following fields:

| Field  | Description                                                        |
| ------ | ------------------------------------------------------------------ |
| `text` | A string without newlines (`\n`). Accepts [Inline markdown text]() |

### `quote`

The `quote` type represents a quote from an author. It has the following fields:

| Field    | Description                                                                                           |
| -------- | ----------------------------------------------------------------------------------------------------- |
| `text`   | The quotation itself. Accepts [Inline markdown text]()                                                |
| `author` | The name of the author who quoted this quote. Semantically, it can be any source (person, book, etc.) |

### `divider`

The `divider` type does not have any custom fields. It represents a line that visually divides the user interface

## Inline markdown text

## Handling inconsistencies

This section lists and details some inconsistency scenarios that could (but should not) eventually be found in a TreeWrite file, and how a parser should behave in each scenario

### `parent` points to an `id` that does not exist in the file

Treat it as `parent: null`, promoting the bullet to the root. Never discard the bullet (to avoid data loss) and never lock the parser

### Cycle in the tree (A is the parent of B, B is the parent of C, C is the parent of A)

It can be detected during tree loading. Upon finding the cycle, break it by obtaining the bullet with the highest `id` (ULID is sortable) and treat it as `parent: null`

### Duplicate `id`

Two or more bullets with the same `id`. The bullet on the last line of the file should be considered, ignoring the other repeated bullets. Do not discard the other bullets to avoid data loss

### Line with syntactically invalid/malformed JSON

Ignore the bullet line. Do not discard it to avoid loss of information

### `id` that is not a valid ULID

The entire bullet point is invalid. The parser should treat it the same way as a malformed JSON

### Required field missing

Invalid line, treated as malformed. Unlike an unknown field (which is preserved), a missing required field prevents the reconstruction of a minimally coherent bullet point, so the line cannot be promoted to a valid bullet point

### `type` recognized but wrong schema for that type

For example: type `image` but without the `src` field. The parser should neither ignore nor discard the bullet point. Render it with the missing fields using a default fallback value

### Metadata with invalid schema

Treat this as an invalid file. The parser should not open the TreeWrite file

### `version` with a major greater than what the parser supports

As discussed in the [Versioning]() section, the parser should not be able to open the file
