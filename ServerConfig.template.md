# ServerConfig.json Template

This is a template for ServerConfig.json. Replace the placeholder values with your actual server configuration.

## Important: Before Using

**DO NOT commit your actual ServerConfig.json with real IP addresses to a public repository!**

This template shows the structure. Copy this to `myGameServer/ServerConfig.json` and customize it for your setup.

## Template

```json
{
  "HostName": "Your Server Name",
  "Description": "Your Server Description",
  "IP": "0.0.0.0",
  "Port": "30002",
  "PublicIP": "YOUR_PUBLIC_IP_HERE",
  "GameRoomIP": "YOUR_PUBLIC_IP_HERE",
  "TCPSendBufferSize": 131072,
  "MaxPlayers": 100,
  "WelcomeMessage": "Welcome to the server!",
  "RestartOnCrash": true,
  "InventorySlotMax": 3000,
  "CheckClientBuild": false,
  "AllowedClientTimestamp": 0,
  "AllowedLauncherVersion": 67,
  "MaxRegistrationsPerIP": 3,
  "SSL": false,
  "Crypt": false,
  "MainMenuSkinEvent": 0,
  "BanListMaxSize": 300,
  "Metadata": {
    "Maplist": true,
    "ClientTable": true,
    "Unk3": true,
    "ItemBox": true,
    "Unk8": true,
    "MatchOption": true,
    "ZombieWarWeaponList": true,
    "WeaponParts": true,
    "Unk20": true,
    "Encyclopedia": false,
    "GameModeList": true,
    "ProgressUnlock": true,
    "WeaponPaints": true,
    "ReinforceMaxLvl": true,
    "ReinforceMaxEXP": true,
    "ReinforceItemsExp": true,
    "Unk31": true,
    "HonorMoneyShop": false,
    "ItemExpireTime": true,
    "ScenarioTX_Common": true,
    "ScenarioTX_Dedi": true,
    "ShopItemList_Dedi": true,
    "ZBCompetitive": true,
    "Unk43": true,
    "Unk49": true,
    "PPSystem": true,
    "CodisData": true,
    "Item": false,
    "WeaponProp": true,
    "Hash": false,
    "RandomWeaponList": true,
    "ModeEvent": true,
    "MileageShop": false,
    "EventShop": false,
    "FamilyTotalWarMap": true,
    "FamilyTotalWar": true,
    "Unk54": true,
    "Unk55": true
  },
  "Room": {
    "HostConnectingMethod": 1,
    "ValidateSettings": false
  }
}
```

## Customization Guide

### Required Changes

1. **PublicIP**: Replace `YOUR_PUBLIC_IP_HERE` with your server's public IP address
   - Example: `"PublicIP": "203.0.113.45"`
   - Find your IP at: https://whatismyipaddress.com/

2. **GameRoomIP**: Should match your PublicIP
   - Example: `"GameRoomIP": "203.0.113.45"`

3. **HostName**: Your server's display name
   - Example: `"HostName": "My CSNZ Server"`

### Optional Changes

- **Description**: Server description shown to players
- **Port**: TCP port for connections (default: 30002)
- **MaxPlayers**: Maximum concurrent players (default: 100)
- **WelcomeMessage**: Message shown when players connect
- **MaxRegistrationsPerIP**: Limit accounts per IP (default: 3)

### Security Considerations

⚠️ **Important Security Notes:**

1. **Don't commit sensitive information**:
   - Real IP addresses
   - Custom passwords (if added)
   - Private server details

2. **Use .gitignore** to exclude your actual config:
   ```
   myGameServer/ServerConfig.json
   ```

3. **Keep a local backup** of your configured ServerConfig.json

4. **Share only templates** when collaborating

## Examples by Scenario

### Local Network (LAN)
```json
{
  "IP": "192.168.1.100",
  "PublicIP": "192.168.1.100",
  "GameRoomIP": "192.168.1.100"
}
```

### Public Internet Server
```json
{
  "IP": "0.0.0.0",
  "PublicIP": "203.0.113.45",
  "GameRoomIP": "203.0.113.45"
}
```

### Behind NAT/Router
```json
{
  "IP": "0.0.0.0",
  "PublicIP": "YOUR_ROUTER_PUBLIC_IP",
  "GameRoomIP": "YOUR_ROUTER_PUBLIC_IP"
}
```

### Development/Testing
```json
{
  "IP": "127.0.0.1",
  "PublicIP": "127.0.0.1",
  "GameRoomIP": "127.0.0.1"
}
```

## Validation

After editing, validate your JSON:

```bash
# Using Python
python3 -m json.tool myGameServer/ServerConfig.json

# Using jq
jq . myGameServer/ServerConfig.json

# Online
# Visit https://jsonlint.com/ and paste your config
```

## See Also

- [SETUP_GUIDE.md](SETUP_GUIDE.md) - Complete setup instructions
- [CONFIG_EXAMPLES.md](CONFIG_EXAMPLES.md) - More configuration examples
- [README.md](README.md) - General documentation
