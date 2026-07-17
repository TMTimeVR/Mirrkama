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
Actual malicious servers? IP/Domain blacklist. MrBeastEvent.com (malicious): in a client-side blacklist.

If you want to remove the blacklist, continue at your own risk.

### Overall enforcement?

Bans need to take effect instantly.

### Discord bot for enforcement?

Yes.

Bans will not be done with the dashboard, but with a discord bot.
All enforement RPCs are guarded with a secret (mute secret, ban secret, etc) that only the Discord bot and Nakama know.
The Discord bot also guards the commands behind roles. The role must be above everything else in the hirarchy, so other bots and users can not change the role members.
Banning will only be available to me and possibly other high-trust individuals with 2FA (authenticator app) enabled.

Also code a HTML file that stays on your PC, never goes out in the internet. That will be used as a replacement dashboard for banning when the bot goes down.
