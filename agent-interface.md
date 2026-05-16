# Agent for Auth Service

## Chat interface

The agent exposes a chat endpoint over a Unix domain socket. `test-chat-functional.mjs` is the conformance smoke test for this section; You can use it to test your implementation.

### Transport

- **Address family**: AF_UNIX, type `SOCK_STREAM`.
- **Socket path**: read from the environment variable `SOCKET_PATH`. If unset, the agent MUST bind to `/tmp/agent.sock`.
- **Permissions / lifecycle**: the agent owns the socket file. It MUST remove any stale socket file at the path before `bind()`, and SHOULD `unlink()` it on clean shutdown. The file MUST be readable and writable by the user running the test.
- **Concurrency**: the agent MUST accept multiple sequential connections. Within a single connection it MAY process requests serially.
- **Connection lifetime**: a connection is valid until either side closes it. The client may send any number of requests on one connection. The agent MUST NOT close the connection after a single reply.

### Framing

Newline-delimited JSON (NDJSON), UTF-8 encoded:

- Each request is exactly one JSON object followed by a single `\n` (0x0A) byte.
- Each reply is exactly one JSON object followed by a single `\n` byte.
- No partial messages, no embedded literal newlines inside the JSON object. Whitespace inside the JSON object is allowed but discouraged.
- The agent MUST emit the reply's trailing `\n` so the client can detect end-of-message; the client's read loop terminates on the first `\n`.

### Request schema

```json
{ "query": "<string>" }
```

- `query` (string, required): the user's message. May be empty or arbitrary text up to 16 KiB.
- Unknown fields can be ignored.

### Reply schema

```json
{ "reply": "<string>" }
```

- `reply` (string, required): the agent's response. MUST be a non-empty string
— the test treats an empty/whitespace-only reply as a failure.

### Latency

- The agent SHOULD respond within 30 seconds; exceeding it is a failure.

### Error handling

- Malformed requests (invalid JSON, missing `query`) SHOULD be answered with a reply object whose `reply` field is a human-readable error string. The agent MUST NOT terminate the connection in response to a bad request — it MUST keep serving subsequent requests on the same connection.
- Internal agent errors MUST be surfaced as a non-empty `reply` string rather than by closing the socket silently.


## Startup Interface

Any compliant agent implementation MUST satisfy the following startup contract.

### Executable

A script named `start-agent` (no extension) at the project root, invocable as `./start-agent`.

### Readiness signal

When ready to accept connections the agent MUST emit exactly one JSON line on **stdout**:

```json
{ "type": "agent.listening", "socket": "<SOCKET_PATH>" }
```

### Shutdown

The agent MUST handle `SIGTERM` by closing the socket and exiting cleanly.

### Stdout format

All log output MUST be NDJSON (one JSON object per line).

---

## Workflows

These illustrate how the test will interact with the agent, your implementation should optimize for these use cases

### Workflow 1 — Sign up a new account

- `[user]` "Sign me up with email alice@example.com, username alice, password hunter2hunter2."
- `[agent]` Calls `auth.register` with those fields.
- `[agent]` Receives `emailVerificationToken` and reports: "Account created. A verification email was sent — paste the link/token when it arrives."
- `[user]` Opens email, clicks the verification link, pastes the token back into the chat.
- `[agent]` Calls `auth.email_verify` with the token and confirms: "Email verified, you're all set."

### Workflow 2 — Log in with MFA

- `[user]` "Log me in as alice / hunter2hunter2."
- `[agent]` Calls `auth.login`. Response is `mfaRequired: true` with `methods: ["totp"]`.
- `[agent]` Tells the user: "MFA required — open your authenticator app and share the 6-digit code."
- `[user]` Reads the code from the authenticator app, replies "code is 481920".
- `[agent]` Calls `auth.mfa_verify` with the code. Session becomes authenticated; agent confirms login.

### Workflow 3 — Forgot password

- `[user]` "I can't log in, please reset my password. Email is alice@example.com."
- `[agent]` Calls `auth.password_reset_request` with the email.
- `[agent]` "Reset link sent. Paste the token from the email when it arrives, plus your new password."
- `[user]` Pastes the reset token and a new password.
- `[agent]` Calls `auth.password_reset_confirm`.
- `[agent]` "Password updated. All existing sessions were signed out — log in again when ready."

### Workflow 4 — Enroll TOTP MFA

- `[user]` (already signed in) "Turn on two-factor with an authenticator app."
- `[agent]` Calls `auth.mfa_enroll_start` with `method: "totp"`.
- `[agent]` Returns the `otpauthUrl` and asks the user to add it to their authenticator.
- `[user]` Scans the QR / adds the secret, replies with the current 6-digit code.
- `[agent]` Calls `auth.mfa_enroll_confirm` with `enrollmentId` and the code.
- `[agent]` Shows the returned `recoveryCodes` once and warns: "Save these — they won't be shown again."
- `[user]` Confirms they've stored the codes.

### Workflow 5 — Audit and revoke devices

- `[user]` "Show me everywhere I'm signed in."
- `[agent]` Calls `me.devices_list`, summarizes each session (device, last-seen, marks the current one).
- `[user]` "Sign out the iPad."
- `[agent]` Calls `me.device_revoke` with that device's id and confirms removal.

### Workflow 6 — Update profile (with username collision)

- `[user]` "Change my display name to Alice Liddell and my username to alice_l."
- `[agent]` Calls `me.update` with both fields.
- `[agent]` Receives `username_taken`. Reports the conflict and asks the user to pick another.
- `[user]` "Try alice.liddell."
- `[agent]` Retries `me.update`, succeeds, shows the updated profile.

### Workflow 7 — Block an abusive user

- `[user]` "Block user usr_abc123 — they keep spamming me."
- `[agent]` Calls `me.block_user` with `reason: "spam"`.
- `[agent]` "Blocked. Run `me.blocks_list` to see everyone you've blocked."

### Workflow 8 — Log in with an MFA recovery code

- `[user]` "Log me in as alice / hunter2hunter2."
- `[agent]` Calls `auth.login`. Response is `mfaRequired: true`.
- `[agent]` Tells the user: "MFA required — enter your authenticator code, or a recovery code if you've lost access."
- `[user]` "I lost my authenticator, use this recovery code: XXXX-XXXX."
- `[agent]` Calls `auth.mfa_verify` with the recovery code. Session becomes authenticated; agent confirms login.
- `[agent]` Warns: "That recovery code has been consumed — you have N remaining. Consider re-enrolling your authenticator."

### Workflow 9 — Log out everywhere (account compromise)

- `[user]` "I think my account was compromised — sign me out everywhere and reset my password."
- `[agent]` Calls `auth.logout { allDevices: true }`.
- `[agent]` Calls `auth.password_reset_request` for the user's email.
- `[agent]` Walks the user through Workflow 3 from the reset-token step onward.
