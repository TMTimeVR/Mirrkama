(Notes for myself)

# PART 1: RULES & MENTAL MODEL

Read this part before writing any code. Every design question later gets answered by these rules.

## Who trusts who

- Client trusts Nakama. Game servers (Mirror) trust Nakama. Nakama decides who is allowed to join a server.
- Identity starts at the platform: Nakama trusts META (nonce validation against graph.oculus.com) to vouch that a claimed org-scoped ID really belongs to this client. A platform ID without a validated nonce is worth nothing.
- Meta also vouches for DEVICE and APP integrity (Attestation API): Nakama generates a challenge nonce, the headset returns a Meta-signed attestation token, Nakama verifies it server-side. Client-side validation of any of this would be worthless, a modified client just skips it.
- Dedicated servers never see the session token. Joining uses a single-use JOIN TICKET: Nakama issues it over TLS, the game server hands it back to Nakama for validation, and the connection's identity is whatever Nakama says the ticket was issued to. Nothing the client claims about who it is counts.
- The game wire itself is ENCRYPTED (pinned per-server keys, distributed through list_servers over TLS): voice and gameplay are personal data and do not cross the internet in plaintext.
- "Ticket is valid" and "account in good standing" are answered in the SAME Nakama call at join; afterwards, ban status flows via push + poll (Phase 5).
- **server_key: the one clients use. Public.**
- **http_key: a completely different thing. Server-to-server RPC endpoints. Secret.**
- One secret per trust edge, never reused: http_key (game server -> Nakama), kick secret (Nakama -> game server admin endpoint), enforcement secrets (Discord bot -> Nakama).
- Nakama is the only source of truth.
- Community backends are trusted for NOTHING that touches official state.

## Core principles

- The client can't tell the server to do anything. It always expresses intent, never state. Validate everything.
- **Always code as if the client is hacked and will try to break the system.**
- Never trust a caller about anything I can determine myself (time, source IP, identity).
- Clients only ever see projections of internal data (through RPCs), never raw storage.
- **register_server is only callable server-to-server via http_key. A user session must NEVER be able to reach it.** If a user session could register a server, anyone could inject a malicious "official" server into the list and harvest traffic from everyone who connects. Worst possible vulnerability in the whole design.

## The system in one paragraph

There is ONE Nakama, and MANY game servers. Nakama keeps a phone book of game servers. Game servers call Nakama every 10 seconds to say "I'm still alive, here's my player count." Clients never see the phone book directly: they call `list_servers`, and Nakama hands them a filtered copy containing only healthy servers. When I ban someone, Nakama reads the same phone book to know which servers to send the kick to. One registry, three readers: heartbeat writes it, list_servers reads it for clients, ban-push reads it for enforcement.

## Data flow

```
LOGIN
Client ──"give me a login challenge"──> Nakama ──generates challenge_nonce──> Client
Client ──GetIntegrityToken(challenge_nonce) + GetUserProof()──> Meta Platform SDK
Client ──org-scoped ID + user-proof nonce + attestation token──> Nakama
Nakama ──verify user proof + attestation token──> graph.oculus.com
        then checks claims itself: nonce match, freshness, package + cert pin,
        device integrity, device_ban.is_banned -> reject, account banned -> reject
Nakama ──session token (JWT)──> Client

BOOT / EVERY 10s
Game server ──POST register_server (http_key)──> Nakama ──writes──> registry storage

CLIENT WANTS TO PLAY
Client ──RPC list_servers (session token)──> Nakama ──reads registry, filters
        stale/full──> [address, port, region, players, transport key fingerprint]
Client ──RPC request_join_ticket(server_id)──> Nakama ──ban check + live check──>
        single-use 30 s ticket ──> Client
Client ──Mirror connect over ENCRYPTED transport (pins server key)──> game server
Game server ──validate_join_ticket (http_key)──> Nakama ──atomic single-use check
        + ban/mute state──> {user_id, banned, muted_until} ──> identity set, allowed in

BAN HAPPENS
Discord bot ──RPC ban (ban secret)──> Nakama:
   1. mark banned in storage + append enforcement record
   2. sessionLogout
   3. read registry ──> POST kick to every live server's kick endpoint (kick secret)
   4. severe tier only: ──device_ban(last-seen unique_id)──> graph.oculus.com,
      store the returned ban_id in the enforcement record

REPORT
Client ──RPC report_user (session token)──> Nakama ──stores report──> reports storage
Nakama ──webhook (category + IDs only)──> #reports ──mod acts via bot──> BAN HAPPENS flow above
```

## Mental model to keep

The registry is not "configuration": it is a live, self-rebuilding picture of the fleet that expires on its own. I never trust it to be complete (poll enforcement covers gaps) and I never trust its writers about anything I can determine myself (time, source IP). Clients only ever see a projection of it, never the thing itself.

---

# PART 2: BUILD PLAN

Work through the phases in order. Each phase ends with an exit test: don't move on until it passes.

## Phase 0: This document

- [x] Resolve the remaining open decisions in Part 3 (three retention periods; everything else is decided).
- [ ] Re-read Part 1 once before every phase. Cheap, and it keeps the rules loaded.

## Phase 1: Nakama standalone + hardening

- [x] Docker compose: Nakama + Postgres on the VPS.
- [x] BEFORE anything is internet-facing: change the default server_key, http_key, and console password. Nakama defaults are publicly known and scanners look for them.
- [ ] Console (port 7351) is NOT publicly exposed. Behind nginx with an allowlist, or not exposed at all (SSH tunnel when I need it).
- [ ] TLS termination for the client-facing API via certbot on the host nginx (same pattern as mail.tmtime.dev).
- [ ] Unity client authenticates via the Meta platform flow and receives a session token:
  1. Client asks the Platform SDK for a user proof: `Users.GetUserProof()` returns a NONCE (single-use, short-lived; fetch a fresh one for every login attempt, never cache).
  2. Client sends org-scoped ID + nonce to Nakama (custom authentication, over TLS).
  3. Nakama, in a beforeAuthenticateCustom hook, calls Meta's validation endpoint (graph.oculus.com/user_nonce_validate) with the nonce, the claimed user ID, and MY app access token (app ID + app secret). Meta answers valid/invalid.
  4. Invalid, expired, or reused nonce -> reject login. NEVER create or log into an account from an unvalidated platform ID, or anyone can impersonate anyone by claiming an ID.
- [ ] The Meta app access token lives ONLY in Nakama runtime config. Never in the client build, never on game servers (they validate join tickets via Nakama, they touch neither platform auth nor session tokens).
- [ ] Namespace the account ID: store it as `meta:<org-scoped-id>`, not the bare number, so a future Steam/PICO port can't collide with Meta IDs.
- [ ] Fail closed: if graph.oculus.com is unreachable, logins fail. Existing sessions keep working (tokens already issued), same philosophy as the Nakama-outage rule in 4.6.
- [ ] Docs:
  - Nakama Docker install: https://heroiclabs.com/docs/nakama/getting-started/install/docker/
  - Nakama server configuration (keys, console, session TTLs): https://heroiclabs.com/docs/nakama/getting-started/configuration/
  - Nakama authentication + before-hooks: https://heroiclabs.com/docs/nakama/concepts/authentication/ and https://heroiclabs.com/docs/nakama/server-framework/
  - Meta User Verification (GetUserProof, user_nonce_validate): https://developers.meta.com/horizon/documentation/unity/ps-ownership/
  - Meta Attestation API: https://developers.meta.com/horizon/documentation/unity/ps-attestation-api/ (if the slug moves, search "Attestation API" in the Horizon Unity docs; the full page text is saved in my notes)
  - certbot: https://certbot.eff.org/instructions and nginx WebSocket proxying: https://nginx.org/en/docs/http/websocket.html
- [ ] Attestation in the SAME login flow (Attestation API; Quest 2/Pro/3/3S only, so minimum supported device is Quest 2):
  1. Login becomes two round trips. Client first asks Nakama for a login challenge; Nakama generates a crypto-random Base64URL challenge_nonce (single use, short TTL, stored server-side).
  2. Client calls `DeviceApplicationIntegrity.GetIntegrityToken(challenge_nonce)` alongside `GetUserProof()`, then sends org-scoped ID + user-proof nonce + attestation token to authenticateCustom.
  3. The two nonces run in OPPOSITE directions, don't mix them up: the user-proof nonce is CLIENT-generated and Nakama validates it with Meta; the attestation challenge_nonce is NAKAMA-generated and must come back inside the Meta-signed token claims.
  4. Nakama verifies the attestation token at graph.oculus.com/platform_integrity/verify (same OC access token), then checks the claims itself: nonce matches the stored challenge (then delete the challenge, single use), timestamp is fresh, package_id and package_cert_sha256_digest match MY build (cert pinning catches repackaged APKs regardless of install source), app_integrity_state is StoreRecognized (Part 3 decision), device_integrity_state is Advanced or Basic (NotTrusted -> reject), and the device_ban section: is_banned -> reject (can surface remaining_ban_time in the rejection message).
  5. Fail closed, per Meta's own best practice: persistent attestation errors are treated the same as NotTrusted. Rate limits (100/hour, 200/day per device) are irrelevant at one token per login.
- [ ] Attestation claims are checked and DISCARDED, no routine device tracking. The only thing persisted: the last-seen unique_id per account, kept at most 30 days (it rotates and goes stale by itself), so a device ban can still be issued shortly after an incident.
- [ ] app_integrity_state: DECIDED, Horizon Store only (Part 3): require StoreRecognized at login. Betas ship via Horizon release channels (alpha/beta), which still count as store-installed. Dev exemption for sideloaded builds only via an allowlist of MY verified org-scoped ID, never via anything the client claims. Cert pinning above stays on either way.
- [ ] File the Data Use Checkup EARLY (Meta dashboard -> Requirements -> Data Use Checkup): User Verification itself requires an approved DUC, and until approval, platform features only work for TEST USERS. Device ban is a second DUC item (Phase 5). Approval takes review time: this is the longest lead-time item in the whole plan, start it before writing code.
- [ ] The login-challenge RPC must be reachable WITHOUT a session (it runs before authentication exists). Nakama supports invoking RPCs with the server_key for pre-auth flows: verify the exact mechanism in the docs. Then treat it as attack surface: rate limit challenge issuance under the same per-IP limiter as login, cap outstanding challenges per IP, and let the ~2 min TTL garbage-collect the rest. Unbounded challenge storage would be a free storage-flood DoS.
- [ ] nginx details that bite later: proxy port 7350 WITH WebSocket upgrade headers (the realtime socket is WS), and forward the real client IP (X-Forwarded-For / realip module), otherwise every rate limiter behind nginx only ever sees nginx's own address.
- [ ] SECRETS LEAK INTO MY OWN LOGS unless prevented: s2s calls carry http_key in the query string, and the client's realtime WebSocket may carry the session token as a ?token= query parameter (verify against the Unity client). Default nginx access logs record the full request line, so the access log becomes a secrets file. Before go-live: redact or drop query strings in the access log for the Nakama vhost, and point game servers at a non-logged internal vhost where possible. This also shrinks what the log-retention decision has to protect.
- [ ] Token lifetimes in Nakama config: session 15 min, refresh 5 h.
- [ ] Exit test: authenticate from Unity over TLS with a fresh nonce. Replay the SAME nonce a second time -> rejected. Claim someone else's org-scoped ID with a garbage nonce -> rejected. Restart Nakama, authenticate again. Attestation: replay an old challenge_nonce -> rejected. Garbage attestation token -> rejected. Confirm my own DEV-MODE headset still passes device integrity (otherwise I just locked myself out of my own game). Device-ban my own test device, confirm login is refused, then REVERSE it via the ban_id. Sideloaded build on a non-exempt account -> rejected (StoreRecognized); on my exempt org-scoped ID -> passes. (The device-ban part of this test needs the approved Data Use Checkup first.)

## Phase 2: The auth spine (join tickets over an encrypted transport)

DECIDED (Part 3): the session token NEVER touches the game wire, and the game wire itself is encrypted. Game servers hold no session encryption key.

### Step 2.1: Encrypt the transport

- [ ] Wrap KCP with Mirror's encryption transport (ephemeral key exchange + authenticated encryption). Docs: the Transports section at https://mirror-networking.gitbook.io/docs (Encryption transport page).
- [ ] Each game server generates a persistent keypair on first boot (file next to its server_id). The SHA-256 fingerprint of the public key goes into the register_server payload and is FROZEN on first registration, exactly like the IP (Step 4.2). Rotation = manual operator edit, same trade-off as addresses.
- [ ] list_servers returns the fingerprint alongside address/port (Step 4.5), over TLS: the client learns each server's key from a channel it already trusts.
- [ ] The client MUST validate the server's public key against that fingerprint during the handshake and abort on mismatch. An unauthenticated key exchange is MITM-able; the pinning is what turns encryption into security. Verify Mirror's encryption transport exposes this validation hook in the current version: HARD requirement, not a nice-to-have. If it truly doesn't, that's a blocker to solve before shipping, not to work around.
- [ ] What this buys beyond the token: VOICE and gameplay packets stop crossing the internet in plaintext. Voice is personal data; encrypting it in transit is an Art. 32 matter, not a luxury.

### Step 2.2: The join ticket

- [ ] New user-callable RPC `request_join_ticket` (ctx.userId REQUIRED, over TLS): payload = target server_id. Nakama checks the account isn't banned, checks the server is live (registry + staleness filter), then mints an OPAQUE ticket: 32 random bytes, stored as {ticket -> user_id, server_id, issued_at}, TTL ~30 s, single use. Opaque beats signed: nothing to forge, validation is a lookup, revocation is free. One active ticket per user (a new request voids the old one): caps storage and doubles as rate limiting.
- [ ] Client connects through the encrypted transport; the Mirror auth message carries ONLY the ticket. Unchanged pre-auth rules: size-limit the auth message, short timeout for unauthenticated connections, process NO other message type before auth succeeds.
- [ ] The game server's NetworkAuthenticator calls Nakama `validate_join_ticket` (server-to-server via http_key: same trust edge as heartbeats) with {ticket, my server_id}.
- [ ] Nakama validation, in order: ticket exists, not expired, not used, bound server_id matches the caller's claim, and (hardening) the call's source IP matches that server_id's registered IP: mismatch -> reject + loud log. Mark used ATOMICALLY (conditional write / version check) so two racing uses can't both win. Then answer {user_id, banned, muted_until, muted_permanent} in ONE response: the old join-time ban check is merged in, zero extra round trips.
- [ ] The game server takes the connection's identity FROM NAKAMA'S ANSWER. Not from the client, not from anything it parsed itself. The connection is whoever Nakama says the ticket was issued to.
- [ ] Fail closed: Nakama unreachable -> joins fail, existing players keep playing (validated at join), consistent with 4.6.

### Step 2.3: What changed in the trust model

- [ ] Game servers no longer hold the session encryption key AT ALL. It lives in exactly one place: Nakama. A compromised game server has no token-minting power anymore (Phase 10 playbook updated to match).
- [ ] Residual ticket-theft analysis, written down so future-me doesn't re-derive it: a stolen ticket impersonates its user for ONE join, within 30 s, on ONE server. The encrypted pinned transport makes stealing it impractical in the first place; the atomic single-use mark makes replay impossible even so. Layers, not either/or.
- [ ] Docs: Mirror authenticators: https://mirror-networking.gitbook.io/docs/manual/components/network-authenticators . Mirror transports (encryption page): https://mirror-networking.gitbook.io/docs . Nakama sessions: https://heroiclabs.com/docs/nakama/concepts/sessions/

### Step 2.4: Exit test

- [ ] Evil client: garbage payloads, oversized payloads, random tickets, expired tickets, a REAL ticket used twice (second use rejected), a real ticket for server A presented to server B (rejected), messages before auth. All rejected cleanly, no gameplay code reached.
- [ ] Race test: two connections present the same ticket simultaneously; exactly one wins.
- [ ] MITM test: connect through a proxy presenting a different transport key; the client aborts on fingerprint mismatch.
- [ ] Packet capture during a live session: no plaintext voice, no plaintext gameplay, no token, no ticket readable anywhere.

## Phase 3: Rooms + authority discipline

- [ ] Mirror Multiple Matches pattern with NetworkMatch: one server process, isolated rooms.
- [ ] Every [Command] validates every argument: bounds, rate, and ownership (does this connection actually own the object it's commanding?).
- [ ] Authoritative state lives server-side only. Client input is a request, never a result.
- [ ] Logging discipline from day one: log user IDs and violation events. Do not hoard IPs tied to behavior; IPs are personal data (GDPR), define retention before collecting.
- [ ] Docs: https://mirror-networking.gitbook.io/docs : the Interest Management and Remote Actions ([Command]/[ClientRpc]) pages. The multi-match demo lives in the repo under Assets/Mirror/Examples: https://github.com/MirrorNetworking/Mirror
- [ ] Exit test: modified client tries to command objects it doesn't own, teleport, and spam. Server rejects all, logs the violations, and the room stays consistent for other players.

## Phase 4: Fleet: registration, heartbeat, list_servers

Docs: Nakama storage engine: https://heroiclabs.com/docs/nakama/concepts/storage/ . Server-to-server RPC + http_key: the "server to server" page under https://heroiclabs.com/docs/nakama/server-framework/ (mind the unwrap payload quirk). Fleet-orchestration concepts as background reading: https://agones.dev/site/docs/

### Step 4.1: Provision the ID list (5 min, one-off)

- [ ] Storage collection `server_registry`, one object per allowed server: key = `mirror1`, `mirror2`, ... Value starts as `{ "provisioned": true }`.
- [ ] Permissions: NO public read, NO public write. Only runtime code touches it.
- [ ] Rule this encodes: a registration for an ID not in this list is REJECTED. An attacker with a leaked http_key still needs a valid ID.

### Step 4.2: The register_server RPC (guard first, logic second)

- [ ] Line 1: `if (ctx.userId) -> throw generic error + log "USER SESSION TRIED register_server"`. This log line is a TRIPWIRE. If it ever fires, someone is attacking exactly the hole I predicted. Never delete this log.
- [ ] Parse payload: server_id, port, kick_port, region, current_players, max_players, transport pubkey_fingerprint (Phase 2). Validate types and ranges strictly (port 1-65535, players >= 0, region from a fixed allowlist). Reject anything malformed, no "fixing up" input.
- [ ] Look up server_id in `server_registry`. Not provisioned -> reject + log.
- [ ] FIRST registration (no IP stored yet): store the IP AND the transport pubkey_fingerprint. Also record the observed source IP of the HTTP call and log if it differs from the claimed one (misconfig or attack, either way I want to know).
- [ ] SUBSEQUENT calls: IGNORE any address/port/fingerprint in the payload. The address is frozen. Changing it = me editing storage manually. Deliberate trade-off: costs me manual work, but a leaked key can't re-point mirror1 at an attacker's machine.
- [ ] Every call: update current_players, set `last_heartbeat` = current NAKAMA server time. Never accept a timestamp from the payload. The caller does not get to say what time it is.
- [ ] Registration and heartbeat are THE SAME RPC: an upsert keyed on server_id. There is no separate heartbeat endpoint to build.
- [ ] Exit test: curl with http_key -> works. Same call as a logged-in user from Unity -> generic error + tripwire log appears. Do not skip the second test.

### Step 4.3: Staleness (what makes the list self-healing)

- [ ] One constant, one place in code: `STALE_AFTER = 25s` (2 missed 10s beats + margin; see Part 3).
- [ ] NO cleanup cron needed. Staleness is checked lazily by the readers: every read of the registry (list_servers AND ban-push) filters out entries where `now - last_heartbeat > STALE_AFTER`.
- [ ] Consequence: a server that stops heartbeating disappears from clients automatically AND stops receiving push-kicks. Poll enforcement still covers it; that's why the poll layer exists. The registry is enforcement-critical, not just discovery-critical.

### Step 4.4: Deregister (graceful shutdown)

- [ ] Same RPC, extra payload flag: `"shutting_down": true`. Nakama marks the entry stale immediately instead of letting it linger 25s while players try to join a dying server.
- [ ] Game server side: SIGTERM handler -> stop accepting new Mirror connections -> call register_server with the flag -> shut down.

### Step 4.5: list_servers RPC (client-facing projection)

- [ ] User-callable: here ctx.userId MUST be present; reject server-to-server calls (mirror image of 4.2's guard).
- [ ] Read all entries -> drop stale -> drop full (current >= max) -> return ONLY: address, port, region, current_players, max_players, transport pubkey_fingerprint.
- [ ] NEVER return: kick_port, internal IDs, heartbeat times, anything else. The RPC is the filter; that's why the collection has no public read.
- [ ] Exit test: dump the RPC response as a hostile client would see it, check nothing internal leaked.

### Step 4.6: Fleet failure drill

- [ ] kill -9 a game server -> vanishes from list_servers within ~25s.
- [ ] Graceful stop -> vanishes immediately.
- [ ] Stop Nakama 30s -> game servers retry heartbeats with backoff and KEEP their current players (sessions were validated at join; don't dump players because the registry blinked). New joins failing during the outage is correct. When Nakama returns, the registry rebuilds within one heartbeat cycle: verify it does.
- [ ] Unprovisioned ID -> rejected + logged.
- [ ] Heartbeat with a changed address for a registered ID -> ignored, mismatch logged.

## Phase 5: Ban system

Target: push ~1 s, poll guarantees <= 15 s worst case, door checks stop returns.

- [ ] Kick endpoint on each game server: small HTTP listener, separate from the game port, accepts one command: "disconnect user X now." Gated with the kick secret (NOT http_key: different trust edge, different secret). Not publicly reachable if topology allows. Payload validated strictly (user ID format, nothing else). Replay protection: authenticate with an HMAC over the body using the kick secret (not a bare secret header), include a Nakama-set timestamp in the body, and reject anything older than ~30 s. A captured kick request then expires instead of working forever. TLS or a private interface on top where topology allows.
- [ ] Ban RPC in Nakama (called by the Discord bot with the ban secret):
  1. Mark account banned in storage (the authoritative act; everything else is propagation), and append a per-account ENFORCEMENT RECORD: timestamp, action type (ban/mute), duration, public reason, acting moderator, internal note. Collection has NO public read; the user-facing view is an RPC projection (Phase 9).
  2. Invalidate the Nakama session server-side (sessionLogout: kills socket + refresh path for that session).
  3. Read live servers via the SAME `getLiveServers()` function as list_servers (shared function so the two can never drift), POST the kick to every server's kick endpoint. Parallel, 2-3 s timeout per server so one dead server can't stall the ban. Push failure is logged but does NOT fail the ban; poll and join-check mop up.
- [ ] Poll fallback: each game server, every 10 s, sends Nakama the list of ITS currently connected user IDs and asks "any banned?" Server disconnects hits. Servers never hold a copy of the full ban list (smaller request, better for privacy).
- [ ] Door checks: Nakama refuses LOGIN for banned accounts AND refuses TOKEN REFRESH for banned accounts. Without the refresh check, a banned user who dodged the kick keeps minting fresh session tokens for up to 5 h.
- [ ] Join-time ban check already built in Phase 2 (merged into validate_join_ticket: one call answers identity, ban, and mute state).
- [ ] Device-ban escalation for SEVERE or repeat cases (Attestation API device ban; needs one-time Data Use Checkup approval in the Meta dashboard before it works):
  1. Nakama calls graph.oculus.com/platform_integrity/device_ban server-to-server (same OC access token) with the offender's last-seen unique_id and a duration.
  2. Store the returned ban_id in the enforcement record. The unique_id rotates every 30 days; the ban_id is the DURABLE handle for updating or reversing the ban. Lose it and the device has to show up again before I can touch the ban.
  3. Enforcement stays on MY side: Meta only carries the flag. My login check (device_ban claim in the attestation token) is what actually refuses service, and it fires BEFORE any account exists, so a fresh alt account on a banned headset is caught at the door.
  4. Reserved for severe/repeat abuse, every one logged with a reason: Meta's Platform Abuse Policy makes ME responsible for conscientious use.
- [ ] NO IP bans, ever. The device ban replaces that idea and beats it on every axis: it hits exactly one headset instead of a whole shared IP (CGNAT would punish innocents), it survives IP rotation, and it stores a rotating pseudonym + ban_id instead of IP addresses.
- [ ] Docs: Nakama runtime function reference (sessionLogout and friends): under https://heroiclabs.com/docs/nakama/server-framework/ . Meta device ban: the device-ban section of the Attestation API page (endpoints graph.oculus.com/platform_integrity/device_ban and /device_ban_ids).
- [ ] Exit test: ban a connected test account. It gets kicked in ~1 s. Ban while its server's kick endpoint is down: kicked within 15 s by poll. Banned account cannot log in, cannot refresh, cannot join.

## Phase 6: Token rules & rate limiting

- [ ] Refresh token ROTATION: every refresh issues a new refresh token and invalidates the old one. A stolen refresh token then dies the next time the legitimate client refreshes (~10 min), instead of living 5 h.
- [ ] VERIFY rotation instead of assuming it: confirm in the Nakama session configuration whether refreshing actually invalidates the old refresh token. If it does not, enforce single-use myself in the refresh before-hook: store a per-account refresh counter/family and reject anything stale. Rotation is a claim about behavior; claims get tested.
- [ ] Client schedule: refresh session token every 10 min, refresh token every 4 h 50 min.
- [ ] Tokens travel ONLY over TLS.
- [ ] Tokens are NEVER written to client-side logs. Unity Debug.Log output lands in Player.log, and Player.log lands in pastebins. Audit for this before every release.
- [ ] Login rate limit, keyed PER IP (pre-auth there is no account yet, and device ID is client-supplied = attacker-controlled): more than 5 login requests in 3 s -> limited.
- [ ] First rate-limit layer in nginx (`limit_req`) before requests even hit Nakama; Nakama-side counting as the second layer.
- [ ] The login rate limit now also protects the OUTBOUND side: every login attempt costs one graph.oculus.com validation call, so unthrottled login spam would make ME hammer Meta's API with my own app credentials.
- [ ] Docs: Nakama session configuration: https://heroiclabs.com/docs/nakama/getting-started/configuration/ (session section). nginx limit_req: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html

## Phase 7: Enforcement ops (Discord bot + logging)

- [ ] Bans/mutes are done through the Discord bot, not the Nakama dashboard.
- [ ] Every enforcement RPC guarded by its own secret (ban secret, mute secret, ...), generated with OpenSSL (`openssl rand -base64 32`), known only to the bot and Nakama. Rotatable without downtime (support two valid secrets during rotation).
- [ ] Mutes: RPC flow mirrors bans, but enforcement happens at the voice relay (Phase 11): the server stops forwarding the muted player's voice packets. Never rely on clients politely muting themselves.
- [ ] Bot commands take a PUBLIC reason (the user will read it on their account page, Phase 9) and an optional INTERNAL note (never shown). Mods write the public reason knowing the person will read it.
- [ ] The ban command gets a device flag (or a separate command) for the severe tier. The bot only tells Nakama; Nakama makes the Meta call. The bot never holds Meta credentials.
- [ ] Bot commands gated behind a Discord role. The role sits above everything else in the hierarchy so other bots/users cannot change its members.
- [ ] Ban rights: me and possibly other high-trust individuals, all with 2FA (authenticator app) enabled.
- [ ] The bot's host machine is now security-critical infrastructure. Whoever owns that box owns ban power over the game. Patch and harden it like the VPS.
- [ ] The AUTHORITATIVE enforcement log is the per-account records in Nakama storage (Phase 5): complete by construction, since every enforcement RPC writes there, including actions from the fallback tool.
- [ ] The bot ALSO writes each action to its own file: when, why, who, how long. This file is not the record, it is an independent WITNESS on a different machine. Cross-checking it against the Nakama records detects (a) actions someone performed by calling Nakama directly with a stolen secret, bypassing the bot, and (b) tampering on either side. This is what the incident playbook leans on.
- [ ] The witness file is append-only (`chattr +a`) or shipped elsewhere, and kept as a ROLLING window: rotate and delete after ~6 months. Its job is detecting recent tampering, not preserving history. The bounded window also resolves the append-only vs GDPR-erasure conflict: old lines age out on their own instead of needing per-account deletion from a flat file.
- [ ] Fallback tool for when the bot is down: my current plan is a local HTML file that never leaves my PC. Caveat, decided knowingly: the enforcement secrets sit in that file in plaintext, and browser CORS may fight the RPC calls. Alternative if it annoys me: a 20-line CLI script reading the secret from a chmod 600 file, same capability, less residue.
- [ ] Docs: discord.js guide: https://discordjs.guide and API reference: https://discord.js.org . Full implementation walkthrough: enforcement-bot-guide.md.

## Phase 8: Community servers

- [ ] The Nakama endpoint (host, port, server key) becomes a client-side configurable value; official is the default.
- [ ] Completely disconnected from the official system. Community servers have no power over official servers.
- [ ] NOTHING stored or earned on a community backend ever flows into official state. Storage is namespaced per backend so a malicious community Nakama can't poison anything official.
- [ ] Community-mode sessions are VISIBLY distinct in the client UI. Warning before first connect to any community backend: "your data is not in my hands." Encourage device-auth-only there; never shared credentials.
- [ ] The warning also covers VOICE: on community servers, voice audio relays through a third party's machine. People should know their microphone stream leaves my infrastructure.
- [ ] Community backends CANNOT verify Meta identity: nonce validation requires my app secret, which they don't have and never will. Identity on community servers is self-asserted. One more reason the warning exists, and one more reason nothing from community backends touches official state.
- [ ] Malicious servers: client-side IP/domain blacklist, re-fetched on every launch and every community-server connection (not on a timer; connect time is the only moment it matters).
- [ ] Honest framing: the blacklist is a seatbelt for honest users, not a security control. Anyone malicious patches it out. "If you want to remove the blacklist, continue at your own risk."

## Phase 9: Legal & GDPR (before launch, not after)

- [ ] Account-management page: request account deletion.
- [ ] Account-management page also shows the account's OWN enforcement history: when, what (ban/mute), duration, public reason. Served by an RPC that reads only ctx.userId's records: the user ID comes from the SESSION, never from a request parameter, so nobody can read anyone else's history.
- [ ] Never exposed in that view: the acting moderator, internal notes, anything about other accounts.
- [ ] The history view is the natural home for the appeal path: show contact@tmtime.dev right next to it.
- [ ] Deletion x bans: the account ID is the Meta org-scoped ID, so deleting an account and re-authing resurrects the SAME identity. A ban must therefore SURVIVE account deletion: on deletion, erase profile and personal data but retain a minimal ban record (platform ID, ban status, expiry) under legitimate interest, and say so in the privacy policy. Otherwise deletion IS the ban evasion. Confirm this construction during the legal reading. Device-ban records (ban_id + reason) survive deletion under the same logic: otherwise deleting the account would orphan an active device ban with no way to reverse it on appeal.
- [ ] Data EXPORT method: this is an obligation (GDPR Art. 15/20 access + portability), not an "if required by law."
- [ ] Privacy policy: what is collected, why, and retention period for each category.
- [ ] Voice chat is processing of personal data even without recording: mention the voice relay in the privacy policy, stating that voice is encrypted in transit and relayed through my server (not end-to-end): honest wording matters.
- [ ] Identity provider is Meta: say so in the privacy policy (what I receive: the org-scoped ID; what I send Meta at every login: that ID + nonce for validation).
- [ ] Attestation in the privacy policy too: at login a Meta-signed device/app integrity token is verified; claims are processed transiently and not stored, except the last-seen device pseudonym (max 30 days, then stale by rotation) and, for device-banned devices, the ban_id + reason for the duration of the ban.
- [ ] Voice RECORDING (e.g. report clips): default is NO recording, anywhere. If I ever want report clips, that is a stop-and-get-real-legal-advice moment first: in Germany, recording the non-public spoken word without consent is a criminal offence (§ 201 StGB), so any consent design would have to be airtight.
- [ ] The enforcement log is itself personal data: define its retention period too (see Part 3 decision).
- [ ] Moderation flows through Discord (a US company): mention this data flow in the privacy policy.
- [ ] Impressum on everything public-facing (German operator, not optional).
- [ ] Legal hold RPC (gated like bans): can block an account's deletion when there's a clear legal reason. Every hold is LOGGED (who placed it, why, when) and gets a REVIEW/EXPIRY date. GDPR allows refusing erasure for legal claims per-case, not as a general veto; an indefinite unlogged hold is a liability, not a safeguard.
- [ ] ToS + NUMBERED code of conduct (R1..Rn) published on tmtime.dev (/terms, /conduct), accepted in-app with {tos_version, accepted_at} stored per account, re-prompt on version bump. The acceptance record is what every ban contractually stands on.
- [ ] Enforcement public reasons cite rule numbers ("R1: harassment in voice"). Account page adds: "Decisions are made by humans; no automated decision-making is used." Together with when/what/duration/appeal, that is the DSA Art. 17 statement-of-reasons shape.
- [ ] The enforcement ladder is published with the CoC (warn -> 10m -> 1h -> 24h -> 7d -> 30d -> permanent -> device), plus "severe violations skip steps."
- [ ] AVV signed with netcup (customer panel), filed. Art. 30 Verzeichnis der Verarbeitungstätigkeiten kept as one private table; the trust model doubles as the TOMs reference.
- [ ] Export respects rights of others: the user's own data + reports they FILED. Never reports about them, reporter identities, internal notes, or moderator identities.
- [ ] One consolidated retention table in the privacy policy (three new decisions in Part 3).
- [ ] Audience skews young: core service runs on contract necessity (Art. 6(1)(b)), not consent (German consent age is 16). Meta's platform age gates + the IARC store rating do the age work; NO own age checks, no birthday collection. Plain-language summary boxes on the privacy policy and CoC.
- [ ] Docs: GDPR articles quick reference: https://gdpr-info.eu . DSA full text: https://eur-lex.europa.eu/eli/reg/2022/2065/oj . Berlin DPA (also where breaches get reported): https://www.datenschutz-berlin.de . § 201 StGB: https://www.gesetze-im-internet.de/stgb/__201.html . DDG (Impressum, § 5): https://www.gesetze-im-internet.de/ddg/ . Detailed walkthrough: compliance-transparency-guide.md.
- [ ] Not legal advice to myself: these are the floor. Read up / brief consult before launch.

## Phase 10: Secrets & incident response

- [ ] Fill the secrets inventory (Part 3) and store it somewhere safe, NOT in the repo.
- [ ] Rotation drill once: actually rotate one secret end-to-end to prove the procedure works while calm.
- [ ] Incident playbook, three sentences each, written BEFORE anything happens:
  - **http_key leaks:** rotate it in Nakama config, redeploy the key to all game servers, audit the registry for entries I didn't provision, check tripwire logs.
  - **Bot host compromised:** revoke the Discord bot token, rotate ALL enforcement secrets, cross-check bot-side log against Nakama-side log for actions I didn't take, review recent bans.
  - **A game server compromised:** deprovision its server_id (it drops off the registry), rotate the kick secret, and replace its transport keypair (manually update the frozen fingerprint). Since the Phase 2 ticket design, game servers hold NO session encryption key: no global session invalidation needed, the blast radius stays local to that box. Inspect what else lived on it.
  - **Session encryption key leaks:** worst case. Rotate immediately; all sessions die; everyone re-authenticates. That's the design working, not failing.
- [ ] Breach discipline (GDPR Art. 33/34): every incident gets a WRITTEN personal-data assessment, even when the conclusion is "none affected" (document why). Personal data + risk -> notify the Berlin DPA (Berliner Beauftragte für Datenschutz und Informationsfreiheit) within 72 h of awareness. HIGH risk -> also inform the affected users directly. Assess first, in writing, always.
- [ ] Backups: encrypted Postgres dumps of Nakama's DB, stored OFF the VPS (the Pi is right there), 30-day rotation (Part 3 decision), restore tested ONCE before it matters. Deleted data ages out of backups within the rotation window: say so in the privacy policy. Transfer design DECIDED: APPEND-ONLY. The VPS holds credentials that can ADD dumps and nothing else: no delete, no overwrite. Consequence: the 30-day pruning runs ON THE PI (a local cron the VPS cannot touch), not on the VPS. After any restore, re-apply deletions performed since that dump was taken. (pg_dump: https://www.postgresql.org/docs/current/app-pgdump.html)

## Phase 11: Voice chat (Dissonance)

Can be started any time after Phase 3 (needs auth + rooms). The mute items need Phases 5/7 in place. Deliberately last in the plan: voice is a feature, everything before it is the spine.

- [ ] Get the pieces: Dissonance Voice Chat (paid asset) plus the free "Dissonance For Mirror Networking" integration from the Asset Store. Docs: placeholder-software.co.uk/dissonance/docs (Mirror quickstart, plus the rooms/channels pages). Issue tracker and demo scenes: github.com/Placeholder-Software/Dissonance.
- [ ] The property everything below depends on: with the Mirror integration, voice packets ride the SAME Mirror connection and are RELAYED THROUGH MY DEDICATED SERVER. Consequences: (a) the server can authoritatively control who hears whom, (b) a kick/ban that drops the Mirror connection kills voice automatically, zero extra work, (c) the VPS pays the bandwidth: roughly 10-40 kbps up per speaking player (Opus, depends on quality setting), multiplied by listeners on the way down. Do the capacity math per room size before picking codec quality.
- [ ] Match isolation: Dissonance's rooms/channels are ITS OWN routing concept, separate from Mirror's NetworkMatch interest management. Scope voice room names per match (room name = match ID) or players in different matches may hear each other. Verify against the docs, then verify in practice.
- [ ] No voice pre-auth: Dissonance components only start after NetworkAuthenticator succeeds. An unauthenticated connection must not be able to push a single voice packet.
- [ ] Treat voice packets like any client message: sender identity comes from the CONNECTION (verified at auth), never from packet contents. Size-cap and rate-cap voice messages server-side so a modified client can't flood the relay.
- [ ] Voice inherits the Phase 2 transport encryption: audio is no longer plaintext on the wire. Honest limits, written down: this is encryption IN TRANSIT, not end-to-end. The relay necessarily sees cleartext frames to route them, and that is precisely what makes server-side muting and moderation possible. Deliberate trade, stated as such in the privacy policy.
- [ ] Server-side mute, enforced at the relay: the server stops forwarding a muted player's packets. A modified client keeps transmitting and nobody hears it, which is the point. Find the right hook in Dissonance's server-side API for packet filtering (check the docs for access control / routing hooks) and wire it to the mute list, same check pattern as the ban poll.
- [ ] Client defaults that respect privacy: an always-visible mic indicator, an easy hard mute, and a conscious choice between push-to-talk and voice activation. Open mic in VR is a privacy footgun: room audio, background family conversations.
- [ ] VR specifics: positional/proximity audio (Dissonance supports positional playback), and test echo cancellation on Quest hardware early. Speaker-into-mic feedback ruins everything.
- [ ] Exit test: two matches running, match A cannot hear match B under any client modification I can produce. Server-mute a player: inaudible to everyone even while their modified client keeps sending. Kick a speaking player: voice stops the instant the connection drops.

## Phase 12: User safety tools (report / block / mute)

Full guide: compliance-transparency-guide.md, Part 2. Needs Phases 5/7 (mod pipeline) for the report queue and Phase 11 (voice) for local mute. Why it exists: Meta's moderation/reporting expectations for social apps + the DSA notice-and-action duty (Art. 16). Enforcement stops being top-down only.

- [ ] Local block/mute: client-side playback control, list in Nakama storage (collection `blocks`, owner-only read/write), size-capped, ID format validated on write. The ONE place client-side enforcement is correct: A not hearing B only affects A.
- [ ] report_user RPC: USER-callable (ctx.userId REQUIRED, mirror image of register_server's guard), category from a fixed allowlist including child safety, text capped, rate limited per reporter (5 / 10 min, 20 / day), stored in a no-public-read `reports` collection with context (server, match, Nakama-clock timestamp).
- [ ] Mod notification: Nakama webhook into a private #reports channel, CATEGORY + IDs only, report text on demand via the bot (data minimization). Child safety gets a [PRIORITY] prefix and is always reviewed first.
- [ ] Bot additions: /reports, /report_view, /dismiss; every enforcement command takes an optional report_id that resolves and links the report.
- [ ] Reporter feedback at next login: "your report was reviewed." Never any detail about the target.
- [ ] Severe-cases runbook written (act, legal hold, preserve, authorities): guide Part 2.4.
- [ ] Exit test: full loop from in-game report to linked enforcement record works; spam capped; unauthenticated and self-reports rejected.

---

# PART 3: REFERENCE

## Constants (one place, so future-me doesn't reinvent numbers)

| Constant | Value | Why |
|---|---|---|
| Heartbeat interval | 10 s | Cheap, fast registry convergence |
| STALE_AFTER | 25 s | 2 missed beats + 5 s margin (was 20 s in old notes; 20 has zero margin) |
| Ban poll interval | 10 s | Guarantees <= 15 s ban worst case with margin; don't crank lower, push does the real work |
| Kick call timeout | 2-3 s | One dead server can't stall a ban |
| Session token TTL | 15 min | Stolen token ceiling |
| Refresh token TTL | 5 h | With rotation, real exposure ~10 min |
| Client session refresh | every 10 min | Before the 15 min expiry |
| Client refresh-token refresh | every 4 h 50 min | Before the 5 h expiry |
| Login rate limit | > 5 requests / 3 s per IP | Pre-auth, so per-IP is the only honest key |
| Attestation challenge_nonce TTL | ~2 min, single use | Nakama-generated in login round trip 1; deleted on first verification |
| Join ticket TTL | ~30 s, single use, one active per user | Bound to (user, server); atomic mark-used prevents replay races |

## Secrets inventory

| Secret | Lives where | Known by | If it leaks |
|---|---|---|---|
| server_key | Client build + Nakama | Everyone (public by design) | Nothing, it's public |
| http_key | Nakama config + game servers | Me, game servers | Rotate, redeploy, audit registry, check tripwire |
| Meta app access token (app ID + app secret) | Nakama runtime config only | Me | Now carries BAN POWER (attestation verify + device_ban endpoints). Rotate in the Meta dev dashboard; logins fail until redeployed, sessions keep working. After a leak, audit active bans via the device_ban_ids endpoint |
| Kick secret | Nakama runtime + game servers | Me, both sides of that edge | Rotate both sides |
| Enforcement secrets (ban, mute, legal hold, reports) | Nakama runtime + Discord bot | Me, bot host | Rotate, audit both enforcement logs |
| Session encryption key | Nakama config ONLY (ticket design: never on game servers) | Me | Rotate = all sessions invalidated, everyone re-logs |
| Game server transport keypairs | One per game server, on that box; public fingerprint frozen in the registry | Me; each server holds only its own | Replace the keypair, manually update the frozen fingerprint, clients pin the new one via list_servers |
| Console password | Nakama | Me | Change, review console audit log |
| Postgres password | Compose env | Me | Rotate, check DB for tampering |
| Discord bot token | Bot host | Me | Revoke in Discord dev portal, re-issue |
| Discord reports webhook URL | Nakama runtime config | Me | Delete + recreate the webhook in channel settings. A leak only lets someone post fake reports into #reports: annoying, not powerful |
| TLS private keys | Host, certbot-managed | Certbot/me | Revoke + reissue cert |
| Backup encryption key | Password manager / offline, NOT on the VPS | Me | Leak: backups readable, rotate + re-encrypt future dumps. LOSS is the real danger: backups become garbage, so store it in two places |

## Open decisions (resolve in Phase 0)

- [x] **What does a ban attach to: DECIDED.** Base: the Meta org-scoped account (survives reinstalls; evasion needs a new Meta account). Escalation for severe/repeat cases: a device ban via the Attestation API (survives new accounts; evasion needs new hardware). Together, evading a full ban costs a fresh Meta account AND a fresh headset. The device identifier is Meta's app-scoped, 30-day-rotating unique_id, nothing I fingerprint myself. No IP bans, no custom hardware fingerprinting, ever.
- [x] **Appeal contact: DECIDED, contact@tmtime.dev.** Shown next to the enforcement history (Phase 9), and it's also where GDPR requests land. That mailbox rides on my own poste.io, so mail server uptime is now compliance-relevant, not just convenience.
- [x] **app_integrity_state policy: DECIDED, Horizon Store only.** Require StoreRecognized at login. Consequences: betas go through Horizon release channels (alpha/beta), which still count as store-installed; any sideloaded build fails login, including my own, so dev builds either use a release channel or get in via an allowlist of MY verified org-scoped ID (allowlist by verified account, never by anything the client claims). Cert pinning (package_cert_sha256_digest) stays on either way. Future SteamVR port, if it ever happens: that is a second identity path (Steam auth session tickets, stored as `steam:<id>`, the namespacing already allows it) and NO attestation there, the API is Quest-only, so no device bans and no integrity claims on PC: server authority carries all the weight on that platform. Decide nothing else about it until it's real.
- [x] **Report retention: DECIDED, reporter's account lifetime ("permanent" from the reporter's side).** Purpose: the account-page Reports section (Phase 12) plus abuse-pattern tracking. Boundaries that make it defensible: rows are erased with the REPORTER's account deletion (they were always the reporter's data); when a TARGET deletes their account, their platform ID is cleared from reports about them, while the display-name snapshot, category, and status stay (that is the reporter's own history of their own action); and the free text is moderation evidence, prunable N months after resolution without breaking the feature if I ever want to shrink the weak point. Known weak point if a regulator squints: long-lived free text of DISMISSED reports, i.e. unfounded accusations kept in storage. Goes in the retention table.
- [x] **Infrastructure log retention: DECIDED, 14 days (nginx, mail).** IPs are personal data. Pairs with the Phase 1 rule that query strings are redacted from access logs, so what little the logs hold is short-lived AND secret-free.
- [x] **Backup retention: DECIDED, 30 days, append-only.** Encrypted dumps, off-box on the Pi; the VPS can only add, never delete or overwrite; the Pi prunes to 30 days locally (Phase 10). Short retention keeps the GDPR story simple: deleted data ages out within one cycle.
- [x] **Mirror auth transport: DECIDED, BOTH: join tickets + encrypted transport with pinned keys.** The either/or was a false choice. Tickets protect the session token and delete game servers' token-minting power, but only transport encryption protects VOICE and gameplay traffic, which are personal data crossing the internet (Art. 32 territory). Tickets without encryption leak audio; encryption without tickets keeps the symmetric session key on every game server. Both together: token never on the game wire, session key exists only on Nakama, voice and gameplay encrypted, and a ticket can't be sniffed in the first place. Implemented in Phase 2 (rewritten), with the key fingerprints riding the registry (4.2/4.5).
- [x] **Enforcement record retention: DECIDED, account lifetime.** Records live until account deletion, then get erased with the account (except the minimal ban record from Phase 9). Stated purpose in the privacy policy: transparency to the user + escalation for repeat offenses. Known weak point if a regulator ever squints: very old minor records (ancient short mutes) are hardest to defend as "still necessary." The bot-side witness file has its OWN retention: rolling ~6 month window (see Phase 7).
- [x] **Registry storage vs in-memory: DECIDED, Nakama storage.** Boring, safe, survives restarts. (In-memory would self-clean on restart but dies with multi-node.)
- [x] **Voice recording for moderation reports: DECIDED, NO.** No recording or storing of voice, anywhere, by anyone including the server. Revisit only ever after real legal advice (§ 201 StGB).
