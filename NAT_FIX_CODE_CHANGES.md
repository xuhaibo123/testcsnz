# NAT Same Network Fix - Code Changes

This file contains the exact code changes needed to implement the NAT/same network fix.

## File: src/manager/packetmanager.cpp

### Change 1: Modify SendHostJoin Function

**Location**: Around line 3420

**Before:**
```cpp
void CPacketManager::SendHostJoin(IExtendedSocket* socket, IUser* host)
{
	CSendPacket* msg = CreatePacket(socket, PacketId::Host);
	msg->BuildHeader();

	msg->WriteUInt8(HostPacketType::HostJoin);
	msg->WriteUInt32(host->GetID());
	msg->WriteUInt64(0); // что это?

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

**After:**
```cpp
void CPacketManager::SendHostJoin(IExtendedSocket* socket, IUser* host)
{
	CSendPacket* msg = CreatePacket(socket, PacketId::Host);
	msg->BuildHeader();

	msg->WriteUInt8(HostPacketType::HostJoin);
	msg->WriteUInt32(host->GetID());
	msg->WriteUInt64(0); // что это?

	UserNetworkConfig_s network = host->GetNetworkConfig();

	// NAT Same Network Fix: Check if joiner and host are on same network
	std::string joinerPublicIP = socket->GetIP();
	std::string hostPublicIP = network.m_szExternalIpAddress;
	bool sameNetwork = (joinerPublicIP == hostPublicIP);
	
	// Use local IP if same network, otherwise use external IP
	std::string ipToSend = sameNetwork ? network.m_szLocalIpAddress : network.m_szExternalIpAddress;
	
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

### Change 2: Modify SendRoomPlayerJoin Function

**Location**: Around line 2737

**Before:**
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

	// user network info
	msg->WriteUInt8(num);
	msg->WriteUInt8(0);
	msg->WriteUInt8(0);
	msg->WriteUInt8(0);
	msg->WriteUInt32(ip_string_to_int(network.m_szExternalIpAddress), false);
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

**After:**
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

	// NAT Same Network Fix: Check if receiver and joining user are on same network
	std::string receiverPublicIP = socket->GetIP();
	std::string userPublicIP = network.m_szExternalIpAddress;
	bool sameNetwork = (receiverPublicIP == userPublicIP);
	
	// Use local IP if same network, otherwise use external IP
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

### Change 3: Modify Room User List Sending

**Location**: Around line 2706 (in SendRoomJoinNewUser or similar function)

**Before:**
```cpp
msg->WriteUInt8(roomInfo->GetUsers().size());
for (auto user : roomInfo->GetUsers())
{
	UserNetworkConfig_s network = user->GetNetworkConfig();

	msg->WriteUInt32(user->GetID());
	msg->WriteUInt32(0);
	msg->WriteString(user->GetUsername());

	// user network info
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

	CUserCharacter character = user->GetCharacter(UFLAG_LOW_ALL, UFLAG_HIGH_ALL);

	CPacketHelper_FullUserInfo fullUserInfo;
	fullUserInfo.Build(msg->m_OutStream, user->GetID(), character);
}
```

**After:**
```cpp
// NAT Same Network Fix: Get requesting client's public IP once before loop
std::string requestingClientIP = socket->GetIP();

msg->WriteUInt8(roomInfo->GetUsers().size());
for (auto user : roomInfo->GetUsers())
{
	UserNetworkConfig_s network = user->GetNetworkConfig();

	msg->WriteUInt32(user->GetID());
	msg->WriteUInt32(0);
	msg->WriteString(user->GetUsername());

	// NAT Same Network Fix: Check if this user and requesting client are on same network
	bool sameNetwork = (requestingClientIP == network.m_szExternalIpAddress);
	std::string ipToSend = sameNetwork ? network.m_szLocalIpAddress : network.m_szExternalIpAddress;

	// user network info
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

	CUserCharacter character = user->GetCharacter(UFLAG_LOW_ALL, UFLAG_HIGH_ALL);

	CPacketHelper_FullUserInfo fullUserInfo;
	fullUserInfo.Build(msg->m_OutStream, user->GetID(), character);
}
```

## Optional: Helper Function

You can add a helper function to reduce code duplication.

### File: src/manager/packetmanager.h

**Add to class declaration:**
```cpp
class CPacketManager : public CBaseManager
{
	// ... existing code ...

private:
	// Helper function for NAT detection
	std::string GetAppropriateIP(const std::string& targetPublicIP, 
	                            const std::string& targetLocalIP, 
	                            const std::string& requestingClientIP);

	// ... rest of the class ...
};
```

### File: src/manager/packetmanager.cpp

**Add implementation:**
```cpp
std::string CPacketManager::GetAppropriateIP(const std::string& targetPublicIP, 
                                            const std::string& targetLocalIP, 
                                            const std::string& requestingClientIP)
{
	// If requesting client and target user share the same public IP,
	// they're behind the same NAT - return local IP for direct LAN connection
	if (requestingClientIP == targetPublicIP)
	{
		Logger().Debug("Same network detected (public IP: %s), using local IP: %s\n", 
		              requestingClientIP.c_str(), targetLocalIP.c_str());
		return targetLocalIP;
	}
	
	// Different networks - return public IP for internet connection
	return targetPublicIP;
}
```

**Then simplify the send functions:**

```cpp
// In SendHostJoin:
std::string ipToSend = GetAppropriateIP(network.m_szExternalIpAddress, 
                                       network.m_szLocalIpAddress, 
                                       socket->GetIP());

// In SendRoomPlayerJoin:
std::string ipToSend = GetAppropriateIP(network.m_szExternalIpAddress, 
                                       network.m_szLocalIpAddress, 
                                       socket->GetIP());

// In room user list loop:
std::string ipToSend = GetAppropriateIP(network.m_szExternalIpAddress, 
                                       network.m_szLocalIpAddress, 
                                       requestingClientIP);
```

## Compilation

After making these changes:

1. **Build the project:**
   ```bash
   # Using your build system (e.g., Visual Studio, CMake, etc.)
   cmake --build . --config Release
   ```

2. **Replace the binary:**
   ```bash
   cp bin/CSNZ_Server.exe /path/to/myGameServer/
   ```

3. **Test the changes:**
   - Test same network scenario
   - Test different network scenario
   - Verify logs show correct IP selection

## Verification

After implementation, check the server logs for messages like:

```
NAT detected: Sending private IP (192.168.10.103) to joiner from same network (119.91.238.117)
```

Or if using the helper function:

```
Same network detected (public IP: 119.91.238.117), using local IP: 192.168.10.103
```

This confirms the fix is working correctly.

## Summary of Changes

1. **Three functions modified** in `packetmanager.cpp`:
   - `SendHostJoin` - Adds NAT detection when sending host info to joiner
   - `SendRoomPlayerJoin` - Adds NAT detection when player joins room
   - Room user list sending - Adds NAT detection for all users in room

2. **Optional helper function** added to reduce code duplication

3. **Logic**: Compare public IPs → If same, use local IP; if different, use public IP

4. **No configuration changes** required in ServerConfig.json (uses existing PublicIP setting)

5. **Backward compatible**: Works with existing clients and maintains current behavior for different-network scenarios
