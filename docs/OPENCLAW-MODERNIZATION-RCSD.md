# OpenClaw Modernization - RCSD Analysis

**Date**: 2026-02-03
**Epic**: Platform Modernization
**Status**: Research Complete - Awaiting Consensus

---

## Executive Summary

This document presents comprehensive research into modernizing OpenClaw's authentication, secrets management, and onboarding experience. The analysis identifies critical friction points and proposes solutions based on industry best practices.

### Key Findings

| Area | Current State | Recommended Solution |
|------|---------------|---------------------|
| **Authentication** | Device-only, gateway token | Better-Auth + API Keys |
| **Secrets Management** | Env vars, plaintext files | Infisical + dotenvx |
| **Onboarding UX** | CLI wizard, manual steps | Streamlined flow + Web UI option |
| **User Management** | None (device-centric) | User accounts + organizations |

---

## Research Phase (R)

### 1. Current Authentication Architecture

**Architecture Type**: Two-layer, device-centric

```
Layer 1: Gateway Token (Shared Secret)
├── Single token for ALL clients
├── Manual rotation breaks everything
├── No per-client differentiation
└── Stored in config or OPENCLAW_GATEWAY_TOKEN env

Layer 2: Device Identity (Ed25519 Crypto)
├── Per-device keypairs
├── SHA256 fingerprint = deviceId
├── Pairing codes for approval
└── Stored in ~/.openclaw/state/devices/paired.json
```

**Critical Pain Points Identified**:

1. **No User Concept** - System is device-only, can't identify "which user"
2. **Token is Shared** - Same token for CLI, mobile, web UI
3. **Manual Pairing** - Each device needs approval at gateway
4. **No Rotation** - Changing token breaks all clients
5. **Plaintext Storage** - Tokens in JSON files, not encrypted
6. **Scope System Unused** - `["operator.admin"]` exists but not enforced

### 2. Better-Auth Feasibility Assessment

**Verdict: HIGHLY FEASIBLE**

| Criterion | Rating | Notes |
|-----------|--------|-------|
| SQLite Support | Excellent | `better-sqlite3` recommended, fully production-ready |
| API Key Plugin | Excellent | Hashed storage, expiration, revocation |
| TypeScript | Excellent | First-class support, full type inference |
| Sessions | Excellent | Database, stateless JWT, or hybrid |
| Multi-User | Excellent | Organizations + teams plugins available |
| Migration | Medium | Requires schema + credential migration |

**Key Features for OpenClaw**:
- API key management with automatic hashing
- Organization/team scoping
- Session management (cookie or header-based)
- OAuth provider support (can act as auth server)
- SQLite fully supported (same as OpenClaw)

### 3. Secrets Management Comparison

| Solution | Self-Hosted | Node.js SDK | RBAC | Audit | Free/OSS | Recommendation |
|----------|-------------|-------------|------|-------|----------|----------------|
| **Doppler** | No | Yes | Yes | Yes | Limited | Not recommended (cloud-only) |
| **Infisical** | Yes | Yes | Yes | Yes | Yes (MIT) | **PRIMARY** |
| **Vault** | Yes | Community | Yes | Yes | Yes | Too complex |
| **SOPS** | N/A | No | No | No | Yes | GitOps only |
| **dotenvx** | N/A | Yes | No | No | Yes | **SECONDARY** |
| **1Password** | Partial | Yes | Yes | Yes | No | Already integrated |

**Recommendation**: **Infisical + dotenvx hybrid**
- Infisical for team/production (RBAC, audit, secret syncing)
- dotenvx for development/local (zero operational overhead)

### 4. Onboarding UX Pain Points

**Critical Issues Identified**:

| Issue | Severity | Impact |
|-------|----------|--------|
| No auto-trigger after install | Medium | Users don't know what to do next |
| Security warning blocks access | High | Friction for trial users |
| Gateway token not displayed | High | Users can't use generated token |
| 15+ auth options with no guidance | High | Decision paralysis |
| No "test connection" option | Medium | Users discover broken config later |
| Wizard guide referenced but missing | Critical | Broken documentation links |
| Cryptic error messages | High | Users can't troubleshoot |

**Current Flow** (10+ manual steps):
```
Install → Manual: openclaw onboard → Security warning →
Flow choice → Auth selection (15+ options) → Gateway config →
Channel setup (overwhelming) → Skills → Service install (may fail) →
Manual: openclaw gateway run → Manual: openclaw channels login →
Manual: openclaw pairing approve → First message works
```

**Ideal Flow**:
```
Install → Auto-detect → Minimal wizard → First message
```

### 5. Industry Auth Standards (2025)

**Patterns from OpenAI, Anthropic, Vercel, LangSmith**:

| Feature | Industry Standard | OpenClaw Gap |
|---------|------------------|--------------|
| API Keys | Long-lived bearer tokens | Mixed with OAuth |
| Org Scoping | Headers (`X-Organization-Id`) | Not implemented |
| Key Rotation | Automatic + manual | Manual only |
| Rate Limiting | Tiered with headers | Not implemented |
| Service Accounts | Separate from users | All profiles equal |
| Key Metadata | Rich (created, last_used, scopes) | Minimal |
| Audit Logs | Standard | None |

---

## Consensus Phase (C)

### Proposed Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    NEW AUTH ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │   Better-Auth    │    │    Infisical     │                   │
│  │  (User Auth)     │    │  (Secrets Mgmt)  │                   │
│  │                  │    │                  │                   │
│  │  - Email/Pass    │    │  - API Keys      │                   │
│  │  - OAuth         │    │  - LLM Tokens    │                   │
│  │  - API Keys      │    │  - Channel Creds │                   │
│  │  - Sessions      │    │  - Env Variables │                   │
│  │  - Orgs/Teams    │    │  - Audit Logs    │                   │
│  └────────┬─────────┘    └────────┬─────────┘                   │
│           │                       │                              │
│           └───────────┬───────────┘                              │
│                       │                                          │
│           ┌───────────▼───────────┐                              │
│           │   OpenClaw Gateway    │                              │
│           │                       │                              │
│           │  - Validates sessions │                              │
│           │  - Checks API keys    │                              │
│           │  - Injects secrets    │                              │
│           │  - Enforces scopes    │                              │
│           └───────────────────────┘                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Decisions Required

1. **User Accounts**: Add user concept or stay device-only?
   - **Recommendation**: Add users, map devices to users

2. **API Key Model**: Replace gateway token with Better-Auth API keys?
   - **Recommendation**: Yes, with scoping and expiration

3. **Secrets Backend**: Infisical, dotenvx, or hybrid?
   - **Recommendation**: Hybrid (Infisical prod, dotenvx dev)

4. **Onboarding**: CLI-first or add web-based wizard?
   - **Recommendation**: Both (web wizard for new users, CLI for power users)

5. **Migration Path**: Breaking change or backward compatible?
   - **Recommendation**: Backward compatible with legacy token support during transition

---

## Specification Phase (S)

### Authentication Specification

#### 1. User Model (New)

```typescript
interface User {
  id: string;           // UUID
  email: string;        // Primary identifier
  name?: string;
  created_at: Date;
  organizations: Organization[];
}

interface Organization {
  id: string;
  name: string;
  members: Member[];
  api_keys: ApiKey[];
}

interface ApiKey {
  id: string;           // key_xxx format
  name: string;
  key_hash: string;     // Hashed, never stored raw
  scopes: string[];     // e.g., ["agents:read", "agents:execute"]
  created_at: Date;
  created_by: string;   // User ID
  last_used_at?: Date;
  expires_at?: Date;
  active: boolean;
}
```

#### 2. Authentication Flows

**Flow A: Web UI Login**
```
Browser → Better-Auth login page → Session cookie → Access granted
```

**Flow B: CLI Login**
```
CLI → Better-Auth device flow → Access token → Stored locally
```

**Flow C: Programmatic API**
```
Request + Authorization: Bearer <api_key> → Validate key → Access granted
```

#### 3. API Key Scopes

| Scope | Description |
|-------|-------------|
| `agents:read` | View agent configs and status |
| `agents:execute` | Run agents, send messages |
| `agents:admin` | Create/delete agents |
| `channels:read` | View channel configs |
| `channels:admin` | Configure channels |
| `gateway:admin` | Gateway management |
| `secrets:read` | Read secrets (via Infisical) |
| `secrets:write` | Modify secrets |

### Secrets Management Specification

#### 1. Integration Points

```typescript
// At gateway startup
const secrets = await loadSecrets({
  infisical: process.env.INFISICAL_ENABLED === 'true'
    ? { projectId: '...', environment: process.env.NODE_ENV }
    : null,
  dotenvx: { path: '.env' },
  fallback: process.env  // Plain env vars as last resort
});

// Inject into process.env
Object.assign(process.env, secrets);
```

#### 2. Secret Categories

| Category | Examples | Storage |
|----------|----------|---------|
| **LLM Credentials** | ANTHROPIC_API_KEY, OPENAI_API_KEY | Infisical (encrypted) |
| **Channel Tokens** | TELEGRAM_BOT_TOKEN, DISCORD_TOKEN | Infisical (encrypted) |
| **Gateway Config** | OPENCLAW_GATEWAY_TOKEN | Infisical or dotenvx |
| **Feature Flags** | ENABLE_MEMORY_SEARCH | dotenvx or env |

### Onboarding Specification

#### 1. New Quick-Start Flow

```
Step 1: Install
  └── npm install -g openclaw (or curl installer)

Step 2: Auto-Launch Setup (NEW)
  └── Browser opens http://localhost:18789/setup
  └── OR CLI wizard if --no-ui flag

Step 3: Create Account (NEW)
  └── Email + password
  └── OR OAuth (Google, GitHub)
  └── Creates first organization automatically

Step 4: Generate API Key (NEW)
  └── Shows key ONCE (copy now!)
  └── Key stored in Infisical/config

Step 5: Configure LLM
  └── Select provider (Anthropic recommended)
  └── Paste API key
  └── Test connection ✓

Step 6: First Message
  └── "Hello, OpenClaw!" → Response
  └── Setup complete!
```

#### 2. Web Setup Wizard

New route: `GET /setup` (only when unconfigured)

```
┌──────────────────────────────────────────┐
│           OpenClaw Setup                  │
├──────────────────────────────────────────┤
│                                          │
│  Step 1 of 4: Create Account             │
│                                          │
│  Email: [_______________________]        │
│  Password: [___________________]         │
│                                          │
│  [ ] I accept the terms                  │
│                                          │
│              [Continue →]                │
│                                          │
│  ─────────── or ───────────              │
│                                          │
│  [G] Continue with Google                │
│  [→] Continue with GitHub                │
│                                          │
└──────────────────────────────────────────┘
```

---

## Decomposition Phase (D)

### Implementation Waves

#### Wave 1: Foundation (Priority: Critical)

| Task | Description | Depends On | Size |
|------|-------------|------------|------|
| T001 | Install Better-Auth + SQLite adapter | - | small |
| T002 | Define user/org/apikey schema | T001 | small |
| T003 | Create auth routes (/auth/*) | T002 | medium |
| T004 | Migrate gateway token to API key | T003 | medium |
| T005 | Add backward compatibility layer | T004 | small |

#### Wave 2: Secrets Management (Priority: High)

| Task | Description | Depends On | Size |
|------|-------------|------------|------|
| T006 | Add Infisical SDK integration | - | small |
| T007 | Add dotenvx fallback | T006 | small |
| T008 | Create secrets loading pipeline | T007 | medium |
| T009 | Migrate LLM credentials to Infisical | T008 | medium |
| T010 | Document secrets setup | T009 | small |

#### Wave 3: Onboarding UX (Priority: High)

| Task | Description | Depends On | Size |
|------|-------------|------------|------|
| T011 | Create web setup wizard route | T003 | medium |
| T012 | Build account creation UI | T011 | medium |
| T013 | Build LLM config UI | T012 | medium |
| T014 | Add "test connection" feature | T013 | small |
| T015 | Auto-launch setup after install | T014 | small |

#### Wave 4: API Keys & Scopes (Priority: Medium)

| Task | Description | Depends On | Size |
|------|-------------|------------|------|
| T016 | Implement API key creation | T003 | medium |
| T017 | Add scope enforcement middleware | T016 | medium |
| T018 | Build API key management UI | T017 | medium |
| T019 | Add key rotation CLI command | T018 | small |
| T020 | Add key expiry warnings | T019 | small |

#### Wave 5: Advanced Features (Priority: Low)

| Task | Description | Depends On | Size |
|------|-------------|------------|------|
| T021 | Add organization/team support | T003 | large |
| T022 | Implement audit logging | T021 | medium |
| T023 | Add usage tracking | T022 | medium |
| T024 | Build usage dashboard | T023 | medium |
| T025 | Add rate limiting | T024 | medium |

---

## Migration Strategy

### Phase 1: Parallel Systems (Weeks 1-2)
- Better-Auth runs alongside existing auth
- New users get Better-Auth
- Existing users keep working with gateway token
- Flag: `OPENCLAW_AUTH_MODE=legacy|modern`

### Phase 2: Migration Tools (Weeks 3-4)
- CLI command: `openclaw auth migrate`
- Imports existing credentials into Better-Auth
- Generates API keys for existing devices
- Updates config files automatically

### Phase 3: Deprecation (Weeks 5-8)
- Warnings on legacy auth usage
- Documentation updated
- Migration deadline announced

### Phase 4: Removal (Week 9+)
- Legacy auth code removed
- Gateway token no longer accepted
- Only Better-Auth API keys work

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Migration breaks existing users | Medium | High | Backward compatibility + gradual rollout |
| Better-Auth CVEs | Low | High | Pin to v1.3.26+, monitor security advisories |
| Infisical complexity | Medium | Medium | dotenvx fallback for simple setups |
| User resistance to accounts | Medium | Medium | Allow "anonymous" local-only mode |
| Performance impact | Low | Medium | Session caching, lazy loading |

---

## Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Time to first message | ~15 min | < 5 min |
| Onboarding completion rate | Unknown | > 80% |
| Support tickets (auth issues) | Unknown | -50% |
| Security incidents | Unknown | 0 |
| API key rotation compliance | 0% | > 90% |

---

## Next Steps

1. **Review this document** with stakeholders
2. **Validate assumptions** about Better-Auth integration
3. **Create detailed specs** for Wave 1 tasks
4. **Set up development environment** with Better-Auth
5. **Begin T001** (Install Better-Auth + SQLite adapter)

---

## References

- [Better-Auth Documentation](https://www.better-auth.com/)
- [Infisical GitHub](https://github.com/Infisical/infisical)
- [dotenvx Documentation](https://dotenvx.com/)
- [OpenAI API Authentication](https://platform.openai.com/docs/api-reference/authentication)
- [OpenClaw Tech Stack Docs](../openclaw-src/docs/techstack/)
