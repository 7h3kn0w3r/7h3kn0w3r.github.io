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
## 1- How Data Deduplication

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

**Where exactly do you cut?**

### Attempt 1: Fixed-size chunking

The simplest idea: cut every file into blocks of exactly, say, 64 KB.

```
File: [============================================]
       |--64KB--|--64KB--|--64KB--|--64KB--|--...
```

Simple to implement. Fast to compute. But it has a fatal weakness, and it's the single most important concept in this entire chapter, so slow down and read this carefully.

Imagine `document_v1.docx` and `document_v2.docx` are identical except someone typed one extra sentence at the very beginning.

```
document_v1: [AAAAAAAA][BBBBBBBB][CCCCCCCC][DDDDDDDD]
document_v2: [XAAAAAAA][ABBBBBBB][BCCCCCCC][CDDDDDDD]
                ^ one byte inserted at the start shifts EVERYTHING after it
```

Because fixed-size chunking cuts at rigid byte offsets (0, 64K, 128K, 192K...), inserting even a single byte at the beginning shifts every single chunk boundary that comes after it. The result: `document_v2` shares *zero* chunks with `document_v1`, even though 99.9% of the actual content is identical.

This is called the **boundary-shift problem**, and it's the reason fixed-size chunking, while simple, performs poorly on real-world edited documents (contracts, source code, logs — anything that gets incrementally modified).

### Attempt 2: Content-defined chunking (variable-size)

What if, instead of cutting at fixed byte positions, we cut at positions determined by the *content itself*? Then inserting a byte at the start would only affect the *one chunk* containing that insertion — every chunk after it stays byte-identical and gets recognized as a duplicate.

This is done using a **rolling hash** — most famously, the **Rabin fingerprint** algorithm (Rabin, 1981), which Windows' post-process engine and many other dedup systems rely on conceptually.

Here's the intuition, no math required yet:

1. Slide a fixed-size "window" (say, 48 bytes) across the file, byte by byte.
2. At every position, compute a fingerprint value from the bytes currently inside the window.
3. Whenever that fingerprint matches a specific pattern (e.g., its lowest N bits are all zero), declare "chunk boundary here."

```
 sliding window (48 bytes) -->
 [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]
   [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]
     [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]
                                     ^
                          fingerprint matches pattern
                          -> CUT HERE
```

Because the fingerprint depends only on the *local* bytes under the window, a change far away doesn't affect it. Insert a byte at the start of the file, and the cut points simply *shift along with the insertion* — everything after the edit still produces the exact same chunks as before.

```
document_v1: [ChunkA][ChunkB][ChunkC][ChunkD]
document_v2: [Chunk_Edited][ChunkB][ChunkC][ChunkD]
                              ^^^^^^^^^^^^^^^^^^^^ still identical! reused!
```

This is *why* content-defined chunking is the standard for real-world dedup: it isolates edits to a small local region and keeps everything else deduplicable.



- - -
