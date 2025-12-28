# CSNZ Server Configuration

## Overview

This repository contains the CSNZ (Counter-Strike: Neo Z) game server binary and configuration files. The server allows players to host and join game rooms in P2P (peer-to-peer) mode.

## Important Note About Source Code

This repository contains **only the compiled binary** (`CSNZ_Server.exe`) and configuration files. It does **not** contain the C++ source code.

For source code changes, please refer to:
- [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server) - Reference C++ source code implementation

## Recent Changes

### NAT/Same Network Connection Fix (NEW)

A fix for clients in the same LAN/NAT unable to join each other's rooms has been documented. See [NAT_SAME_NETWORK_FIX.md](NAT_SAME_NETWORK_FIX.md) or [同局域网连接修复方案.md](同局域网连接修复方案.md) for details.

**Problem:** Two clients behind the same NAT (sharing the same public IP) cannot join each other's game rooms.

**Solution:** Server detects when clients share the same public IP and sends the host's private/local IP instead of public IP.

**Implementation:** Requires source code modification in `packetmanager.cpp`. See [NAT_FIX_CODE_CHANGES.md](NAT_FIX_CODE_CHANGES.md) for exact code changes.

### P2P Connection Fix

A fix for P2P connection failures has been documented. See [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md) for details.

**Key changes:**
1. Added `PublicIP` configuration option to `ServerConfig.json`
2. Documentation for required source code changes to support the new configuration

## Configuration

### ServerConfig.json

The main server configuration file located at `myGameServer/ServerConfig.json`.

⚠️ **IMPORTANT**: The included ServerConfig.json uses placeholder IP addresses (0.0.0.0). **The server will NOT work properly until you replace these with your actual server's public IP address.** See [ServerConfig.template.md](ServerConfig.template.md) for detailed configuration guidance.

⚠️ **Configuration Required**: Before starting the server, you MUST:
1. Replace `"PublicIP": "0.0.0.0"` with your actual public IP
2. Replace `"GameRoomIP": "0.0.0.0"` with your actual public IP (same as PublicIP)
3. Both values should be identical for proper P2P functionality

#### Important Settings for P2P Connections:

```json
{
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "YOUR_PUBLIC_IP_HERE",
  "GameRoomIP": "YOUR_PUBLIC_IP_HERE"
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

### Same LAN/Network Issue

If two clients on the **same LAN** (behind the same NAT router) cannot join each other:

1. This is a **known issue** - see [NAT_SAME_NETWORK_FIX.md](NAT_SAME_NETWORK_FIX.md)
2. The fix requires **source code modification** (not just configuration)
3. See [NAT_FIX_CODE_CHANGES.md](NAT_FIX_CODE_CHANGES.md) for implementation details
4. Alternatively, use a binary that already includes this fix

## Known Limitations

### PublicIP Configuration Support

The current binary may not support the `PublicIP` configuration field. To enable this feature:

1. The source code must be modified as documented in [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md)
2. The server must be recompiled with the changes
3. Replace `CSNZ_Server.exe` with the new binary

Alternatively, use a server binary that already includes these fixes from the reference repository.

### Same LAN/Network Connections

The current binary does not support the NAT/same network fix. Clients behind the same NAT (same public IP) cannot join each other's rooms. To enable this:

1. The source code must be modified as documented in [NAT_SAME_NETWORK_FIX.md](NAT_SAME_NETWORK_FIX.md)
2. See [NAT_FIX_CODE_CHANGES.md](NAT_FIX_CODE_CHANGES.md) for exact code changes
3. Recompile and replace the binary

**Workarounds** (without recompiling):
- Host can enable NAT hairpining/loopback on their router (if supported)
- Use a VPN to put clients on different "networks" from the router's perspective

## Support

For issues and questions:
- **Same LAN connection issues**: Check [NAT_SAME_NETWORK_FIX.md](NAT_SAME_NETWORK_FIX.md) or [同局域网连接修复方案.md](%E5%90%8C%E5%B1%80%E5%9F%9F%E7%BD%91%E8%BF%9E%E6%8E%A5%E4%BF%AE%E5%A4%8D%E6%96%B9%E6%A1%88.md)
- **General P2P issues**: Check [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md)
- **Source code**: Refer to [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server) repository

## License

Please refer to the license of the original CSNZ server project.
