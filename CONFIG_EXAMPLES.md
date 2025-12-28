# ServerConfig.json Examples

This file provides examples of different ServerConfig.json configurations for various deployment scenarios.

## Example 1: Local Network Server (LAN Only)

For a server that only accepts connections from the local network:

```json
{
  "HostName": "Local CSNZ Server",
  "Description": "LAN Server",
  "IP": "192.168.1.100",
  "Port": "30002",
  "PublicIP": "192.168.1.100",
  "GameRoomIP": "192.168.1.100",
  ...
}
```

## Example 2: Public Internet Server

For a server accessible from the internet:

```json
{
  "HostName": "Public CSNZ Server",
  "Description": "Internet Server",
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "203.0.113.45",
  "GameRoomIP": "203.0.113.45",
  ...
}
```

**Note:** Replace `203.0.113.45` with your actual public IP address.

## Example 3: Server Behind NAT/Router

For a server behind a router with port forwarding:

```json
{
  "HostName": "Home CSNZ Server",
  "Description": "Server behind NAT",
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "98.76.54.32",
  "GameRoomIP": "98.76.54.32",
  ...
}
```

**Requirements:**
- Forward TCP port 30002 to your server's local IP
- Forward UDP ports 27005 and 27015 to your server's local IP
- Replace `98.76.54.32` with your router's public IP

## Example 4: Cloud/VPS Server

For a server on a cloud provider (AWS, DigitalOcean, etc.):

```json
{
  "HostName": "Cloud CSNZ Server",
  "Description": "VPS Server",
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "185.199.108.153",
  "GameRoomIP": "185.199.108.153",
  ...
}
```

**Note:** 
- Use your VPS's public IP address
- Ensure firewall rules allow TCP 30002 and UDP 27005, 27015

## Example 5: Localhost Testing (Development)

For testing on your local machine only:

```json
{
  "HostName": "Dev CSNZ Server",
  "Description": "Development Server",
  "IP": "127.0.0.1",
  "Port": "30002",
  "PublicIP": "127.0.0.1",
  "GameRoomIP": "127.0.0.1",
  ...
}
```

**Note:** This configuration only works for testing with clients on the same machine.

## Port Configuration

### Standard Ports:
- **TCP Port**: 30002 (configurable via "Port")
- **UDP Client Port**: 27005 (hardcoded in binary)
- **UDP Server Port**: 27015 (hardcoded in binary)

### Custom Port Example:

```json
{
  "IP": "0.0.0.0",
  "Port": "30003",
  "PublicIP": "YOUR.PUBLIC.IP",
  "GameRoomIP": "YOUR.PUBLIC.IP",
  ...
}
```

**Important:** If you change the TCP port, update your port forwarding rules accordingly.

## Troubleshooting

### Problem: Clients can't join P2P rooms

**Solution:** Ensure `PublicIP` and `GameRoomIP` are set to your public IP address (not 127.0.0.1 or a private IP).

### Problem: Server not accessible from internet

**Possible causes:**
1. `PublicIP` is incorrect
2. Port forwarding not configured properly
3. Firewall blocking connections
4. ISP blocking ports

**Solutions:**
1. Verify your public IP at https://whatismyipaddress.com/
2. Check router's port forwarding settings
3. Allow TCP 30002 and UDP 27005, 27015 through firewall
4. Contact ISP if ports are blocked

### Problem: P2P connections show 127.0.0.1

**Solution:** This is the issue addressed by the P2P connection fix. Either:
1. Use a server binary that includes the fix (see P2P_CONNECTION_FIX.md)
2. Ensure `PublicIP` is correctly configured in ServerConfig.json

## Additional Configuration Options

For a complete list of all configuration options, refer to the main `ServerConfig.json` file. Some important ones:

```json
{
  "MaxPlayers": 100,              // Maximum concurrent players
  "WelcomeMessage": "Welcome!",    // Message shown on connect
  "CheckClientBuild": false,       // Verify client version
  "MaxRegistrationsPerIP": 3,      // Limit accounts per IP
  "Room": {
    "HostConnectingMethod": 1,     // 1 = Direct, 2 = P2P
    "ValidateSettings": false      // Validate room settings
  }
}
```

### HostConnectingMethod Values:
- `1`: Direct connection (server relays traffic)
- `2`: P2P connection (clients connect directly)

**Note:** For P2P mode to work properly, the PublicIP fix must be applied.
