(Notes for myself)

# PART 1: RULES & MENTAL MODEL

Read this part before writing any code. Every design question later gets answered by these rules.

## Who trusts who

- Client trusts Nakama. Game servers (Mirror) trust Nakama. Nakama decides who is allowed to join a server.
- Identity starts at the platform: Nakama trusts META (nonce validation against graph.oculus.com) to vouch that a claimed org-scoped ID really belongs to this client. A platform ID without a validated nonce is worth nothing.
- Meta also vouches for DEVICE and APP integrity (Attestation API): Nakama generates a challenge nonce, the headset returns a Meta-signed attestation token, Nakama verifies it server-side. Client-side validation of any of this would be worthless, a modified client just skips it.
- Dedicated servers verify Nakama session tokens themselves (JWT signature + expiry). The token IS the identity: nothing else the client sends about who it is counts.
- Local JWT verification answers "is this token real", NOT "is this account in good standing". Good standing (ban status) is always asked from Nakama.
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
Client ──RPC list_servers (session token)──> Nakama ──reads registry,
        filters stale/full──> returns [address, port, region, players] ──> Client
Client ──Mirror connect──> chosen game server ──NetworkAuthenticator verifies JWT
        + asks Nakama "banned?"──> allowed in (or not)

BAN HAPPENS
Discord bot ──RPC ban (ban secret)──> Nakama:
   1. mark banned in storage + append enforcement record
   2. sessionLogout
   3. read registry ──> POST kick to every live server's kick endpoint (kick secret)
   4. severe tier only: ──device_ban(last-seen unique_id)──> graph.oculus.com,
      store the returned ban_id in the enforcement record
```

## Mental model to keep

The registry is not "configuration": it is a live, self-rebuilding picture of the fleet that expires on its own. I never trust it to be complete (poll enforcement covers gaps) and I never trust its writers about anything I can determine myself (time, source IP). Clients only ever see a projection of it, never the thing itself.

---

# PART 2: BUILD PLAN

Work through the phases in order. Each phase ends with an exit test: don't move on until it passes.

## Phase 0: This document

- [ ] Resolve the open decisions in Part 3 (ban attachment, retention periods).
- [ ] Re-read Part 1 once before every phase. Cheap, and it keeps the rules loaded.

## Phase 1: Nakama standalone + hardening

- [ ] Docker compose: Nakama + Postgres on the VPS.
- [ ] BEFORE anything is internet-facing: change the default server_key, http_key, and console password. Nakama defaults are publicly known and scanners look for them.
- [ ] Console (port 7351) is NOT publicly exposed. Behind nginx with an allowlist, or not exposed at all (SSH tunnel when I need it).
- [ ] TLS termination for the client-facing API via certbot on the host nginx (same pattern as mail.tmtime.dev).
- [ ] Unity client authenticates via the Meta platform flow and receives a session token:
  1. Client asks the Platform SDK for a user proof: `Users.GetUserProof()` returns a NONCE (single-use, short-lived; fetch a fresh one for every login attempt, never cache).
  2. Client sends org-scoped ID + nonce to Nakama (custom authentication, over TLS).
  3. Nakama, in a beforeAuthenticateCustom hook, calls Meta's validation endpoint (graph.oculus.com/user_nonce_validate) with the nonce, the claimed user ID, and MY app access token (app ID + app secret). Meta answers valid/invalid.
  4. Invalid, expired, or reused nonce -> reject login. NEVER create or log into an account from an unvalidated platform ID, or anyone can impersonate anyone by claiming an ID.
- [ ] The Meta app access token lives ONLY in Nakama runtime config. Never in the client build, never on game servers (they verify session JWTs, they don't do platform auth).
- [ ] Namespace the account ID: store it as `meta:<org-scoped-id>`, not the bare number, so a future Steam/PICO port can't collide with Meta IDs.
- [ ] Fail closed: if graph.oculus.com is unreachable, logins fail. Existing sessions keep working (tokens already issued), same philosophy as the Nakama-outage rule in 4.6.
- [ ] Docs: Meta Horizon platform docs, "User Verification" page (Unity Platform SDK), plus Nakama server framework docs on authentication before-hooks.
- [ ] Attestation in the SAME login flow (Attestation API; Quest 2/Pro/3/3S only, so minimum supported device is Quest 2):
  1. Login becomes two round trips. Client first asks Nakama for a login challenge; Nakama generates a crypto-random Base64URL challenge_nonce (single use, short TTL, stored server-side).
  2. Client calls `DeviceApplicationIntegrity.GetIntegrityToken(challenge_nonce)` alongside `GetUserProof()`, then sends org-scoped ID + user-proof nonce + attestation token to authenticateCustom.
  3. The two nonces run in OPPOSITE directions, don't mix them up: the user-proof nonce is CLIENT-generated and Nakama validates it with Meta; the attestation challenge_nonce is NAKAMA-generated and must come back inside the Meta-signed token claims.
  4. Nakama verifies the attestation token at graph.oculus.com/platform_integrity/verify (same OC access token), then checks the claims itself: nonce matches the stored challenge (then delete the challenge, single use), timestamp is fresh, package_id and package_cert_sha256_digest match MY build (cert pinning catches repackaged APKs regardless of install source), device_integrity_state is Advanced or Basic (NotTrusted -> reject), and the device_ban section: is_banned -> reject (can surface remaining_ban_time in the rejection message).
  5. Fail closed, per Meta's own best practice: persistent attestation errors are treated the same as NotTrusted. Rate limits (100/hour, 200/day per device) are irrelevant at one token per login.
- [ ] Attestation claims are checked and DISCARDED, no routine device tracking. The only thing persisted: the last-seen unique_id per account, kept at most 30 days (it rotates and goes stale by itself), so a device ban can still be issued shortly after an incident.
- [ ] app_integrity_state (StoreRecognized) is a DISTRIBUTION decision, see Part 3: enforcing it blocks every sideloaded build, including my own betas. Cert pinning above stays on either way.
- [ ] Token lifetimes in Nakama config: session 15 min, refresh 5 h.
- [ ] Exit test: authenticate from Unity over TLS with a fresh nonce. Replay the SAME nonce a second time -> rejected. Claim someone else's org-scoped ID with a garbage nonce -> rejected. Restart Nakama, authenticate again. Attestation: replay an old challenge_nonce -> rejected. Garbage attestation token -> rejected. Confirm my own DEV-MODE headset still passes device integrity (otherwise I just locked myself out of my own game). Device-ban my own test device, confirm login is refused, then REVERSE it via the ban_id.

## Phase 2: The auth spine (custom NetworkAuthenticator)

- [ ] Client sends its Nakama session token inside the Mirror auth message.
- [ ] Dedicated server verifies the JWT signature locally using the session encryption key, checks expiry (exp), and takes the user ID FROM THE TOKEN, never from any other field.
- [ ] The session encryption key lives ONLY on Nakama and the dedicated servers. Never in the client build.
- [ ] Pre-auth phase is hostile territory: size-limit the auth message, short timeout for unauthenticated connections, process NO other message type before auth succeeds.
- [ ] Join-time ban check: after JWT verification, one RPC to Nakama asking "is this user banned?" (a valid token can be 10 minutes old; local verification can't know about a fresh ban).
- [ ] Exit test: write a deliberately evil client. Garbage payloads, oversized payloads, expired tokens, valid-looking-but-unsigned tokens, messages before auth. All must be rejected cleanly with no gameplay code reached.

## Phase 3: Rooms + authority discipline

- [ ] Mirror Multiple Matches pattern with NetworkMatch: one server process, isolated rooms.
- [ ] Every [Command] validates every argument: bounds, rate, and ownership (does this connection actually own the object it's commanding?).
- [ ] Authoritative state lives server-side only. Client input is a request, never a result.
- [ ] Logging discipline from day one: log user IDs and violation events. Do not hoard IPs tied to behavior; IPs are personal data (GDPR), define retention before collecting.
- [ ] Exit test: modified client tries to command objects it doesn't own, teleport, and spam. Server rejects all, logs the violations, and the room stays consistent for other players.

## Phase 4: Fleet: registration, heartbeat, list_servers

### Step 4.1: Provision the ID list (5 min, one-off)

- [ ] Storage collection `server_registry`, one object per allowed server: key = `mirror1`, `mirror2`, ... Value starts as `{ "provisioned": true }`.
- [ ] Permissions: NO public read, NO public write. Only runtime code touches it.
- [ ] Rule this encodes: a registration for an ID not in this list is REJECTED. An attacker with a leaked http_key still needs a valid ID.

### Step 4.2: The register_server RPC (guard first, logic second)

- [ ] Line 1: `if (ctx.userId) -> throw generic error + log "USER SESSION TRIED register_server"`. This log line is a TRIPWIRE. If it ever fires, someone is attacking exactly the hole I predicted. Never delete this log.
- [ ] Parse payload: server_id, port, kick_port, region, current_players, max_players. Validate types and ranges strictly (port 1-65535, players >= 0, region from a fixed allowlist). Reject anything malformed, no "fixing up" input.
- [ ] Look up server_id in `server_registry`. Not provisioned -> reject + log.
- [ ] FIRST registration (no IP stored yet): store the IP. Also record the observed source IP of the HTTP call and log if it differs from the claimed one (misconfig or attack, either way I want to know).
- [ ] SUBSEQUENT calls: IGNORE any address/port in the payload. The address is frozen. Changing it = me editing storage manually. Deliberate trade-off: costs me manual work, but a leaked key can't re-point mirror1 at an attacker's machine.
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
- [ ] Read all entries -> drop stale -> drop full (current >= max) -> return ONLY: address, port, region, current_players, max_players.
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

- [ ] Kick endpoint on each game server: small HTTP listener, separate from the game port, accepts one command: "disconnect user X now." Gated with the kick secret (NOT http_key: different trust edge, different secret). Not publicly reachable if topology allows. Payload validated strictly (user ID format, nothing else).
- [ ] Ban RPC in Nakama (called by the Discord bot with the ban secret):
  1. Mark account banned in storage (the authoritative act; everything else is propagation), and append a per-account ENFORCEMENT RECORD: timestamp, action type (ban/mute), duration, public reason, acting moderator, internal note. Collection has NO public read; the user-facing view is an RPC projection (Phase 9).
  2. Invalidate the Nakama session server-side (sessionLogout: kills socket + refresh path for that session).
  3. Read live servers via the SAME `getLiveServers()` function as list_servers (shared function so the two can never drift), POST the kick to every server's kick endpoint. Parallel, 2-3 s timeout per server so one dead server can't stall the ban. Push failure is logged but does NOT fail the ban; poll and join-check mop up.
- [ ] Poll fallback: each game server, every 10 s, sends Nakama the list of ITS currently connected user IDs and asks "any banned?" Server disconnects hits. Servers never hold a copy of the full ban list (smaller request, better for privacy).
- [ ] Door checks: Nakama refuses LOGIN for banned accounts AND refuses TOKEN REFRESH for banned accounts. Without the refresh check, a banned user who dodged the kick keeps minting fresh session tokens for up to 5 h.
- [ ] Join-time ban check already built in Phase 2.
- [ ] Device-ban escalation for SEVERE or repeat cases (Attestation API device ban; needs one-time Data Use Checkup approval in the Meta dashboard before it works):
  1. Nakama calls graph.oculus.com/platform_integrity/device_ban server-to-server (same OC access token) with the offender's last-seen unique_id and a duration.
  2. Store the returned ban_id in the enforcement record. The unique_id rotates every 30 days; the ban_id is the DURABLE handle for updating or reversing the ban. Lose it and the device has to show up again before I can touch the ban.
  3. Enforcement stays on MY side: Meta only carries the flag. My login check (device_ban claim in the attestation token) is what actually refuses service, and it fires BEFORE any account exists, so a fresh alt account on a banned headset is caught at the door.
  4. Reserved for severe/repeat abuse, every one logged with a reason: Meta's Platform Abuse Policy makes ME responsible for conscientious use.
- [ ] NO IP bans, ever. The device ban replaces that idea and beats it on every axis: it hits exactly one headset instead of a whole shared IP (CGNAT would punish innocents), it survives IP rotation, and it stores a rotating pseudonym + ban_id instead of IP addresses.
- [ ] Exit test: ban a connected test account. It gets kicked in ~1 s. Ban while its server's kick endpoint is down: kicked within 15 s by poll. Banned account cannot log in, cannot refresh, cannot join.

## Phase 6: Token rules & rate limiting

- [ ] Refresh token ROTATION: every refresh issues a new refresh token and invalidates the old one. A stolen refresh token then dies the next time the legitimate client refreshes (~10 min), instead of living 5 h.
- [ ] Client schedule: refresh session token every 10 min, refresh token every 4 h 50 min.
- [ ] Tokens travel ONLY over TLS.
- [ ] Tokens are NEVER written to client-side logs. Unity Debug.Log output lands in Player.log, and Player.log lands in pastebins. Audit for this before every release.
- [ ] Login rate limit, keyed PER IP (pre-auth there is no account yet, and device ID is client-supplied = attacker-controlled): more than 5 login requests in 3 s -> limited.
- [ ] First rate-limit layer in nginx (`limit_req`) before requests even hit Nakama; Nakama-side counting as the second layer.
- [ ] The login rate limit now also protects the OUTBOUND side: every login attempt costs one graph.oculus.com validation call, so unthrottled login spam would make ME hammer Meta's API with my own app credentials.

## Phase 7: Enforcement ops (Discord bot + logging)

- [ ] Bans/mutes are done through the Discord bot, not the Nakama dashboard.
- [ ] Every enforcement RPC guarded by its own secret (ban secret, mute secret, ...), generated with OpenSSL, known only to the bot and Nakama. Rotatable without downtime (support two valid secrets during rotation).
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
- [ ] The history view is the natural home for the appeal path: put the appeal contact right next to it.
- [ ] Deletion x bans: the account ID is the Meta org-scoped ID, so deleting an account and re-authing resurrects the SAME identity. A ban must therefore SURVIVE account deletion: on deletion, erase profile and personal data but retain a minimal ban record (platform ID, ban status, expiry) under legitimate interest, and say so in the privacy policy. Otherwise deletion IS the ban evasion. Confirm this construction during the legal reading. Device-ban records (ban_id + reason) survive deletion under the same logic: otherwise deleting the account would orphan an active device ban with no way to reverse it on appeal.
- [ ] Data EXPORT method: this is an obligation (GDPR Art. 15/20 access + portability), not an "if required by law."
- [ ] Privacy policy: what is collected, why, and retention period for each category.
- [ ] Voice chat is processing of personal data even without recording: mention the voice relay in the privacy policy.
- [ ] Identity provider is Meta: say so in the privacy policy (what I receive: the org-scoped ID; what I send Meta at every login: that ID + nonce for validation).
- [ ] Attestation in the privacy policy too: at login a Meta-signed device/app integrity token is verified; claims are processed transiently and not stored, except the last-seen device pseudonym (max 30 days, then stale by rotation) and, for device-banned devices, the ban_id + reason for the duration of the ban.
- [ ] Voice RECORDING (e.g. report clips): default is NO recording, anywhere. If I ever want report clips, that is a stop-and-get-real-legal-advice moment first: in Germany, recording the non-public spoken word without consent is a criminal offence (§ 201 StGB), so any consent design would have to be airtight.
- [ ] The enforcement log is itself personal data: define its retention period too (see Part 3 decision).
- [ ] Moderation flows through Discord (a US company): mention this data flow in the privacy policy.
- [ ] Impressum on everything public-facing (German operator, not optional).
- [ ] Legal hold RPC (gated like bans): can block an account's deletion when there's a clear legal reason. Every hold is LOGGED (who placed it, why, when) and gets a REVIEW/EXPIRY date. GDPR allows refusing erasure for legal claims per-case, not as a general veto; an indefinite unlogged hold is a liability, not a safeguard.
- [ ] Not legal advice to myself: these are the floor. Read up / brief consult before launch.

## Phase 10: Secrets & incident response

- [ ] Fill the secrets inventory (Part 3) and store it somewhere safe, NOT in the repo.
- [ ] Rotation drill once: actually rotate one secret end-to-end to prove the procedure works while calm.
- [ ] Incident playbook, three sentences each, written BEFORE anything happens:
  - **http_key leaks:** rotate it in Nakama config, redeploy the key to all game servers, audit the registry for entries I didn't provision, check tripwire logs.
  - **Bot host compromised:** revoke the Discord bot token, rotate ALL enforcement secrets, cross-check bot-side log against Nakama-side log for actions I didn't take, review recent bans.
  - **A game server compromised:** deprovision its server_id (it drops off the registry), rotate the kick secret, rotate the session encryption key if the box held it (this invalidates ALL live sessions: everyone re-logs, acceptable cost), inspect what else lived on that box.
  - **Session encryption key leaks:** worst case. Rotate immediately; all sessions die; everyone re-authenticates. That's the design working, not failing.

## Phase 11: Voice chat (Dissonance)

Can be started any time after Phase 3 (needs auth + rooms). The mute items need Phases 5/7 in place. Deliberately last in the plan: voice is a feature, everything before it is the spine.

- [ ] Get the pieces: Dissonance Voice Chat (paid asset) plus the free "Dissonance For Mirror Networking" integration from the Asset Store. Docs: placeholder-software.co.uk/dissonance/docs (Mirror quickstart, plus the rooms/channels pages). Issue tracker and demo scenes: github.com/Placeholder-Software/Dissonance.
- [ ] The property everything below depends on: with the Mirror integration, voice packets ride the SAME Mirror connection and are RELAYED THROUGH MY DEDICATED SERVER. Consequences: (a) the server can authoritatively control who hears whom, (b) a kick/ban that drops the Mirror connection kills voice automatically, zero extra work, (c) the VPS pays the bandwidth: roughly 10-40 kbps up per speaking player (Opus, depends on quality setting), multiplied by listeners on the way down. Do the capacity math per room size before picking codec quality.
- [ ] Match isolation: Dissonance's rooms/channels are ITS OWN routing concept, separate from Mirror's NetworkMatch interest management. Scope voice room names per match (room name = match ID) or players in different matches may hear each other. Verify against the docs, then verify in practice.
- [ ] No voice pre-auth: Dissonance components only start after NetworkAuthenticator succeeds. An unauthenticated connection must not be able to push a single voice packet.
- [ ] Treat voice packets like any client message: sender identity comes from the CONNECTION (verified at auth), never from packet contents. Size-cap and rate-cap voice messages server-side so a modified client can't flood the relay.
- [ ] Server-side mute, enforced at the relay: the server stops forwarding a muted player's packets. A modified client keeps transmitting and nobody hears it, which is the point. Find the right hook in Dissonance's server-side API for packet filtering (check the docs for access control / routing hooks) and wire it to the mute list, same check pattern as the ban poll.
- [ ] Client defaults that respect privacy: an always-visible mic indicator, an easy hard mute, and a conscious choice between push-to-talk and voice activation. Open mic in VR is a privacy footgun: room audio, background family conversations.
- [ ] VR specifics: positional/proximity audio (Dissonance supports positional playback), and test echo cancellation on Quest hardware early. Speaker-into-mic feedback ruins everything.
- [ ] Exit test: two matches running, match A cannot hear match B under any client modification I can produce. Server-mute a player: inaudible to everyone even while their modified client keeps sending. Kick a speaking player: voice stops the instant the connection drops.

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

## Secrets inventory

| Secret | Lives where | Known by | If it leaks |
|---|---|---|---|
| server_key | Client build + Nakama | Everyone (public by design) | Nothing, it's public |
| http_key | Nakama config + game servers | Me, game servers | Rotate, redeploy, audit registry, check tripwire |
| Meta app access token (app ID + app secret) | Nakama runtime config only | Me | Now carries BAN POWER (attestation verify + device_ban endpoints). Rotate in the Meta dev dashboard; logins fail until redeployed, sessions keep working. After a leak, audit active bans via the device_ban_ids endpoint |
| Kick secret | Nakama runtime + game servers | Me, both sides of that edge | Rotate both sides |
| Ban / mute secrets | Nakama runtime + Discord bot | Me, bot host | Rotate, audit both enforcement logs |
| Session encryption key | Nakama config + game servers | Me, both | Rotate = all sessions invalidated, everyone re-logs |
| Console password | Nakama | Me | Change, review console audit log |
| Postgres password | Compose env | Me | Rotate, check DB for tampering |
| Discord bot token | Bot host | Me | Revoke in Discord dev portal, re-issue |
| TLS private keys | Host, certbot-managed | Certbot/me | Revoke + reissue cert |

## Open decisions (resolve in Phase 0)

- [x] **What does a ban attach to: DECIDED.** Base: the Meta org-scoped account (survives reinstalls; evasion needs a new Meta account). Escalation for severe/repeat cases: a device ban via the Attestation API (survives new accounts; evasion needs new hardware). Together, evading a full ban costs a fresh Meta account AND a fresh headset. The device identifier is Meta's app-scoped, 30-day-rotating unique_id, nothing I fingerprint myself. No IP bans, no custom hardware fingerprinting, ever.
- [ ] **Appeal contact:** pick the address/channel shown next to the enforcement history (Phase 9).
- [ ] **app_integrity_state policy:** if distribution is Horizon Store only, require StoreRecognized at login. If I ever ship sideloaded/beta builds, those return NotRecognized for legitimate users, so the check must be per-channel or off. Cert pinning (package_cert_sha256_digest) stays on either way: it catches repackaged APKs regardless of install source.
- [x] **Enforcement record retention: DECIDED, account lifetime.** Records live until account deletion, then get erased with the account (except the minimal ban record from Phase 9). Stated purpose in the privacy policy: transparency to the user + escalation for repeat offenses. Known weak point if a regulator ever squints: very old minor records (ancient short mutes) are hardest to defend as "still necessary." The bot-side witness file has its OWN retention: rolling ~6 month window (see Phase 7).
- [x] **Registry storage vs in-memory: DECIDED, Nakama storage.** Boring, safe, survives restarts. (In-memory would self-clean on restart but dies with multi-node.)
- [x] **Voice recording for moderation reports: DECIDED, NO.** No recording or storing of voice, anywhere, by anyone including the server. Revisit only ever after real legal advice (§ 201 StGB).
