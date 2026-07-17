(Notes for myself)
# Who trusts who?
Client trusts Nakama, Mirror trusts Nakama. Nakama decides who's allowed to join a server.
Dedicated servers verify Nakama session tokens themselves (JWT signature + expiry). The token is the identity; nothing else the client sends about who it is counts.

**server_key: this is the one clients use. Public.**
**http_key: this is a completely different thing. Server-to-server RPC endpoints. Secret.**

Nakama is the only source of truth.

The client can't tell the server to do anything. It always expresses intent, never state. Validate everything.
**Always code as if the client is hacked and will try to break the system.**

## Community servers?

Completely disconnected from the official system. 
Community servers have no power over official servers.

### Enforcement on community servers?
For all community servers: Display a warning that the client's data is not in my hands.
Actual malicious servers? IP/Domain blacklist. MrBeastEvent.com (malicious): in a client-side blacklist, that re-fetches from a domain every launch and community server connection.

If you want to remove the blacklist, continue at your own risk.

# Overall enforcement?

Bans need to take effect instantly.

## How?

Method 1: Push (~1 second enforcement time)
Dedicated servers each expose a small admin endpoint (HTTP listener) that accepts a single command: "disconnect user X now." (Of course, gated with a secret)
When fired, Nakama does:
1. Mark the account banned in storage
2. Invalidate their Nakama session server-side (nk.sessionlogout)
3. Read the live server list from the registry, and fire the kick call at every registered server (nk.httpRequest in runtime, parallel)

Method 2: Poll
Each game server also polls Nakama every 10 seconds "here are all connected user IDs, are any banned?" Nakama answeres using the ban list; the server disconnects any hit.

Method 3: NetworkAuthenticator
The flow should ask Nakama if the user is banned at join time.
### Discord bot for enforcement?

Yes.

Bans will not be done with the dashboard, but with a discord bot.
All enforement RPCs are guarded with a secret (mute secret, ban secret, etc) that only the Discord bot and Nakama know. The secrets will be generated using a OpenSSL command.
The Discord bot also guards the commands behind roles. The role must be above everything else in the hirarchy, so other bots and users can not change the role members.
Banning will only be available to me and possibly other high-trust individuals with 2FA (authenticator app) enabled.

Log every enforcement action in a .txt file: When, why, who, how long.

Also code a HTML file that stays on your PC, never goes out in the internet. That will be used as a replacement dashboard for banning when the bot goes down.

### What if a token is stolen?

Session token expires in 15 minutes, refresh token expires every 5 hours. Refresh the session token every 10 minutes. Refresh the refresh token every 4 hours and 50 minutes.

# Login rate-limit:

If a client starts to send a login request in a 3 second time limit 5 times, rate limit them.

# Comply with data protection laws.

Make an account-management page where you can request the deletion of your account. Also, somehow, if required by law, create a method to request collected data.
Make a legal hold rps (gated) to hold any account from deletion if there is a clear risk that the user is doing something illegal.

# Server registration:

Game server calls /v2/rpc/register_server?http_key=... as a POST request.
**If ctx.userId is present, reject with a generic error code. And log the attempt.** 

On boot and every heartbeat, the game servers POST this payload:

server_ID ( create a list of IDs in the storage, such as mirror1, mirror2, etc. )
address + port (Only accept if it is the first registration. Ignore at subsequent calls.)
kick entpoint
region, current_players, etc (whatever matchmaking will need)

server_ID storage (no public read):
- ID
- IP
- Port
- last heartbeat

If the heartbeat is 20 seconds long, do not send it to clients through the list_servers rpc until the heartbeat is healthy again.

There is no need for servers to change their IP. The Nakama operator can do that as well manually.

---

# HOW THE REGISTRATION SYSTEM ACTUALLY WORKS (implementation guide)

## The big picture in one paragraph

There is ONE Nakama, and MANY game servers. Nakama keeps a phone book of game servers.
Game servers call Nakama every 10 seconds to say "I'm still alive, here's my player count."
Clients never see the phone book directly: they call `list_servers`, and Nakama hands them
a filtered copy containing only healthy servers. When I ban someone, Nakama reads the same
phone book to know which servers to send the kick to. That's the whole system: one
registry, three readers (heartbeat writes it, list_servers reads it for clients,
ban-push reads it for enforcement).

## The data flow, drawn out

```
BOOT / EVERY 10s
Game server ──POST register_server (http_key)──> Nakama ──writes──> registry storage

CLIENT WANTS TO PLAY
Client ──RPC list_servers (session token)──> Nakama ──reads registry,
        filters stale/full──> returns [address, port, region, players] ──> Client
Client ──Mirror connect──> chosen game server ──NetworkAuthenticator verifies JWT
        + asks Nakama "banned?"──> allowed in (or not)

BAN HAPPENS
Discord bot ──RPC ban (ban secret)──> Nakama:
   1. mark banned in storage
   2. sessionLogout
   3. read registry ──> POST kick to every live server's kick endpoint (kick secret)
```

## Coding order: follow these steps, test after each one

### Step 1: Provision the ID list (5 min, do this in the Nakama console or a one-off script)

- [ ] Create a storage collection `server_registry`, one object per allowed server:
      key = `mirror1`, `mirror2`, ... Value starts as `{ "provisioned": true }`.
- [ ] Permissions: NO public read, NO public write. Only runtime code touches it.
- [ ] Rule this encodes: a registration for an ID that is not in this list is REJECTED.
      An attacker with a leaked http_key still needs a valid ID.

### Step 2: The register_server RPC (the guard first, logic second)

Write the RPC in this exact order, because the first check is the one that matters:

- [ ] Line 1: `if (ctx.userId) -> throw generic error + nk.loggerWarn("USER SESSION TRIED register_server", userId)`.
      This log line is a TRIPWIRE. If it ever fires, someone is attacking exactly
      the hole I predicted. Never delete this log.
- [ ] Parse payload: `server_id`, `port`, `kick_port`, `region`, `current_players`, `max_players`.
      Validate types and ranges strictly (port 1-65535, players >= 0, region from a
      fixed allowlist of strings). Reject anything malformed, no "fixing up" input.
- [ ] Look up `server_id` in `server_registry`. Not provisioned -> reject + log.
- [ ] FIRST registration (no IP stored yet): store the IP. Take it from what the
      server claims, but ALSO record the observed source IP of the HTTP call and
      log if they differ (misconfig or attack, either way I want to know).
- [ ] SUBSEQUENT calls (IP already stored): IGNORE any address/port in the payload.
      The address is frozen. Changing it = me editing storage manually. This is
      deliberate: availability costs me manual work, but a leaked key can't
      re-point mirror1 at an attacker's machine.
- [ ] Every call (first or not): update `current_players`, and set
      `last_heartbeat = current Nakama server time`. NEVER accept a timestamp
      from the payload. The caller does not get to say what time it is.

Registration and heartbeat are THE SAME RPC. It's an upsert keyed on server_id.
There is no separate heartbeat endpoint to build.

- [ ] Test: call it via curl with http_key -> works. Call it as a logged-in user
      from the Unity client -> generic error + tripwire log appears. Do not skip
      this second test.

### Step 3: Staleness rule (this is what makes the list self-healing)

- [ ] Pick the constant once: `STALE_AFTER = 25 seconds` (2 missed 10s beats + margin)
      and put it in ONE place in the code.
- [ ] There is NO cleanup cron job needed. Staleness is checked lazily by the readers:
      every read of the registry (list_servers AND ban-push) filters out entries
      where `now - last_heartbeat > STALE_AFTER`.
- [ ] Consequence to remember: a server that stops heartbeating disappears from
      clients automatically AND stops receiving push-kicks (poll enforcement still
      covers it, that's why Method 2 exists).

### Step 4: The deregister path (graceful shutdown)

- [ ] Same RPC, extra payload flag: `"shutting_down": true`.
- [ ] Nakama handling: clear `last_heartbeat` (or set a `down` flag) so the entry
      is immediately treated as stale instead of lingering for 25s while players
      try to join a dying server.
- [ ] Game server side: SIGTERM handler -> stop accepting new Mirror connections ->
      call register_server with the flag -> then shut down.

### Step 5: list_servers RPC (the client-facing projection)

- [ ] User-callable (normal session token, here ctx.userId MUST be present;
      reject server-to-server calls, mirror image of Step 2's guard).
- [ ] Read all registry entries -> drop stale -> drop full (current >= max) ->
      return ONLY: address, port, region, current_players, max_players.
- [ ] NEVER return: kick_port, server_id internals, heartbeat times, anything else.
      The RPC is the filter; that's why the collection has no public read.
- [ ] Test: dump the RPC response as a hostile client would see it and check
      nothing internal leaked.

### Step 6: Wire the ban-push into the registry

- [ ] The ban RPC's step 3 ("fire kick at every registered server") uses the SAME
      read + staleness filter as list_servers. Factor it into one shared function
      (`getLiveServers()`) so the two can never drift apart.
- [ ] Kick call: POST to the stored IP + kick_port with the kick secret, payload =
      just the user ID. Parallel, with a short timeout (2-3s) per server so one
      dead server can't stall the ban.
- [ ] A push failure is logged but does NOT fail the ban: the ban is already
      authoritative in storage; poll (Method 2) and join-check (Method 3) mop up.

### Step 7: Failure drill (do these manually before calling it done)

- [ ] Kill a game server with kill -9 -> it must vanish from list_servers within ~25s.
- [ ] Stop it gracefully -> must vanish immediately (deregister flag).
- [ ] Stop Nakama for 30s -> game servers must RETRY heartbeats with backoff and
      KEEP their current players (do not dump players because the registry blinked;
      their sessions were validated at join). New joins failing during the outage
      is correct behavior. When Nakama returns, the registry rebuilds itself
      within one heartbeat cycle, verify it does.
- [ ] Register with an unprovisioned ID -> rejected + logged.
- [ ] Send a heartbeat containing a different address for an already-registered ID ->
      address change ignored, mismatch logged.

## Mental model to keep

The registry is not "configuration": it is a live, self-rebuilding picture of the
fleet that expires on its own. I never trust it to be complete (poll enforcement
covers gaps) and I never trust its writers about anything I can determine myself
(time, source IP). Clients only ever see a projection of it, never the thing itself.
