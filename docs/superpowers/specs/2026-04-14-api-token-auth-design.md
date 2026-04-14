# API Token Auth — Design

**Date:** 2026-04-14
**Target version:** 1.3.0
**Status:** Approved

## Motivation

The LM Studio server was moved off `localhost` onto the local network and is now protected by an API token. The extension currently sends unauthenticated requests and will fail against the new server. Users need a secure way to supply the token.

## Goals

- Send `Authorization: Bearer <token>` on all requests when a token is configured.
- Store the token securely (VS Code `SecretStorage`, not settings.json).
- Provide a command to set/clear the token.
- Surface clear errors on 401/403.
- Stay backwards-compatible: no token → unauthenticated requests as before.

## Non-goals

- No per-workspace tokens (global only).
- No OAuth, refresh tokens, or token rotation.
- No username/password auth.

## Components

### 1. SecretStorage access

- Secret key: `lmChat.apiToken`.
- `extension.ts` passes `context.secrets` into `LmStudioClient` (constructor param).
- Client reads the secret lazily before each request — no cached copy, so "set token" takes effect immediately.

### 2. Command: `lmChat.setApiToken`

- Title: **"LM Chat: Set API Token"**.
- Opens `vscode.window.showInputBox({ password: true, placeHolder: 'Paste API token (leave empty to clear)' })`.
- Empty string / cancel cleared via `secrets.delete(...)`.
- On success: info message *"API token updated"* / *"API token cleared"*.
- Registered in `package.json` under `contributes.commands` and visible in the command palette.

### 3. `LmStudioClient` changes

- Constructor accepts `secrets: vscode.SecretStorage`.
- New private `async getAuthHeaders(): Promise<Record<string, string>>` — returns `{ Authorization: 'Bearer <token>' }` if token present, else `{}`.
- Called in: `healthCheck()`, `listModels()`, `streamChat()` (and any other HTTP request).

### 4. Error handling

- On `401` or `403` response:
  - Throw an error with message: *"Authentication failed (HTTP <code>). Run 'LM Chat: Set API Token' to update your token."*
  - `chatViewProvider` already surfaces client errors in the chat view — no extra wiring needed, just ensure the message is descriptive.

### 5. `package.json`

- Add the new command under `contributes.commands`.
- No new setting (secrets don't live in settings.json).

## Data flow

```
User → Command Palette → lmChat.setApiToken
    → showInputBox → context.secrets.store('lmChat.apiToken', value)

Chat message → LmStudioClient.streamChat
    → getAuthHeaders() reads secrets
    → fetch(..., { headers: { ..., Authorization: 'Bearer <token>' } })
```

## Testing (manual, Extension Development Host)

1. **No token, unauth server** → chat works (regression check).
2. **Token set, auth server** → chat works; health + model list succeed.
3. **Wrong token** → chat view shows the "Authentication failed" message.
4. **Clear token (empty input)** → secret removed; reverts to unauthenticated.
5. **Set token command** → visible in palette, masked input.

## Version bump

`1.1.7` → `1.3.0` via `/roll`.
