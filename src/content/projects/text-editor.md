---
title: Text Editor
description: A fully functional text editor built from scratch in raw JavaScript — zero frameworks, zero libraries. At its core is a gap buffer, a contiguous array split into two halves by a movable gap sitting at the cursor position.
tags: [JavaScript, Gap Buffer, Data Structures]
url: https://editor.gauravnegi.in
order: 1
---

A fully functional text editor built from scratch in raw JavaScript — zero frameworks, zero libraries.

## How It Works

At its core is a **gap buffer**: a contiguous array split into two halves by a movable gap sitting at the cursor position. Inserting or deleting a character near the cursor is O(1) because characters go directly into the gap without shifting the rest of the buffer. Moving the cursor shifts the gap by copying characters across it.

This is the same technique used by Emacs and the early VS Code text model.

## Key Features

- Pure JavaScript — no dependencies
- Gap buffer data structure for efficient editing
- Real-time cursor tracking
- Lightweight and fast
