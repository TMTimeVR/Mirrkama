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
Actual malicious servers? IP/Domain blacklist. MrBeastEvent.com (malicious): in a client-side blacklist, that re-fetches from a domain every 15 seconds.

If you want to remove the blacklist, continue at your own risk.

### Overall enforcement?

Bans need to take effect instantly.

### Discord bot for enforcement?

Yes.

Bans will not be done with the dashboard, but with a discord bot.
All enforement RPCs are guarded with a secret (mute secret, ban secret, etc) that only the Discord bot and Nakama know. The secrets will be generated using a OpenSSL command.
The Discord bot also guards the commands behind roles. The role must be above everything else in the hirarchy, so other bots and users can not change the role members.
Banning will only be available to me and possibly other high-trust individuals with 2FA (authenticator app) enabled.

Log every enforcement action in a .txt file: When, why, who.

Also code a HTML file that stays on your PC, never goes out in the internet. That will be used as a replacement dashboard for banning when the bot goes down.

### What if a token is stolen?

Session token expires in 15 minutes, refresh token expires every 5 hours. Refresh the session token every 10 minutes. Refresh the refresh token every 4 hours and 50 minutes.



# Comply with data protection laws.

Make an account-management page where you can request the deletion of your account. Also, somehow, if required by law, create a method to request collected data.
