# Topology

Recommended default topology:

- one or more CS2 servers
- host-level updater per machine or per runtime
- optional panel service with SQLite for small installs or Redis-backed sessions for multi-instance deployments

The updater talks to systemd and SteamCMD.
The panel talks to running servers over RCON.
Provisioning assets stay static and feed both sides without becoming a runtime service themselves.
