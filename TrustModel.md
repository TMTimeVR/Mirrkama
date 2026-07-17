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

Method 3: MetworkAuthenticator
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
