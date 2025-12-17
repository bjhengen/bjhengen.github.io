---
title: "Rich Text Editing with Image Paste and OneNote Integration"
date: 2025-11-04
draft: true
tags: ["flutter", "quill", "onenote", "clipboard", "rag", "image-processing"]
categories: ["Development", "Flutter", "Knowledge Management"]
description: "Adding clipboard image paste and OneNote integration to a Flutter knowledge management app using Quill rich text editor."
---

## Overview

Building a knowledge management application requires powerful note-taking capabilities. Over the past week, I've been enhancing my Flutter-based MGR App with advanced rich text editing features, including clipboard image paste support and complete OneNote integration. Here's the journey.

## The Challenge: Rich Text Editing

Modern note-taking applications need more than basic text formatting. Users expect to:
- Paste images directly from clipboard
- Format text with rich styling
- Embed various content types
- Edit notes seamlessly across devices

I chose **Flutter Quill v11** as the rich text editor, but quickly discovered that the v11 API had significant breaking changes from previous versions.

## Implementing Clipboard Image Paste

### The Problem

The initial Quill implementation worked well for text, but had no way to paste images from the clipboard. The image insert button required file selection, which disrupts the writing flow.

### The Solution

I implemented clipboard paste support using three key components:

#### 1. Clipboard Detection with super_clipboard

```dart
Future<void> _handlePaste() async {
  final clipboard = SystemClipboard.instance;
  if (clipboard == null) return;

  final reader = await clipboard.read();

  // Try PNG format
  if (reader.canProvide(Formats.png)) {
    reader.getFile(Formats.png, (file) async {
      final bytes = await file.readAll();
      _insertImageFromBytes(bytes, 'png');
    });
    return;
  }
  // Similar for JPEG and GIF...
}
```

#### 2. Keyboard Event Handling

```dart
RawKeyboardListener(
  focusNode: FocusNode(),
  onKey: (RawKeyEvent event) {
    if (event is RawKeyDownEvent) {
      final isMetaPressed = event.isMetaPressed || event.isControlPressed;
      if (isMetaPressed && event.logicalKey == LogicalKeyboardKey.keyV) {
        _handlePaste();
      }
    }
  },
  child: QuillEditor(/* ... */),
)
```

#### 3. Base64 Image Embedding

```dart
void _insertImageFromBytes(Uint8List bytes, String format) {
  final base64String = base64Encode(bytes);
  final imageUrl = 'data:image/$format;base64,$base64String';

  _quillController.replaceText(
    index,
    length,
    BlockEmbed.image(imageUrl),
    null,
  );
}
```

### API Challenges

The `super_clipboard` package API was tricky to work with. The documentation suggested methods that didn't exist in v0.9.1. After multiple iterations, I discovered the correct pattern:

- Use `reader.canProvide()` to check for format availability
- Use `reader.getFile()` with a callback (not `readFile()` or `readValue()`)
- Call `readAll()` on the file object to get bytes

## OneNote Integration: 5,000+ Documents

In parallel with the UI enhancements, I completed a massive data migration project importing over 5,000 OneNote documents into the RAG system.

### The Process

1. **API Authentication**: Implemented device code flow auth for Microsoft Graph API
2. **Recursive Extraction**: Built scripts to traverse nested section groups
3. **Content Processing**: Extracted HTML content and converted to searchable text
4. **Fiscal Year Organization**: Standardized document dates (FY14-FY26)
5. **Vectorization**: Created 23,000+ embeddings using Oracle's in-database CLIP model
6. **Image Handling**: Preserved image references and folder structure

### Key Challenges

**Rate Limiting**: Microsoft Graph API has strict throttling. Solution: Built resumable scripts with checkpoint tracking and exponential backoff.

**Large Documents**: Some OneNote pages exceeded 70KB. Solution: Enhanced CLOB reading with proper connection management.

**Date Extraction**: OneNote metadata was inconsistent. Solution: Fiscal year pattern matching with June 1st standardization.

## Backend Services Management

With multiple backend services (Node.js API on port 3000, Python vision service on port 3002), I needed better dev tools.

### Unified Services Control Script

Created `services-control.sh` for managing both services:

```bash
#!/bin/bash
# Usage: ./services-control.sh {start|stop|restart|status|logs} [backend|vision|all]

# Start both services
./services-control.sh start

# Check status
./services-control.sh status

# View logs
./services-control.sh logs
```

#### Features

- **Smart Process Management**: Detects services by PID and port
- **Orphan Cleanup**: Automatically kills abandoned processes
- **Color-Coded Output**: Easy-to-read status information
- **Graceful Shutdown**: SIGTERM first, SIGKILL as fallback
- **Live Log Viewing**: Tail both service logs simultaneously

Example status output:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 MGR App Services Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔹 Node.js Backend (port 3000):
   ✅ RUNNING
   PID: 84208
   Endpoint: http://localhost:3000

🔹 Python Vision Service (port 3002):
   ✅ RUNNING
   PID: 91294
   Endpoint: http://localhost:3002
```

## Technical Stack Updates

### Flutter Dependencies

```yaml
dependencies:
  flutter_quill: ^11.5.0
  flutter_quill_extensions: ^11.0.0
  super_clipboard: ^0.9.1
  html_editor_enhanced: ^2.7.1
  file_picker: ^10.3.3
```

### Backend Components

- **Node.js**: Express API server with Oracle database integration
- **Python**: Vision service for image analysis (Grok Vision API)
- **Oracle Database**: 23ai with vector search and in-database embeddings
- **Microsoft Graph**: OneNote API integration with OAuth device flow

## Results and Impact

### Quill Editor Enhancements

✅ **Clipboard paste working** for PNG, JPEG, and GIF images
✅ **Base64 embedding** - no separate file storage needed
✅ **Cross-platform support** - Cmd+V (macOS) and Ctrl+V (Windows/Linux)
✅ **Flutter Quill v11** - fully migrated to latest API

### OneNote Integration

✅ **5,178 documents** fully vectorized and searchable
✅ **23,000+ embeddings** using in-database CLIP model
✅ **Fiscal year organization** with standardized dates
✅ **Complete folder structure** preserved from OneNote

### Developer Experience

✅ **One command** to start/stop all backend services
✅ **Clear status visibility** for all running processes
✅ **Centralized logging** for easier debugging
✅ **Automated cleanup** of orphaned processes

## Lessons Learned

### 1. Read the Source Code

Documentation for `super_clipboard` was misleading. The actual working API was different from examples. Reading the source code and examining type definitions saved hours of debugging.

### 2. Incremental Testing is Critical

Each Quill API change required a full `flutter clean && flutter run` cycle. Testing small changes incrementally prevented cascading errors.

### 3. Rate Limiting Strategies Matter

For large-scale API integrations like OneNote, proper rate limiting with checkpoint resume is essential. Lost several hours of progress early on due to incomplete error handling.

### 4. Base64 vs External Storage Trade-offs

Embedding images as base64 data URLs is convenient but increases document size. For production, consider:
- Image compression before encoding
- Size limits for inline images
- External storage for large images
- Lazy loading for image-heavy documents

## What's Next

### Planned Enhancements

- **Image Compression**: Automatically resize/compress pasted images
- **External Image Storage**: S3/Oracle Object Storage for large images
- **OneNote Sync**: Bi-directional sync with OneNote API
- **Rich Text Search**: Full-text search across formatted content
- **Collaborative Editing**: Real-time multi-user editing

### Performance Optimizations

- Lazy loading for large documents
- Virtual scrolling for long note lists
- Background vectorization for new notes
- Caching layer for frequently accessed content

## Conclusion

Building a modern knowledge management system requires attention to user experience details. Clipboard image paste seems like a small feature, but it dramatically improves the note-taking workflow. Combined with the massive OneNote integration, the app now provides a powerful foundation for organizational knowledge capture and retrieval.

The combination of Flutter's cross-platform capabilities, Quill's rich text editing, Oracle's vector search, and Microsoft's OneNote API creates a comprehensive knowledge management solution that scales from individual notes to enterprise-wide knowledge bases.

## Resources

- [Flutter Quill Documentation](https://github.com/singerdmx/flutter-quill)
- [super_clipboard Package](https://pub.dev/packages/super_clipboard)
- [Microsoft Graph OneNote API](https://learn.microsoft.com/en-us/graph/api/resources/onenote)
- [Oracle Database 23ai Vector Search](https://docs.oracle.com/en/database/oracle/oracle-database/23/vecse/)

---

*This work was done with assistance from Claude Code, an AI-powered development tool that helps with code generation, debugging, and documentation.*

