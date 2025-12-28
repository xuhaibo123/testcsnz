# CSNZ Server Configuration

## Overview

This repository contains the CSNZ (Counter-Strike: Neo Z) game server binary and configuration files. The server allows players to host and join game rooms in P2P (peer-to-peer) mode.

## Important Note About Source Code

This repository contains **only the compiled binary** (`CSNZ_Server.exe`) and configuration files. It does **not** contain the C++ source code.

For source code changes, please refer to:
- [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server) - Reference C++ source code implementation

## Recent Changes

### P2P Connection Fix

A fix for P2P connection failures has been documented. See [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md) for details.

**Key changes:**
1. Added `PublicIP` configuration option to `ServerConfig.json`
2. Documentation for required source code changes to support the new configuration

## Configuration

### ServerConfig.json

The main server configuration file located at `myGameServer/ServerConfig.json`.

#### Important Settings for P2P Connections:

```json
{
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "119.91.238.117",
  "GameRoomIP": "119.91.238.117"
}
```

- **IP**: The IP address the server binds to (use "0.0.0.0" to listen on all interfaces)
- **Port**: The TCP port for client connections
- **PublicIP**: (**New**) The public IP address that clients should use for P2P connections. This fixes the localhost (127.0.0.1) issue.
- **GameRoomIP**: The IP address advertised for game rooms (should match PublicIP)

### Setting Up Your Server

1. **Obtain your public IP address:**
   - If hosting locally, this is your router's external IP
   - If hosting on a VPS/cloud server, this is your server's public IP
   - You can find it at https://whatismyipaddress.com/ or similar services

2. **Configure ServerConfig.json:**
   ```json
   {
     "PublicIP": "YOUR.PUBLIC.IP.HERE",
     "GameRoomIP": "YOUR.PUBLIC.IP.HERE",
     ...
   }
   ```

3. **Configure port forwarding** (if behind NAT/router):
   - Forward TCP port 30002 (or your configured port)
   - Forward UDP ports 27005 and 27015 for P2P game traffic

4. **Start the server:**
   ```
   cd myGameServer
   CSNZ_Server.exe
   ```

## Directory Structure

```
myGameServer/
├── CSNZ_Server.exe          # Main server executable
├── ServerConfig.json         # Main configuration file
├── UserDatabase.db3          # User database
├── Data/                     # Game data files
├── Logs/                     # Server logs
└── *.json                    # Various game configuration files
```

## Troubleshooting P2P Connections

If clients cannot join P2P rooms:

1. **Verify PublicIP is set correctly** in ServerConfig.json
2. **Check port forwarding** on your router/firewall
3. **Ensure UDP ports 27005 and 27015 are open**
4. **Review server logs** in the `Logs/` directory
5. **Test connectivity** from an external network (not localhost)

## Known Limitations

The current binary may not support the `PublicIP` configuration field. To enable this feature:

1. The source code must be modified as documented in [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md)
2. The server must be recompiled with the changes
3. Replace `CSNZ_Server.exe` with the new binary

Alternatively, use a server binary that already includes these fixes from the reference repository.

## Support

For issues and questions:
- Check the [P2P Connection Fix documentation](P2P_CONNECTION_FIX.md)
- Refer to the [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server) repository for source code

## License

Please refer to the license of the original CSNZ server project.
