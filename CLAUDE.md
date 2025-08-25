# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is "mojimoji" (Minutes Maker) - a local audio transcription and meeting minutes formatting application. It processes long meeting recordings (e.g., city council meetings) and outputs formatted meeting minutes in Word format, prioritizing user-friendly UI/UX and confidential data processing.

## Architecture

**Technology Stack:**
- **Frontend**: Electron + React with Material Design 3 (MUI v5)
- **Backend**: Node.js within Electron
- **APIs**: OpenAI Whisper API (transcription) + GPT API (formatting)
- **Audio Processing**: ffmpeg for splitting audio into ~5-minute chunks
- **Output**: Word (.docx) format using `docx` npm library

**Data Flow:**
1. User drops audio file (WAV/MP3/M4A) → temporary storage
2. ffmpeg splits audio into ~5-minute chunks
3. Whisper API transcribes each chunk sequentially with progress updates
4. GPT API formats complete transcription to formal meeting minutes style
5. Preview updates in UI → user saves as .docx file

## Key Development Commands

Based on the todo.md milestone structure, the expected commands will be:

```bash
# Development
pnpm install          # Install dependencies
pnpm dev             # Start development mode
pnpm test            # Run unit tests (Vitest)
pnpm e2e             # Run E2E tests (Playwright)
pnpm build           # Build distributable packages

# Linting & Formatting
pnpm lint            # ESLint with strict TypeScript rules
pnpm format          # Prettier formatting
```

## Code Organization & Development Approach

**Test-Driven Development (TDD)**:
- Each milestone requires passing verification commands
- Unit tests for: ffmpeg splitting, Whisper API calls (mocked), GPT formatting, docx generation
- E2E tests for full workflow scenarios
- Mock APIs during development, switch to real APIs in final stages

**State Management**:
- Zustand for React state management
- Session state auto-saved every 30 seconds to `userData/sessions/<hash>.json`
- Crash recovery by detecting and resuming from last saved session

**Security Requirements**:
- API keys stored securely using `keytar` (never in plain text)
- OpenAI data not used for training (documented policy compliance)
- No PII in logs (configurable debug levels)
- Secure IPC with `contextIsolation: true`

## Core Components

**Audio Processing**:
- `ffmpeg-static` + `child_process.spawn` for chunking
- Progress events per chunk (index/start/end/path)
- Error handling: network retry with exponential backoff, rate limit queuing

**Transcription Pipeline**:
- Queue-based processing (`p-queue`) with configurable concurrency
- Retry logic: 3 attempts max with jitter, handles 429/Retry-After
- Incremental UI updates as chunks complete

**Formatting System**:
- Local rules as fallback + GPT API for polishing
- Segment-based processing for long texts with overlap handling  
- Rules: Remove fillers/interjections, fix repetitions, formal "desu/masu" tone, split long sentences

**File Output**:
- Naming convention: `<original>_minutes.docx`
- Duplicate handling with `_<n>` numbering
- Save dialog integration

## Critical Implementation Notes

**Performance Considerations**:
- Non-blocking UI during 2+ hour audio processing
- Virtualized preview for large text (react-window for 100k+ characters)
- Configurable chunk size and concurrency settings

**Error Recovery**:
- Graceful degradation: partial exports when processing fails
- User-friendly error messages mapped from technical errors
- Resume capability from interruption points

**Platform Support**:
- Primary: Windows (electron-builder with nsis)
- Secondary: macOS (dmg packaging)
- Cross-platform file handling and temp directory management

## Development Workflow

Follow the milestone-based approach in `todo.md`:
1. Always maintain buildable/testable state
2. Complete milestones sequentially (M1→M16)
3. Run verification commands after each milestone
4. Use feature branches with descriptive commit messages
5. Mock external APIs during development, real APIs only in final testing

The project emphasizes incremental development with continuous validation to ensure a production-ready desktop application for sensitive meeting transcription workflows.