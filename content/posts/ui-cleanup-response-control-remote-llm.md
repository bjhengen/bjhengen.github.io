---
title: "UI Refinement, Response Control, and Remote GPU Integration"
date: 2025-11-05
draft: true
tags: ["flutter", "ui-ux", "llm", "ollama", "ssh-tunnel", "gpu", "knowledge-management"]
categories: ["Development", "Flutter", "AI Integration"]
series: ["SLM Research"]
---

## Overview

Today's development session focused on three key areas: cleaning up the user interface, implementing response length control for AI interactions, and successfully integrating a remote GPU-powered LLM via SSH tunnel. These improvements enhance both usability and flexibility of the knowledge management application.

## UI Cleanup: Simplifying Note Creation

### The Problem

The Notes screen had accumulated multiple note creation options over time:
- Simple Paste (text only)
- Clipboard Paste (Screenshots!)
- Enhanced Paste (With File Picker)
- Note with Screenshots
- Rich Text Note

With the Quill rich text editor now fully functional (including clipboard image paste), most of these options were redundant.

### The Solution

Streamlined to a single, powerful option:

```dart
floatingActionButton: FloatingActionButton(
  heroTag: 'notes_upload_fab',
  onPressed: _showRichTextNoteCreation,
  tooltip: 'Add Note',
  child: const Icon(Icons.note_add),
),
```

**Changes Made:**
- Removed 4 obsolete menu items
- Updated floating action button to call Rich Text Note creation
- Cleaned up unused methods and imports
- Simplified the PopupMenuButton to show only essential options

### Font Contrast Fix

Updated the note creation screen's AppBar for better accessibility:

```dart
appBar: AppBar(
  title: Text(
    _isEditMode ? 'Edit Note' : 'Create Note',
    style: const TextStyle(color: Colors.white),
  ),
  backgroundColor: Colors.deepPurple,
  foregroundColor: Colors.white,
  iconTheme: const IconThemeData(color: Colors.white),
),
```

**Result:** White text on dark purple background provides much better contrast and readability.

## Response Length Control

### The Challenge

Different use cases require different response lengths:
- Quick facts → Brief responses
- General questions → Standard responses
- Deep analysis → Detailed or verbose responses

Users needed control over how much detail the AI provides.

### Implementation

#### Frontend: Segmented Button Control

Added a clean UI control in the chat dialog:

```dart
SegmentedButton<String>(
  segments: [
    ButtonSegment(
      value: 'brief',
      label: Text('Brief'),
      icon: Icon(Icons.short_text)
    ),
    ButtonSegment(
      value: 'standard',
      label: Text('Standard'),
      icon: Icon(Icons.subject)
    ),
    ButtonSegment(
      value: 'detailed',
      label: Text('Detailed'),
      icon: Icon(Icons.description)
    ),
    ButtonSegment(
      value: 'verbose',
      label: Text('Verbose'),
      icon: Icon(Icons.article)
    ),
  ],
  selected: {_responseLength},
  onSelectionChanged: (newSelection) {
    setState(() => _responseLength = newSelection.first);
  },
)
```

#### Backend: Prompt Engineering

The backend maps each length setting to specific instructions:

```javascript
const lengthInstructions = {
  brief: 'Keep your answer very concise - 1-2 sentences maximum. Focus only on the key facts.',
  standard: 'Provide a clear, focused answer. Include key details but remain concise.',
  detailed: 'Provide a comprehensive answer with relevant details, examples, and context.',
  verbose: 'Provide an exhaustive, thorough answer. Include all relevant information, context, examples, and supporting details.'
};

const systemPrompt = `You are a helpful AI assistant analyzing documents and notes.

Context documents retrieved from the database:
${contextDocs}

${lengthInstructions[responseLength]}

Provide accurate, helpful answers based on the context provided.`;
```

### Data Flow

```
User selects length → Flutter sends parameter → Backend injects instruction → LLM responds accordingly
```

## Remote LLM Integration: The 5090 GPU Challenge

### The Problem

The application has a toggle for using a remote NVIDIA RTX 5090 GPU (slmbeast server) for LLM inference. However, the option was showing as grayed out/offline because:

1. **Ollama binding issue**: The Ollama service on slmbeast was only listening on `127.0.0.1:11434` (localhost), refusing external connections
2. **No status checking**: The Flutter app had no way to detect LLM availability
3. **Direct connection attempt**: The backend was trying to connect directly to `slmbeast:11434`

### The Solution: SSH Tunnel + Status Endpoint

#### Step 1: SSH Tunnel

Created an SSH tunnel to forward the local port to the remote Ollama service:

```bash
ssh -f -N -L 11434:localhost:11434 bhengen@slmbeast
```

**What this does:**
- `-f`: Runs in background
- `-N`: No remote commands (just port forwarding)
- `-L 11434:localhost:11434`: Forward local 11434 to remote localhost:11434

This allows the backend to access Ollama on `localhost:11434` which gets tunneled to slmbeast.

#### Step 2: LLM Status Endpoint

Created a new endpoint to check all LLM providers:

```javascript
router.get('/llm-status', authenticateToken, async (req, res) => {
  const ociGenAI = require('../services/genai');
  const localLLM = require('../services/localLLM');
  const remoteLLM = require('../services/remoteLLM');
  const visionLLM = require('../services/visionLLM');

  // Check all providers in parallel
  const [ociAvailable, localAvailable, remoteAvailable, visionAvailable] =
    await Promise.all([
      ociGenAI.isAvailable().catch(() => false),
      localLLM.isAvailable().catch(() => false),
      remoteLLM.isAvailable().catch(() => false),
      visionLLM.isAvailable().catch(() => false)
    ]);

  res.json({
    oci: { available: ociAvailable, name: 'OCI GenAI' },
    local: { available: localAvailable, name: 'M2 Mac' },
    remote: { available: remoteAvailable, name: '5090 GPU' },
    vision: { available: visionAvailable, name: 'Vision LLM' }
  });
});
```

#### Step 3: Configuration Update

Updated `.env` to use the tunnel:

```env
# Remote LLM Configuration
# SSH tunnel: ssh -f -N -L 11434:localhost:11434 bhengen@slmbeast
REMOTE_LLM_URL=http://localhost:11434
REMOTE_LLM_MODEL=qwen3-vl-128k:latest
```

### Verification

Testing the setup:

```bash
# Check tunnel is running
$ ps aux | grep "ssh.*11434"
BHENGEN   70884   ssh -f -N -L 11434:localhost:11434 bhengen@slmbeast

# Test Ollama API
$ curl http://localhost:11434/api/tags
{
  "models": [
    {
      "name": "qwen3-vl-128k:latest",
      "size": 6140415896,
      "parameter_size": "8.8B"
    }
  ]
}
```

## Architecture Diagram

```
┌─────────────────┐
│  Flutter App    │
│                 │
│  ┌───────────┐  │
│  │ UI Toggle │  │  [OCI] [Vision] [M2] [5090]
│  └─────┬─────┘  │         ↑
└────────┼────────┘         │
         │                  │
    checkLLMStatus()         │
         │                  │
         ▼                  │
┌─────────────────┐         │
│  Node Backend   │         │
│  Port 3000      │         │
│                 │         │
│  GET /llm-status├─────────┘
│                 │
│  Remote LLM:    │
│  localhost:11434├────┐
└─────────────────┘    │
                       │ SSH Tunnel
                       │ (Port Forward)
                       ▼
              ┌─────────────────┐
              │   slmbeast      │
              │   (5090 GPU)    │
              │                 │
              │  Ollama:11434   │
              │  qwen3-vl-128k  │
              └─────────────────┘
```

## Results

### UI Improvements
✅ Single, intuitive note creation option
✅ Cleaner menu with only essential functions
✅ Better font contrast for accessibility
✅ Removed 4 obsolete menu items

### Response Control
✅ 4 length options (Brief, Standard, Detailed, Verbose)
✅ Seamless frontend-to-backend integration
✅ Prompt engineering for consistent behavior
✅ User control over response verbosity

### Remote LLM Integration
✅ 5090 GPU option now active
✅ SSH tunnel successfully forwarding Ollama API
✅ LLM status detection working
✅ 4 LLM providers available (OCI, Vision, M2, 5090)

## Technical Stack

**Frontend:**
- Flutter 3.9+ with macOS desktop support
- Segmented buttons for clean UI controls
- Provider pattern for state management

**Backend:**
- Node.js Express API
- Ollama integration via REST API
- Multiple LLM provider support

**Infrastructure:**
- SSH tunnel for secure remote access
- NVIDIA RTX 5090 GPU (32GB VRAM)
- Qwen3-VL 128K multimodal model
- 128K token context window

## Lessons Learned

### 1. SSH Tunnels for Localhost-Only Services

When a remote service only binds to localhost, SSH port forwarding is an elegant solution that:
- Maintains security (no exposed ports)
- Works through firewalls
- Requires no server configuration changes
- Easy to set up and tear down

### 2. Status Endpoints Are Essential

For any external service integration, implement status checking:
- Prevents user confusion (grayed out options)
- Enables graceful degradation
- Allows dynamic feature availability
- Parallel checks minimize latency

### 3. UI Simplification

Less is often more:
- Remove options as capabilities consolidate
- One powerful feature beats multiple limited ones
- Regular UI audits prevent feature creep

### 4. Prompt Engineering for Control

Response length control through system prompts is:
- Simple to implement
- Highly effective
- No model fine-tuning required
- Works across different LLM providers

## Making the SSH Tunnel Persistent

The tunnel will stay active until system reboot. To make it persistent:

### Option 1: Shell Alias

Add to `~/.zshrc` or `~/.bashrc`:

```bash
alias tunnel-slm='ssh -f -N -L 11434:localhost:11434 bhengen@slmbeast'
```

### Option 2: LaunchAgent (macOS)

Create `~/Library/LaunchAgents/com.user.ollama-tunnel.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.ollama-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>-L</string>
        <string>11434:localhost:11434</string>
        <string>bhengen@slmbeast</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Load with:
```bash
launchctl load ~/Library/LaunchAgents/com.user.ollama-tunnel.plist
```

## What's Next

### Planned Features

1. **LangChain Integration for Spreadsheets**
   - Oracle Database 23ai has native LangChain support
   - Enhanced spreadsheet analysis capabilities
   - Leverage existing RAG context

2. **Image Optimization**
   - Compress pasted images automatically
   - External storage for large images (S3/Oracle Object Storage)
   - Lazy loading for image-heavy documents

3. **Collaborative Features**
   - Real-time multi-user editing
   - Shared folders and permissions
   - Activity feeds

## Conclusion

This session delivered meaningful improvements across three key areas. The UI is cleaner and more intuitive, users have fine-grained control over AI response length, and the remote GPU integration is now fully functional via SSH tunnel.

The combination of thoughtful UI design, practical prompt engineering, and clever networking solutions demonstrates that effective software development often comes down to understanding both user needs and infrastructure capabilities.

The application now provides a polished, flexible platform for knowledge management with access to multiple LLM providers including a powerful remote GPU option.

## Resources

- [Ollama Documentation](https://github.com/ollama/ollama)
- [SSH Port Forwarding Guide](https://www.ssh.com/academy/ssh/tunneling/example)
- [Flutter Segmented Button](https://api.flutter.dev/flutter/material/SegmentedButton-class.html)
- [Qwen3-VL Model](https://huggingface.co/Qwen/Qwen3-VL)

---

*This work was done with assistance from Claude Code, an AI-powered development tool that helps with code generation, debugging, and documentation.*
