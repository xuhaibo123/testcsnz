# P2P Connection Fix Documentation

## Problem Description

The CSNZ server is experiencing P2P connection failures when clients try to join rooms created by other clients. The issue stems from the UDP holepunch mechanism recording localhost (127.0.0.1) as the local IP address, causing joining clients to attempt connections to their own localhost instead of the host's correct IP.

## Solution Overview

The fix requires three main changes:

1. **Add PublicIP configuration option** in ServerConfig.json
2. **Update CUser constructor** to use the public IP for external IP address when configured
3. **Modify UpdateHolepunch function** to replace localhost IPs with the socket's IP address

## Configuration Changes

### ServerConfig.json

A new `PublicIP` field has been added to the configuration:

```json
{
  "HostName": "Giacomo CSNZ",
  "Description": "平行时空2024",
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "119.91.238.117",
  "GameRoomIP": "119.91.238.117",
  ...
}
```

**Field Description:**
- `PublicIP`: The public IP address of the server that clients should connect to. This is used to replace localhost (127.0.0.1) addresses in the UDP holepunch mechanism.
- If not specified or empty, the server will use the socket's IP address as before.

## Required Source Code Changes

> **Note:** This repository contains only binary files. The following changes would need to be implemented in the C++ source code (reference: JusicP/CSNZ_Server repository) and recompiled.

### 1. ServerConfig (serverconfig.h / serverconfig.cpp)

#### serverconfig.h
Add the publicIP field to the CServerConfig class:

```cpp
class CServerConfig
{
public:
    // ... existing fields ...
    std::string publicIP;  // Add this line
    std::string hostName;
    std::string description;
    // ... rest of fields ...
};
```

#### serverconfig.cpp
Load the PublicIP from the configuration file in the `Load()` function:

```cpp
bool CServerConfig::Load()
{
    try
    {
        ordered_json cfg;
        ifstream f("ServerConfig.json");
        
        // ... existing code ...
        
        hostName = cfg.value("HostName", "CSO Server");
        description = cfg.value("Description", "");
        publicIP = cfg.value("PublicIP", "");  // Add this line
        tcpPort = udpPort = cfg.value("Port", DEFAULT_PORT);
        
        // ... rest of the code ...
    }
    // ... rest of function ...
}
```

### 2. CUser Constructor (user.cpp)

Update the constructor to use publicIP when configured:

```cpp
CUser::CUser(IExtendedSocket* sock, int userID, const std::string& userName)
{
    m_pCurrentRoom = NULL;
    m_pCurrentChannel = NULL;
    m_pLastChannelServer = NULL;
    m_pRoomData = NULL;

    m_pSocket = sock;
    m_nID = userID;
    m_UserName = userName;

    m_Status = UserStatus::STATUS_MENU;

    // Modified: Use publicIP if configured, otherwise use socket IP
    std::string externalIP = m_pSocket->GetIP();
    if (!g_pServerConfig->publicIP.empty())
    {
        externalIP = g_pServerConfig->publicIP;
    }
    
    m_NetworkData.m_szExternalIpAddress = externalIP;
    m_NetworkData.m_nExternalClientPort = 27005;
    m_NetworkData.m_nExternalServerPort = 27015;
    m_NetworkData.m_szLocalIpAddress = externalIP;  // Changed from m_pSocket->GetIP()
    m_NetworkData.m_nLocalClientPort = 27005;
    m_NetworkData.m_nLocalServerPort = 27015;

    m_nUptime = 0;
}
```

### 3. UpdateHolepunch Function (user.cpp)

Modify the function to replace localhost (127.0.0.1) with the socket's IP:

```cpp
int CUser::UpdateHolepunch(int portId, const string& localIpAddress, int localPort, int externalPort)
{
    // Replace localhost with the socket's actual IP address
    std::string ipAddress = localIpAddress;
    if (ipAddress == "127.0.0.1" || ipAddress == "localhost")
    {
        ipAddress = m_pSocket->GetIP();
        
        // If publicIP is configured, use that instead
        if (!g_pServerConfig->publicIP.empty())
        {
            ipAddress = g_pServerConfig->publicIP;
        }
    }
    
    switch (portId)
    {
    case 0:
        m_NetworkData.m_szLocalIpAddress = ipAddress;  // Changed from localIpAddress
        m_NetworkData.m_nLocalServerPort = localPort;
        m_NetworkData.m_nExternalServerPort = externalPort;
        return 0;
    case 1:
        m_NetworkData.m_szLocalIpAddress = ipAddress;  // Changed from localIpAddress
        m_NetworkData.m_nLocalClientPort = localPort;
        m_NetworkData.m_nExternalClientPort = externalPort;
        return 1;
    default:
        return -1;
    };
}
```

## Testing Procedure

1. **Configure the server:**
   - Set `PublicIP` to your server's public IP address in ServerConfig.json
   - Ensure `GameRoomIP` matches the `PublicIP` value

2. **Test P2P connections:**
   - Host a room from Client A
   - Join the room from Client B (different network)
   - Verify that Client B connects to the correct IP (not 127.0.0.1)
   - Check game logs for holepunch data

3. **Verify localhost replacement:**
   - Monitor network traffic to ensure no 127.0.0.1 addresses are being transmitted
   - Confirm clients can successfully join and play in P2P rooms

## Compatibility Notes

- If `PublicIP` is not configured (empty or missing), the server behaves as before
- This change is backward compatible with existing configurations
- The `GameRoomIP` field should typically match the `PublicIP` value for proper operation

## References

- Reference implementation: [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server)
- Related files:
  - `src/serverconfig.h` - Configuration header
  - `src/serverconfig.cpp` - Configuration implementation
  - `src/user/user.h` - User class header
  - `src/user/user.cpp` - User class implementation with holepunch logic
