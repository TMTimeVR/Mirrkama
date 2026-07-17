(Notes for myself)
# Who trusts who?
Client trusts Nakama, Mirror trusts Nakama. Nakama decides who's allowed to join a server.

Nakama is the only source of truth.

The client can't tell the server anything. 
**Always code as if the client is hacked and will try to break the system.**

## Community servers?

Completely disconnected from the official system. 
Community servers have no power over official servers.

### Enforcement on community servers?
For all community servers: Display a warning that the client's data is not in my hands.
Actual malicious servers? IP/Domain blacklist. MrBeastEvent.com (malicious): in a client-side blacklist.

If you want to remove the blacklist, continue at your own risk.
