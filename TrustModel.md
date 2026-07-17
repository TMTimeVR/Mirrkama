(Notes for myself)
# Who trusts who?
Client trusts Nakama, Mirror trusts Nakama. Nakama decides who's allowed to join a server.

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
