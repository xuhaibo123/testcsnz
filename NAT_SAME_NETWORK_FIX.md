# NAT/Same Network Connection Fix Documentation

## Problem Description

The CSNZ server experiences connection failures when two clients behind the same NAT (Network Address Translation) try to join each other's game rooms.

### Current Situation

- **Server**: Deployed on a public server with IP `119.91.238.117`
- **Clients A and B**: Both in the same LAN, with different private IPs (e.g., `192.168.10.103` and `192.168.10.102`)
- **Shared Public IP**: Both clients share the same public IP address when connecting to the server
- **Symptom**: Clients can independently connect to the server and create rooms, but cannot join each other's rooms

### Error Message

When Client A tries to join Client B's room (or vice versa), the joining client receives:

```
Cannot establish connection to <PUBLIC_IP>:<PORT>. Possible reasons:
1. Host connected to the master server with localhost(127.0.0.1)ip
2. Host has port 27005 or 18327 closed (UDP)
3. You are trying to connect to a host with private IP
4. You are trying to connect to a host with strict NAT
...
链接到服务器错误
```

## Root Cause Analysis

### Why the Problem Occurs

1. **Client A** (192.168.10.103) creates a room
2. **Server** records Client A's public IP (119.91.238.117) as the external address
3. **Client B** (192.168.10.102) attempts to join the room
4. **Server** sends Client A's **public IP** (119.91.238.117) to Client B
5. **Client B** tries to connect to 119.91.238.117
6. **Connection fails** because:
   - Both clients are behind the same NAT
   - The router cannot hairpin/loop back connections from internal clients to its own public IP
   - Client B needs to connect to Client A's **private IP** (192.168.10.103) instead

### Network Diagram

```
                            Internet
                               |
                Public IP: 119.91.238.117
                               |
                        [NAT Router]
                               |
            +------------------+------------------+
            |                                     |
       Client A                              Client B
   192.168.10.103                        192.168.10.102
```

**Current (Broken) Flow:**
```
Client B → Try to connect to 119.91.238.117 (Public IP) → ❌ FAILS
```

**Desired (Working) Flow:**
```
Client B → Try to connect to 192.168.10.103 (Private IP) → ✅ SUCCESS
```

## Solution Overview

The server must detect when the joining client and the host client share the same public IP address, and provide the host's **private/local IP** instead of the **public/external IP** in such cases.

### Logic Flow

```
┌─────────────────────────────────────┐
│ Client B requests to join room      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Get joining client's public IP      │
│ (from socket connection)            │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Get host's public IP                │
│ (from host user's network config)   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Compare: Are IPs the same?          │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       │               │
     YES              NO
       │               │
       ▼               ▼
┌────────────┐  ┌────────────┐
│ Send host's│  │ Send host's│
│ PRIVATE IP │  │ PUBLIC IP  │
│ to joiner  │  │ to joiner  │
└────────────┘  └────────────┘
```

## Required Implementation Changes

> **Note:** This document provides the implementation guide for fixing the NAT/same network issue. The changes must be applied to the C++ source code (available at [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server)) and the server must be recompiled.

### 1. Modify `SendHostJoin` Function

**File:** `src/manager/packetmanager.cpp`

**Current Code:**
```cpp
void CPacketManager::SendHostJoin(IExtendedSocket* socket, IUser* host)
{
    CSendPacket* msg = CreatePacket(socket, PacketId::Host);
    msg->BuildHeader();

    msg->WriteUInt8(HostPacketType::HostJoin);
    msg->WriteUInt32(host->GetID());
    msg->WriteUInt64(0);

    UserNetworkConfig_s network = host->GetNetworkConfig();

    msg->WriteUInt32(ip_string_to_int(network.m_szExternalIpAddress), false);
    msg->WriteUInt16(network.m_nExternalClientPort);
    msg->WriteUInt16(network.m_nExternalServerPort);
    msg->WriteUInt32(ip_string_to_int(network.m_szLocalIpAddress), false);
    msg->WriteUInt16(network.m_nLocalClientPort);
    msg->WriteUInt16(network.m_nLocalServerPort);

    socket->Send(msg);
}
```

**Modified Code with NAT Detection:**
```cpp
void CPacketManager::SendHostJoin(IExtendedSocket* socket, IUser* host)
{
    CSendPacket* msg = CreatePacket(socket, PacketId::Host);
    msg->BuildHeader();

    msg->WriteUInt8(HostPacketType::HostJoin);
    msg->WriteUInt32(host->GetID());
    msg->WriteUInt64(0);

    UserNetworkConfig_s network = host->GetNetworkConfig();

    // Get the joining client's public IP (from socket connection)
    std::string joinerPublicIP = socket->GetIP();
    
    // Get the host's public IP
    std::string hostPublicIP = network.m_szExternalIpAddress;
    
    // Check if both clients are behind the same NAT
    bool sameNetwork = (joinerPublicIP == hostPublicIP);
    
    // Decide which IP to send
    std::string ipToSend = sameNetwork ? network.m_szLocalIpAddress : network.m_szExternalIpAddress;
    
    // Log the decision for debugging
    if (sameNetwork)
    {
        Logger().Info("NAT detected: Sending private IP (%s) to joiner from same network (%s)\n",
                     network.m_szLocalIpAddress.c_str(), joinerPublicIP.c_str());
    }

    msg->WriteUInt32(ip_string_to_int(ipToSend), false);
    msg->WriteUInt16(network.m_nExternalClientPort);
    msg->WriteUInt16(network.m_nExternalServerPort);
    msg->WriteUInt32(ip_string_to_int(network.m_szLocalIpAddress), false);
    msg->WriteUInt16(network.m_nLocalClientPort);
    msg->WriteUInt16(network.m_nLocalServerPort);

    socket->Send(msg);
}
```

### 2. Modify Room User List Sending

**File:** `src/manager/packetmanager.cpp`

Find the function that sends the room user list (around line 2706) and apply similar logic:

**Current Code:**
```cpp
// In SendRoomUserList or similar function
for (auto user : roomInfo->GetUsers())
{
    UserNetworkConfig_s network = user->GetNetworkConfig();
    
    msg->WriteUInt8(user->GetRoomData()->m_Team);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt32(ip_string_to_int(network.m_szExternalIpAddress), false);
    msg->WriteUInt16(network.m_nExternalClientPort);
    msg->WriteUInt16(network.m_nExternalServerPort);
    msg->WriteUInt32(ip_string_to_int(network.m_szLocalIpAddress), false);
    msg->WriteUInt16(network.m_nLocalClientPort);
    msg->WriteUInt16(network.m_nLocalServerPort);
    // ... rest of code
}
```

**Modified Code:**
```cpp
// Get the requesting client's public IP
std::string requestingClientIP = socket->GetIP();

for (auto user : roomInfo->GetUsers())
{
    UserNetworkConfig_s network = user->GetNetworkConfig();
    
    // Check if this user and the requesting client are on the same network
    bool sameNetwork = (requestingClientIP == network.m_szExternalIpAddress);
    std::string ipToSend = sameNetwork ? network.m_szLocalIpAddress : network.m_szExternalIpAddress;
    
    msg->WriteUInt8(user->GetRoomData()->m_Team);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt32(ip_string_to_int(ipToSend), false);
    msg->WriteUInt16(network.m_nExternalClientPort);
    msg->WriteUInt16(network.m_nExternalServerPort);
    msg->WriteUInt32(ip_string_to_int(network.m_szLocalIpAddress), false);
    msg->WriteUInt16(network.m_nLocalClientPort);
    msg->WriteUInt16(network.m_nLocalServerPort);
    // ... rest of code
}
```

### 3. Modify `SendRoomPlayerJoin` Function

**File:** `src/manager/packetmanager.cpp`

Apply the same NAT detection logic:

**Modified Code:**
```cpp
void CPacketManager::SendRoomPlayerJoin(IExtendedSocket* socket, IUser* user, RoomTeamNum num)
{
    CSendPacket* msg = CreatePacket(socket, PacketId::Room);
    msg->BuildHeader();

    msg->WriteUInt8(OutRoomType::PlayerJoin);

    msg->WriteUInt32(user->GetID());
    msg->WriteUInt32(0);
    msg->WriteString(user->GetUsername());

    UserNetworkConfig_s network = user->GetNetworkConfig();
    
    // Get the receiving client's public IP
    std::string receivingClientIP = socket->GetIP();
    
    // Check if they're on the same network
    bool sameNetwork = (receivingClientIP == network.m_szExternalIpAddress);
    std::string ipToSend = sameNetwork ? network.m_szLocalIpAddress : network.m_szExternalIpAddress;

    // user network info
    msg->WriteUInt8(num);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt8(0);
    msg->WriteUInt32(ip_string_to_int(ipToSend), false);
    msg->WriteUInt16(network.m_nExternalClientPort);
    msg->WriteUInt16(network.m_nExternalServerPort);
    msg->WriteUInt32(ip_string_to_int(network.m_szLocalIpAddress), false);
    msg->WriteUInt16(network.m_nLocalClientPort);
    msg->WriteUInt16(network.m_nLocalServerPort);

    CUserCharacter character = user->GetCharacter(UFLAG_LOW_ALL, UFLAG_HIGH_ALL);

    CPacketHelper_FullUserInfo fullUserInfo;
    fullUserInfo.Build(msg->m_OutStream, user->GetID(), character);

    socket->Send(msg);
}
```

## Helper Function (Optional)

To avoid code duplication, you can create a helper function:

```cpp
// Add to packetmanager.h
std::string GetAppropriateIP(const std::string& targetPublicIP, 
                            const std::string& targetLocalIP, 
                            const std::string& requestingClientIP);

// Add to packetmanager.cpp
std::string CPacketManager::GetAppropriateIP(const std::string& targetPublicIP, 
                                            const std::string& targetLocalIP, 
                                            const std::string& requestingClientIP)
{
    // If requesting client and target user share the same public IP, 
    // they're behind the same NAT - return local IP
    if (requestingClientIP == targetPublicIP)
    {
        Logger().Debug("Same network detected (%s), using local IP: %s\n", 
                      requestingClientIP.c_str(), targetLocalIP.c_str());
        return targetLocalIP;
    }
    
    // Different networks - return public IP
    return targetPublicIP;
}
```

Then use it in the send functions:

```cpp
std::string ipToSend = GetAppropriateIP(network.m_szExternalIpAddress, 
                                       network.m_szLocalIpAddress, 
                                       socket->GetIP());
```

## Testing Procedures

### Test Setup

1. **Setup two computers on the same LAN:**
   - Computer A: 192.168.10.103
   - Computer B: 192.168.10.102
   - Both behind NAT router with public IP: 119.91.238.117

2. **Server configuration:**
   - Public server IP: 119.91.238.117 (or different from clients' public IP)
   - ServerConfig.json properly configured with PublicIP

### Test Cases

#### Test Case 1: Same Network - Room Creation and Join

1. **Client A** connects to server and creates a room
2. **Server logs** should show Client A's public IP (119.91.238.117)
3. **Client B** connects to server
4. **Client B** tries to join Client A's room
5. **Expected result:**
   - Server detects same public IP
   - Server sends Client A's private IP (192.168.10.103) to Client B
   - Client B successfully connects to 192.168.10.103
   - Game starts successfully

**Verification in logs:**
```
NAT detected: Sending private IP (192.168.10.103) to joiner from same network (119.91.238.117)
```

#### Test Case 2: Different Networks - Room Creation and Join

1. **Client A** (behind NAT, public IP: 119.91.238.117) creates a room
2. **Client C** (different network, public IP: 185.199.108.153) tries to join
3. **Expected result:**
   - Server detects different public IPs
   - Server sends Client A's public IP (119.91.238.117) to Client C
   - Client C connects to 119.91.238.117
   - Game starts successfully (assuming ports are forwarded)

#### Test Case 3: Three Clients - Mixed Scenario

1. **Client A** and **Client B** on same network (192.168.10.x, public: 119.91.238.117)
2. **Client C** on different network (public: 185.199.108.153)
3. **Client A** creates room
4. **Expected results:**
   - Client B receives Client A's private IP (192.168.10.103)
   - Client C receives Client A's public IP (119.91.238.117)
   - Both clients can join successfully

### Debug Logging

Add these log statements for debugging:

```cpp
Logger().Info("Room join: Joiner IP=%s, Host External IP=%s, Host Local IP=%s, Same Network=%s\n",
             socket->GetIP().c_str(),
             network.m_szExternalIpAddress.c_str(),
             network.m_szLocalIpAddress.c_str(),
             (socket->GetIP() == network.m_szExternalIpAddress) ? "YES" : "NO");
```

## Configuration Requirements

No additional ServerConfig.json changes are required beyond the existing PublicIP configuration. However, ensure:

1. **PublicIP** is correctly set to the server's public IP
2. **Clients' local IP addresses** are properly recorded by the holepunch mechanism
3. The existing P2P connection fix is implemented (see [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md))

## Known Limitations and Considerations

### 1. Firewall Rules

Even with the same network fix, ensure:
- **UDP ports 27005 and 27015** are open on clients' firewalls
- **Local firewall** allows LAN connections

### 2. Private IP Accuracy

The fix relies on clients correctly reporting their private IP addresses through the holepunch mechanism. If a client reports 127.0.0.1:
- The existing P2P fix should replace it with the socket's IP
- However, this may still be the public IP, not the true private IP
- Consider additional logic to detect and handle this scenario

### 3. Complex NAT Scenarios

This solution handles the common case of **symmetric NAT** where clients share the same public IP. More complex scenarios:
- **Carrier-grade NAT (CGNAT)**: Multiple LANs sharing the same public IP
- **Double NAT**: Clients behind multiple layers of NAT

For these cases, additional techniques (STUN/TURN servers) might be needed.

### 4. VPN Connections

Clients using VPNs will have different public IPs even if on the same LAN. This is expected behavior - they should use public IPs to connect.

## Troubleshooting

### Issue: Still cannot connect on same network

**Possible causes:**
1. Client's private IP is not correctly recorded
   - Check server logs for the IP addresses being sent
   - Verify holepunch mechanism is working
2. Local firewall blocking connections
   - Check Windows Firewall rules
   - Temporarily disable to test
3. NAT hairpining is enabled on router
   - Some routers support NAT loopback/hairpining
   - If enabled, the original public IP connection might work

### Issue: Different network clients cannot connect

**Causes:**
1. Port forwarding not configured
2. Firewall blocking external connections
3. ISP blocking UDP ports

**Solution:**
- This fix should not affect different-network connections
- Verify existing P2P connection fix is implemented
- Check port forwarding and firewall rules

## Implementation Checklist

- [ ] Modify `SendHostJoin` function in packetmanager.cpp
- [ ] Modify `SendRoomPlayerJoin` function in packetmanager.cpp
- [ ] Modify room user list sending function in packetmanager.cpp
- [ ] Add debug logging for NAT detection
- [ ] Compile the modified source code
- [ ] Replace CSNZ_Server.exe with the new binary
- [ ] Test with two clients on same network
- [ ] Test with clients on different networks
- [ ] Test with mixed scenario (same + different networks)
- [ ] Verify server logs show correct IP selection
- [ ] Document any edge cases discovered during testing

## References

- **Source Repository**: [JusicP/CSNZ_Server](https://github.com/JusicP/CSNZ_Server)
- **P2P Connection Fix**: [P2P_CONNECTION_FIX.md](P2P_CONNECTION_FIX.md)
- **Related Issue**: NAT traversal for P2P game connections
- **Reference Files**:
  - `src/manager/packetmanager.cpp` - Packet sending logic
  - `src/user/user.cpp` - User network configuration
  - `src/definitions.h` - UserNetworkConfig_s structure

## Summary

This fix enables clients behind the same NAT to play together by:

1. **Detecting** when joining client and host share the same public IP
2. **Providing** the host's private/local IP instead of public IP to the joiner
3. **Maintaining** existing behavior for clients on different networks

The implementation requires modifying the packet sending functions in `packetmanager.cpp` to compare client public IPs and conditionally send private or public IPs accordingly.
