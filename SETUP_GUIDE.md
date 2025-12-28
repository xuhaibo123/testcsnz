# Quick Setup Guide

## For Server Administrators

### Prerequisites
- Windows server or computer
- Public IP address or port forwarding capability
- Firewall access to configure ports

### Step 1: Get Your Public IP

Find your public IP address:
- Visit https://whatismyipaddress.com/
- Or run: `curl ifconfig.me` (Linux/Mac)
- Or run: `Invoke-WebRequest -Uri "https://ifconfig.me" | Select-Object -ExpandProperty Content` (PowerShell)

Example: `119.91.238.117`

### Step 2: Configure ServerConfig.json

1. Navigate to `myGameServer/` directory
2. Open `ServerConfig.json` in a text editor
3. Update these fields:

```json
{
  "PublicIP": "YOUR_PUBLIC_IP_HERE",
  "GameRoomIP": "YOUR_PUBLIC_IP_HERE"
}
```

Example:
```json
{
  "PublicIP": "119.91.238.117",
  "GameRoomIP": "119.91.238.117"
}
```

4. Save the file

### Step 3: Configure Firewall

#### Windows Firewall:
```powershell
# Allow TCP port 30002
New-NetFirewallRule -DisplayName "CSNZ TCP" -Direction Inbound -Protocol TCP -LocalPort 30002 -Action Allow

# Allow UDP ports 27005 and 27015
New-NetFirewallRule -DisplayName "CSNZ UDP 1" -Direction Inbound -Protocol UDP -LocalPort 27005 -Action Allow
New-NetFirewallRule -DisplayName "CSNZ UDP 2" -Direction Inbound -Protocol UDP -LocalPort 27015 -Action Allow
```

#### Linux (iptables):
```bash
# Allow TCP port 30002
sudo iptables -A INPUT -p tcp --dport 30002 -j ACCEPT

# Allow UDP ports 27005 and 27015
sudo iptables -A INPUT -p udp --dport 27005 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 27015 -j ACCEPT
```

### Step 4: Configure Router (if behind NAT)

If your server is behind a router:

1. Log in to your router's admin panel (usually http://192.168.1.1 or http://192.168.0.1)
2. Find "Port Forwarding" or "Virtual Server" section
3. Add these port forwarding rules:

| Service Name | External Port | Internal Port | Internal IP    | Protocol |
|-------------|---------------|---------------|----------------|----------|
| CSNZ TCP    | 30002         | 30002         | YOUR_SERVER_IP | TCP      |
| CSNZ UDP 1  | 27005         | 27005         | YOUR_SERVER_IP | UDP      |
| CSNZ UDP 2  | 27015         | 27015         | YOUR_SERVER_IP | UDP      |

**Note:** Replace `YOUR_SERVER_IP` with your server's local IP (e.g., 192.168.1.100)

### Step 5: Start the Server

```batch
cd myGameServer
CSNZ_Server.exe
```

The server should start and display:
```
[Server] Starting CSNZ Server...
[Server] Listening on port 30002...
```

### Step 6: Test the Server

1. **Local test:**
   - Connect from the same machine using `127.0.0.1:30002`
   
2. **External test:**
   - Connect from another network using `YOUR_PUBLIC_IP:30002`
   - Have someone try to join a P2P room you create

## For Players

### Connecting to a Server

1. Launch the CSNZ client
2. Enter server information:
   - **IP**: `SERVER_PUBLIC_IP`
   - **Port**: `30002` (or custom port)
3. Click Connect

### Joining P2P Rooms

1. Connect to the server
2. Navigate to the room list
3. Select a room and click "Join"
4. Wait for connection (should connect to host's IP, not 127.0.0.1)

## Troubleshooting

### Can't connect to server

**Check:**
1. Is the server running?
2. Is PublicIP correctly configured?
3. Are firewall ports open?
4. Is router port forwarding configured?

**Test:**
```bash
# Test TCP connection
telnet YOUR_PUBLIC_IP 30002

# Or using PowerShell
Test-NetConnection -ComputerName YOUR_PUBLIC_IP -Port 30002
```

### Can't join P2P rooms

**Check:**
1. Is PublicIP set in ServerConfig.json?
2. Are UDP ports 27005 and 27015 open?
3. Is the binary updated to support PublicIP? (see P2P_CONNECTION_FIX.md)

**Verify:**
- Check server logs for holepunch data
- Ensure logs don't show `127.0.0.1` for remote clients

### Rooms show 127.0.0.1 as host IP

**This is the bug!**

**Solutions:**
1. Ensure `PublicIP` is configured in ServerConfig.json
2. Verify you're using an updated binary that supports the fix
3. If using old binary, follow P2P_CONNECTION_FIX.md to compile updated version

## Quick Reference

### Configuration File Locations
- Main config: `myGameServer/ServerConfig.json`
- Game data: `myGameServer/Data/`
- Logs: `myGameServer/Logs/`

### Required Ports
- **30002/TCP** - Client connections
- **27005/UDP** - P2P client port
- **27015/UDP** - P2P server port

### Important Settings
```json
{
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "YOUR_PUBLIC_IP",
  "GameRoomIP": "YOUR_PUBLIC_IP",
  "MaxPlayers": 100,
  "Room": {
    "HostConnectingMethod": 1
  }
}
```

**HostConnectingMethod:**
- `1` = Direct (server relays, no P2P)
- `2` = P2P (clients connect directly)

## Advanced Configuration

### Multiple Servers

If running multiple server instances:

**Server 1:**
```json
{
  "Port": "30002",
  "PublicIP": "YOUR_IP",
  "GameRoomIP": "YOUR_IP"
}
```

**Server 2:**
```json
{
  "Port": "30003",
  "PublicIP": "YOUR_IP",
  "GameRoomIP": "YOUR_IP"
}
```

**Port Forwarding:** Forward each port (30002, 30003, etc.) to the respective server instance.

### Using Custom Ports

To use a custom port:

1. Change `Port` in ServerConfig.json
2. Update firewall rules
3. Update port forwarding rules
4. Inform players of the new port

## Getting Help

1. Check documentation:
   - [README.md](README.md) - General information
   - [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md) - Technical details
   - [CONFIG_EXAMPLES.md](CONFIG_EXAMPLES.md) - Configuration examples
   - [P2P_FIX_OVERVIEW.md](P2P_FIX_OVERVIEW.md) - Visual guide

2. Verify server status:
   - Check `myGameServer/Logs/` for error messages
   - Ensure configuration is valid JSON

3. Test network connectivity:
   - Use online port checkers
   - Test from external network

## Security Notes

- Change default admin passwords
- Limit `MaxRegistrationsPerIP` to prevent abuse
- Monitor logs for suspicious activity
- Keep server software updated
- Use firewall to restrict access if needed

## Performance Tips

- **TCPSendBufferSize**: Default 131072 works for most cases
- **MaxPlayers**: Adjust based on server capacity
- Regular database backups: Copy `UserDatabase.db3`
- Monitor server resources (CPU, RAM, bandwidth)

---

**Ready to start?** Follow steps 1-6 above and your server will be ready for players!

For detailed information about the P2P connection fix, see [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md).
