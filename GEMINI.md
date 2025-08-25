# GEMINI.md

This file provides guidance to Gemini when working with code in this repository.

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
