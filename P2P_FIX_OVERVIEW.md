# P2P Connection Flow - Before and After Fix

## Problem: Localhost IP in P2P Connections

### Before Fix ❌

```
Client A (Host)                      Server                      Client B (Joiner)
-----------                          ------                      --------------
Creates room
  |
  |--(Holepunch: 127.0.0.1)-------->|
  |                                  |
  |                                  |---(Room Info: Host IP = 127.0.0.1)---->|
  |                                  |                                         |
  |<-----------------------(Try to connect to 127.0.0.1)----------------------|
  |                                                                            |
  X Connection fails! (Client B tries to connect to its own localhost)        X
```

**Issue:** When Client A creates a room, the holepunch mechanism records the local IP as `127.0.0.1`. When Client B tries to join, it receives `127.0.0.1` as the host's IP address and attempts to connect to its own localhost, which fails.

### After Fix ✅

```
Client A (Host)                      Server (PublicIP configured)    Client B (Joiner)
-----------                          ----------------------------    --------------
Creates room
  |
  |--(Holepunch: 127.0.0.1)-------->|
  |                                  | Server detects localhost
  |                                  | Replaces with PublicIP
  |                                  | (119.91.238.117)
  |                                  |
  |                                  |---(Room Info: Host IP = 119.91.238.117)-->|
  |                                  |                                            |
  |<-----------------------(Connect to 119.91.238.117)---------------------------|
  |                                                                               |
  ✓ Connection successful! (Client B connects to actual public IP)               ✓
```

**Solution:** The server detects when a client sends `127.0.0.1` in the holepunch data and replaces it with the configured `PublicIP` (or the socket's IP address). Client B now receives the correct public IP and can successfully connect.

## Code Changes Summary

### Configuration
```json
{
  "PublicIP": "YOUR_PUBLIC_IP",  // <-- New field (replace with actual IP)
  "GameRoomIP": "YOUR_PUBLIC_IP" // <-- Should match PublicIP
}
```

### 2. Server Initialization (CUser Constructor)

```cpp
// OLD CODE:
m_NetworkData.m_szExternalIpAddress = m_pSocket->GetIP();
m_NetworkData.m_szLocalIpAddress = m_pSocket->GetIP();

// NEW CODE:
std::string externalIP = m_pSocket->GetIP();
if (!g_pServerConfig->publicIP.empty()) {
    externalIP = g_pServerConfig->publicIP;
}
m_NetworkData.m_szExternalIpAddress = externalIP;
m_NetworkData.m_szLocalIpAddress = externalIP;
```

### 3. Holepunch Handler (UpdateHolepunch)

```cpp
// OLD CODE:
m_NetworkData.m_szLocalIpAddress = localIpAddress;

// NEW CODE:
std::string ipAddress = localIpAddress;
if (ipAddress == "127.0.0.1" || ipAddress == "localhost") {
    ipAddress = m_pSocket->GetIP();
    if (!g_pServerConfig->publicIP.empty()) {
        ipAddress = g_pServerConfig->publicIP;
    }
}
m_NetworkData.m_szLocalIpAddress = ipAddress;
```

## Network Architecture

### Deployment Scenario 1: Server Behind NAT

```
                                Internet
                                   |
                      Router (Public IP: 119.91.238.117)
                                   |
                          Port Forwarding:
                          - TCP 30002
                          - UDP 27005, 27015
                                   |
                      +------------+------------+
                      |                         |
            CSNZ Server (LAN)         Other Devices
            IP: 192.168.1.100
            PublicIP: 119.91.238.117
```

**Configuration:**
```json
{
  "IP": "0.0.0.0",
  "PublicIP": "119.91.238.117",
  "GameRoomIP": "119.91.238.117"
}
```

### Deployment Scenario 2: Cloud/VPS Server

```
                                Internet
                                   |
                      Cloud Server (Public IP: 185.199.108.153)
                      - No NAT
                      - Direct public IP
                                   |
                      CSNZ Server
                      IP: 185.199.108.153
                      PublicIP: 185.199.108.153
```

**Configuration:**
```json
{
  "IP": "0.0.0.0",
  "PublicIP": "185.199.108.153",
  "GameRoomIP": "185.199.108.153"
}
```

## Port Usage

| Port  | Protocol | Purpose                | Required For        |
|-------|----------|------------------------|---------------------|
| 30002 | TCP      | Client connections     | Server access       |
| 27005 | UDP      | P2P client port        | P2P room hosting    |
| 27015 | UDP      | P2P server port        | P2P room joining    |

## Testing Checklist

- [ ] Configure `PublicIP` in ServerConfig.json
- [ ] Verify JSON is valid
- [ ] Start server and check logs for errors
- [ ] Create a room from Client A
- [ ] Join room from Client B (different network)
- [ ] Verify connection succeeds
- [ ] Check network logs for IP addresses
- [ ] Confirm no `127.0.0.1` addresses in P2P traffic
- [ ] Test gameplay stability

## Common Issues and Solutions

### Issue: "Connection timed out"
**Cause:** Firewall or port forwarding misconfigured  
**Solution:** Verify ports 30002 (TCP), 27005 (UDP), 27015 (UDP) are open

### Issue: "Host unreachable"
**Cause:** Incorrect PublicIP configuration  
**Solution:** Verify PublicIP matches your actual public IP

### Issue: Still connecting to 127.0.0.1
**Cause:** Binary doesn't support PublicIP configuration  
**Solution:** Recompile server with source code changes (see P2P_CONNECTION_FIX.md)

### Issue: Works on LAN but not internet
**Cause:** Router not forwarding ports correctly  
**Solution:** Check router port forwarding configuration

## Implementation Status

### Current Repository Status

- ✅ **Configuration changes**: ServerConfig.json updated with PublicIP field
- ✅ **Documentation**: Complete implementation guide provided
- ⚠️ **Binary support**: Current CSNZ_Server.exe may not support PublicIP field
- ⚠️ **Source code**: Not included in this repository

### Next Steps to Enable Fix

1. **Option A - Recompile from source:**
   - Get source code from [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server)
   - Apply changes documented in P2P_CONNECTION_FIX.md
   - Compile new binary
   - Replace CSNZ_Server.exe

2. **Option B - Use updated binary:**
   - Obtain a binary that already includes the fix
   - Replace CSNZ_Server.exe
   - Configuration already correct (PublicIP field present)

## References

- **Source Repository**: [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server)
- **Implementation Guide**: [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md)
- **Configuration Examples**: [CONFIG_EXAMPLES.md](CONFIG_EXAMPLES.md)
- **General Documentation**: [README.md](README.md)
