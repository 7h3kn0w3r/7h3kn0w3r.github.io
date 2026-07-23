---
title: Windows Data Deduplication
description: A Forensic Investigation into Windows Data Deduplication [The Vanishing File]
date: 2026-07-22
category: Research
tags:
  - DFIR
cover_image: /uploads/cover-2-.png
draft: false
---
## How Data Deduplication Work

Before we learn how Windows Data Deduplication works, we first need to understand the problem it was designed to solve.\
Modern organizations store enormous amounts of data. File servers, virtual machines, backups, databases, and user documents can easily consume terabytes or even petabytes of storage.\
One common problem is that much of this data is duplicated.\
Imagine a company that stores the following files: 

> File A
>
> Windows_Server_Backup.vhdx
>
> File B
>
> Windows_Server_Backup_Copy.vhdx
>
> File C\
> Windows_Server_Backup_2026.vhdx

In a traditional file system, each file is stored independently, even if most or all of its content is identical to another file. This results in multiple copies of the same data being written to disk, wasting valuable storage space.

Data Deduplication takes a different approach. Instead of storing each file as a complete, independent copy, it breaks file data into smaller pieces called **chunks**. Each chunk is analyzed, and only unique chunks are stored in a shared storage area known as the **Chunk Store**. If another file contains a chunk that has already been stored, Windows simply creates a reference to the existing chunk instead of writing a duplicate copy.

![](/uploads/gemini_generated_image_xr2xq1xr2xq1xr2x.png)

### This occurs in the following four steps:

1. Scan the file system for files meeting the optimization policy.

   ![](/uploads/understanding-dedup-how-dedup-works-1.gif)
2. Break files into variable-size chunks.

   ![](/uploads/understanding-dedup-how-dedup-works-2.gif)
3. Identify unique chunks.

   ![](/uploads/understanding-dedup-how-dedup-works-3.gif)
4. Place chunks in the chunk store and optionally compress.

   ![](/uploads/understanding-dedup-how-dedup-works-4.gif)

### Where exactly do you cut?

#### Attempt 1: Fixed-size chunking

The simplest idea: cut every file into blocks of exactly, say, 64 KB.

Simple to implement. Fast to compute. But it has a fatal weakness, and it's the single most important concept in this entire chapter, so slow down and read this carefully.

Imagine `document_v1.docx` and `document_v2.docx` are identical except someone typed one extra sentence at the very beginning.

Because fixed-size chunking cuts at rigid byte offsets (0, 64K, 128K, 192K...), inserting even a single byte at the beginning shifts every single chunk boundary that comes after it. The result: `document_v2` shares *zero* chunks with `document_v1`, even though 99.9% of the actual content is identical.

This is called the **boundary-shift problem**, and it's the reason fixed-size chunking, while simple, performs poorly on real-world edited documents (contracts, source code, logs — anything that gets incrementally modified).

#### Attempt 2: Content-defined chunking (variable-size)

What if, instead of cutting at fixed byte positions, we cut at positions determined by the *content itself*? Then inserting a byte at the start would only affect the *one chunk* containing that insertion — every chunk after it stays byte-identical and gets recognized as a duplicate.

This is done using a **rolling hash** — most famously, the **Rabin fingerprint** algorithm (Rabin, 1981), which Windows' post-process engine and many other dedup systems rely on conceptually. 

For more details about Rabin fingerprint <https://moinakg.wordpress.com/tag/rabin-fingerprint/>

Here's the intuition, no math required yet:

1. Slide a fixed-size "window" (say, 48 bytes) across the file, byte by byte.
2. At every position, compute a fingerprint value from the bytes currently inside the window.
3. Whenever that fingerprint matches a specific pattern (e.g., its lowest N bits are all zero), declare "chunk boundary here."

Because the fingerprint depends only on the *local* bytes under the window, a change far away doesn't affect it. Insert a byte at the start of the file, and the cut points simply *shift along with the insertion* — everything after the edit still produces the exact same chunks as before.

This is *why* content-defined chunking is the standard for real-world dedup: it isolates edits to a small local region and keeps everything else deduplicable.

![](/uploads/gemini_generated_image_n4szddn4szddn4sz.png)

## Windows Deduplication Storage Architecture

The three core components of Windows Data Deduplication are:

1. **`$REPARSE_POINT`** – The starting point. It marks the file as deduplicated and provides the metadata required to locate its associated stream.
2. Stream File / Stream Map – The reconstruction blueprint. It contains the ordered list of chunk references that describes how the original file can be rebuilt.
3. **Chunk Store (Container Files)** – The physical storage location for the actual data. It stores unique chunks inside container (`.ccc`) files, allowing multiple files to share identical data without duplication.

### 1-**`REPARSE_POINT`** 

This is **not** an introduction to NTFS. Instead, the following sections provide only the concepts necessary to understand how Windows stores and reconstructs deduplicated files.

If you are already familiar with NTFS, you can safely skip this section.

For readers who would like a deeper understanding of NTFS internals, I highly recommend the following references:

* <https://internals-for-interns.com/posts/ntfs-filesystem/>
* <https://a1l4m.github.io/research/posts/MFT/index.html>
* <https://thrwt.ninja/NOTES/MFT.html>

Throughout this research, we only need to understand one NTFS attributes: `REPARSE_POINT`.

When a file is optimized by Windows Data Deduplication, Windows adds a `REPARSE_POINT` attribute to its MFT entry. This attribute indicates that the file is no longer stored as a traditional NTFS file and must be processed by the Deduplication filter driver.

The reparse point contains several pieces of metadata that are essential for locating the deduplicated data.

Among the most important fields are:

* **Original File Size** – The logical size of the file before deduplication. This is the size that applications see when accessing the file.
* **Chunk Store Identifier (GUID)** – A unique identifier that specifies which Chunk Store contains the file's deduplicated data.
* **Stream Header** – A unique identifier used to locate the corresponding stream metadata. The stream metadata describes how the original file can be reconstructed by referencing chunks stored in the Chunk Store.

The `REPARSE_POINT` attribute does **not** contain the actual file data. Instead, it provides the information required for Windows to locate the stream metadata, which ultimately leads to the chunks needed to reconstruct the original file.

#### Reverse Engineering the `$REPARSE_POINT`

![](/uploads/gemini_generated_image_d0i1ldd0i1ldd0i1.png)

> The Stream Header acts as a unique identifier that allows Windows to locate the correct stream metadata for the deduplicated file. This stream metadata contains the mapping between the file and the chunks stored in the Chunk Store.

### 2- The Stream Map File (`.stream` metadata)

After a file is optimized, Windows no longer stores the original file contents directly inside the file. Instead, it creates a **Stream File**, which contains the metadata required to reconstruct the original file.
