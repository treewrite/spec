# TreeWrite file format specification

## Goals of this document

This document describes a format, called **TreeWrite file format**, for storing bullet points in a `.jsonl` ([JSON Lines](https://jsonlines.org)) file

The purpose of this format is to propose an open standard by which [outliner applications](https://en.wikipedia.org/wiki/Outliner) store and process bullet points of various types (text, images, code, quotes, etc.) in a text file

The design goal is to have a transparent, human-readable format that is backward compatible with previous versions, so that notes written today can still be accessed by outliners 20 years from now. We prioritize simplicity and stability over abrupt changes

It's an open format, meaning any outliner or developer can use this format for any purpose. The specification license is [CC0 1.0 Universal](https://github.com/treewrite/spec/blob/main/LICENSE)

The TreeWrite format was created specifically for the [TreeWrite text editor](https://treewrite.com), but since it's an open format, it can be used by any other tools

> To understand why we created a new specification instead of using an existing one (such as [OPML 2.0](https://opml.org/spec2.opml)), see [this section]()

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

1. **Text file:** The `.json` file is a text file, not a binary one, which makes it more accessible for manual editing and integration with tools like [Git](https://git-scm.com)
1. **File diffs:** changing a bullet point is equivalent to changing a single line of the file, which makes it easier to view file diffs in tools like Git
1. **Merge by line:** Text merge tools operate on lines. Two devices editing different bullet points never produce conflicts. A conflict only exists when editing the same bullet point, and the resolution remains local to that line
1. **Isolated corruption:** A poorly formatted line (interrupted writing, incorrect manual editing) invalidates a single bullet point, not all of them
1. **Centralization:** All bullet points are stored in a single file, instead of being scattered across multiple files
1. **Predictable scalability:** The cost of any operation (write, read) is linear to the number and size of bullet points, with no operations affected by the depth of the tree (as would be the case in nested JSON recursion or a Markdown list)
