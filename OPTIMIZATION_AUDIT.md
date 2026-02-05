# Token Optimization Audit - Jonasbot

## Overview
Analyzing the Jonasbot codebase for token expenditure optimizations.

## Areas Under Review

### 1. **Model Selection Logic**
- **Files:** `src/auto-reply/reply/model-selection*.ts`, `src/auto-reply/model*.ts`
- **Status:** 🔍 REVIEWING
- **Potential Wins:**
  - Verify Haiku is consistently the default
  - Ensure fallbacks (Sonnet/Opus) only trigger for specific conditions
  - Add cost warnings for model escalation

### 2. **Prompt Caching Implementation**
- **Files:** `src/agents/`, `src/config/`
- **Status:** 🔍 REVIEWING
- **Known:** Already configured in `openclaw.cost-optimized.json5`
- **Potential Wins:**
  - Verify cache headers are set correctly on all agents
  - Ensure stable system prompts for cache warmth
  - Batch related operations to maximize cache hits

### 3. **Context Pruning & Memory**
- **Files:** `src/auto-reply/reply/`, `src/session/`
- **Status:** 🔍 REVIEWING
- **Current:** Cache TTL-based pruning (5m)
- **Potential Wins:**
  - Review default context window sizes
  - Ensure old messages are discarded after cache expires
  - Optimize session storage for large histories

### 4. **Heartbeat Implementation**
- **Files:** `src/heartbeat/`, `src/agents/`
- **Status:** 🔍 REVIEWING
- **Current:** 4m interval with Haiku model
- **Potential Wins:**
  - Verify heartbeat doesn't load full session history
  - Check that heartbeat uses only necessary context
  - Consider conditional heartbeats (skip if nothing to do)

### 5. **API Call Batching**
- **Files:** `extensions/`, `src/providers/`
- **Status:** 🔍 REVIEWING
- **Potential Wins:**
  - Batch weather/calendar checks together
  - Consolidate multiple tool calls into single requests
  - Reduce redundant API calls to external services

### 6. **Unused Dependencies / Dead Code**
- **Status:** 🔍 REVIEWING
- **Potential Wins:**
  - Remove unused model definitions
  - Clean up old provider integrations
  - Reduce bundle size for faster startup

## Next Steps
1. Deep-dive on each area
2. Create specific PRs with measurable impact
3. Add test coverage for optimizations
4. Document cost savings per optimization

## Budget Tracking
- **Daily:** $5 limit
- **Monthly:** $200 limit
- **Goal:** Reduce to $5-15/month

---
**Created:** 2026-02-05
**Auditor:** Jonasbot
