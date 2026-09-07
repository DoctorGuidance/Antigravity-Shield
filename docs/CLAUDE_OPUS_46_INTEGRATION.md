# Claude Opus 4.6 Thinking Integration

## Overview
This document describes the integration of Claude Opus 4.6 Thinking model into the Antigravity Proxy.

## Changes Made

### Backend (Rust)

#### model_mapping.rs
- Added direct mapping for `claude-opus-4-6-thinking`
- Added alias mappings: `claude-opus-4-6`, `claude-opus-4-6-20260201`
- Updated `normalize_to_standard_id()` to include the new model

#### request.rs
- Updated `should_enable_thinking_by_default()` to auto-enable thinking for Opus 4.6 models

### Frontend (TypeScript/React)

#### modelConfig.ts
- Added `claude-opus-4-6-thinking` entry with `protectedKey: 'claude-opus'`

#### useProxyModels.tsx
- Added model to the models array with group `Claude 4.6`

## Usage

### API Request Example
```json
{
  "model": "claude-opus-4-6-thinking",
  "messages": [...]
}
```

### Supported Aliases
- `claude-opus-4-6-thinking` (canonical)
- `claude-opus-4-6`
- `claude-opus-4-6-20260201`

## Notes
- Thinking mode is auto-enabled for Opus 4.6 models
- Quota protection uses the shared `claude-opus` key
- No localization changes required (reuses existing keys)
