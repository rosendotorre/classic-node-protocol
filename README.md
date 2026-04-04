<div align="center">

# classic-node-protocol

<p>
  <a href="https://www.npmjs.com/package/classic-node-protocol">
    <img src="https://img.shields.io/npm/v/classic-node-protocol?style=flat-square&color=cb3837&logo=npm" alt="npm version" />
  </a>
  <a href="https://www.npmjs.com/package/classic-node-protocol">
    <img src="https://img.shields.io/npm/dm/classic-node-protocol?style=flat-square&color=cb3837&logo=npm" alt="npm downloads" />
  </a>
  <img src="https://img.shields.io/badge/node-%3E%3D18-brightgreen?style=flat-square&logo=nodedotjs" alt="Node.js ≥ 18" />
  <img src="https://img.shields.io/badge/dependencies-0-success?style=flat-square" alt="zero dependencies" />
  <img src="https://img.shields.io/badge/license-MIT-blue?style=flat-square" alt="MIT License" />
  <img src="https://img.shields.io/badge/protocol-Classic%20v7-blueviolet?style=flat-square" alt="Classic v7" />
</p>

**Minecraft Classic v7 / ClassiCube protocol library for Node.js.**  
Full client & server · 24 CPE extensions · Auth & heartbeat · Zero dependencies.

```sh
npm install classic-node-protocol
```

</div>

---

## Table of Contents

- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [createServer & ClassiCubeServer](#createserver--classiccubeserver)
- [ClientConnection](#clientconnection)
- [createClient & ClassiCubeClient](#createclient--classiccubeclient)
- [PacketDecoder](#packetdecoder)
- [encoder](#encoder)
- [protocol — Constants](#protocol--constants)
- [level — Maps & Compression](#level--maps--compression)
- [auth — Authentication & Heartbeat](#auth--authentication--heartbeat)
- [cpe — Classic Protocol Extensions](#cpe--classic-protocol-extensions)
- [jugadorUUID](#jugadoruuid)
- [TypeScript](#typescript)
- [Protocol Reference](#protocol-reference)

---

## Quick Start

<details>
<summary><strong>Minimal server</strong></summary>

```js
const { createServer, level } = require('classic-node-protocol');

const srv = createServer({ port: 25565, pingInterval: 2000 });

srv.on('connection', async (client) => {
  client.sendIdentification('My Server', 'Welcome!');

  const blocks = level.buildFlatMap(64, 64, 64);
  await client.sendLevel(blocks, 64, 64, 64);

  client.spawnAt(-1, 'You', 32, 33, 32);

  client.on('packet', (p) => {
    if (p.name === 'message') {
      srv.broadcastMessage(`<${client.username}> ${p.message}`);
    }
  });
});

srv.on('listening', ({ port }) => console.log(`Listening on :${port}`));
```

</details>

<details>
<summary><strong>Minimal bot (client)</strong></summary>

```js
const { createClient } = require('classic-node-protocol');

const bot = createClient({ host: 'localhost', port: 25565 });

bot.on('connect', () => bot.sendIdentification('MyBot'));

bot.on('level', ({ xSize, ySize, zSize }) => {
  console.log(`Map received: ${xSize}×${ySize}×${zSize}`);
  bot.sendMessage('Hello from the bot!');
});

bot.on('packet', (p) => {
  if (p.name === 'message') console.log('[Chat]', p.message);
});
```

</details>

<details>
<summary><strong>Full multiplayer relay server</strong></summary>

```js
const { createServer, level, encoder, BLOCKS } = require('classic-node-protocol');

const srv   = createServer({ port: 25565, pingInterval: 3000 });
const world = level.buildFlatMap(128, 64, 128);

srv.on('connection', async (client) => {
  // 1. Greet client
  client.sendIdentification('Demo Server', 'Have fun!');

  // 2. Wait for identification packet
  await new Promise((resolve) => {
    client.once('packet', (p) => {
      if (p.name === 'identification') {
        client.username = p.username;
        resolve();
      }
    });
  });

  // 3. Send the world
  await client.sendLevel(world, 128, 64, 128);

  // 4. Spawn self + already connected players
  client.spawnAt(-1, client.username, 64, 33, 64);
  for (const [id, other] of srv.clients) {
    if (id === client.id) continue;
    client.spawnAt(other.id, other.username, 64, 33, 64);
    other.spawnAt(client.id, client.username, 64, 33, 64);
  }

  // 5. Handle packets
  client.on('packet', (p) => {
    switch (p.name) {
      case 'message':
        srv.broadcastMessage(`<${client.username}> ${p.message}`);
        break;

      case 'setBlock': {
        const blockId = p.mode === 1 ? p.blockType : BLOCKS.AIR;
        world[level.blockIndex(p.x, p.z, p.y, 128, 128)] = blockId;
        srv.broadcast(encoder.encodeServerSetBlock(p.x, p.y, p.z, blockId));
        break;
      }

      case 'position':
        srv.broadcastExcept(
          client.id,
          encoder.encodePosition(client.id, p.x, p.y, p.z, p.yaw, p.pitch)
        );
        break;
    }
  });
});

srv.on('disconnect', (client) => {
  srv.broadcastMessage(`${client.username} left the game`);
  srv.broadcast(encoder.encodeDespawnPlayer(client.id));
});
```

</details>

---

## Architecture

```
classic-node-protocol
│
├── createServer()  ──→  ClassiCubeServer
│                          └── emits ClientConnection per player
│
├── createClient()  ──→  ClassiCubeClient
│                          └── auto-assembles level → emits 'level'
│
├── PacketDecoder   ──→  streaming parser (EventEmitter → 'packet')
│
├── encoder         ──→  pure functions → Buffer  (no I/O)
│
├── protocol        ──→  packet IDs, sizes, BLOCKS, CPE_EXTENSIONS …
│
├── level           ──→  map generators + gzip pipeline
│
├── auth            ──→  ClassiCubeAuth (server) + ClassiCubeAccount (bot)
│
├── cpe             ──→  24+ CPE extension encoders / decoders
│
└── jugadorUUID     ──→  deterministic UUID from username
```

Everything is **CommonJS** (`require()`), zero runtime dependencies, Node ≥ 18.

---

## `createServer` & `ClassiCubeServer`

### Factory

```js
const { createServer } = require('classic-node-protocol');

const server = createServer({
  port:         25565,   // start listening immediately when provided
  host:         '0.0.0.0',
  maxClients:   20,      // 0 = unlimited (default)
  pingInterval: 2000,    // ms between auto-pings to all clients (0 = off)
});
```

### Manual instantiation

```js
const { ClassiCubeServer } = require('classic-node-protocol');

const server = new ClassiCubeServer({ maxClients: 10, pingInterval: 5000 });
await server.listen(25565, '0.0.0.0');
```

### Events

| Event | Callback | When |
|---|---|---|
| `'listening'` | `({ port, host })` | Server is accepting connections |
| `'connection'` | `(client: ClientConnection)` | New TCP connection |
| `'disconnect'` | `(client: ClientConnection)` | Client disconnected cleanly |
| `'clientError'` | `(client, err)` | Socket error on a client |
| `'error'` | `(err)` | Server-level error (e.g. `EADDRINUSE`) |

### Properties & methods

```js
server.clients       // Map<number, ClientConnection> — all live connections
server.playerCount   // shorthand for server.clients.size

// Lifecycle
await server.listen(port, host)       // start accepting connections
await server.close()                  // stop accepting new connections
await server.shutdown(reason)         // disconnect everyone, then close

// Broadcast helpers
server.broadcast(buf)                 // raw Buffer → every client
server.broadcastMessage(msg, id)      // chat packet (id default 0xFF = server)
server.broadcastExcept(excludeId, buf)// everyone except one client id
server.relayMessage(fromId, message)  // relay player chat to everyone else
server.pingAll()                      // send 0x01 Ping to all clients
```

---

## `ClientConnection`

Every connected player is a `ClientConnection` received in the `'connection'` event.

### Properties

```js
client.id             // number   — unique session ID
client.socket         // net.Socket
client.username       // string | null — set this yourself after identification
client.state          // 'login' | 'level' | 'play'
client.data           // {}       — free-form storage for your app

client.remoteAddress  // string
client.remotePort     // number
client.isConnected    // boolean

// CPE
client.supportsCpe    // boolean — did client send unused=0x42?
client.cpeReady       // boolean — CPE negotiation complete
client.extensions     // Set<string> — mutually agreed extensions
```

### Sending packets (server → client)

<details>
<summary><strong>Core protocol methods</strong></summary>

```js
client.sendIdentification(serverName, motd, userType)  // 0x00
client.sendPing()                                       // 0x01
client.sendLevelInitialize()                            // 0x02  state → 'level'
client.sendLevelDataChunk(chunkBuf, percentComplete)    // 0x03
client.sendLevelFinalize(xSize, ySize, zSize)           // 0x04  state → 'play'
client.sendSetBlock(x, y, z, blockType)                 // 0x06
client.sendSpawnPlayer(id, name, x, y, z, yaw, pitch)  // 0x07  FShort coords
client.spawnAt(id, name, bx, by, bz, yaw, pitch)       // 0x07  block coords
client.sendPosition(id, x, y, z, yaw, pitch)            // 0x08  absolute FShort
client.sendPositionOrientation(id, dx, dy, dz, yaw, pitch) // 0x09
client.sendPositionUpdate(id, dx, dy, dz)               // 0x0A
client.sendOrientationUpdate(id, yaw, pitch)            // 0x0B
client.sendDespawnPlayer(playerId)                      // 0x0C
client.sendMessage(senderId, message)                   // 0x0D
client.sendServerMessage(message)                       // 0x0D  sender = 0xFF
client.disconnect(reason)                               // 0x0E + socket.destroy()
client.sendUpdateUserType(userType)                     // 0x0F
client.setOperator(isOp)                                // 0x0F  convenience

// All-in-one level send:
await client.sendLevel(blocks, xSize, ySize, zSize, zlibOpts)
// Sends: LevelInitialize → N×DataChunk → LevelFinalize
// zlibOpts example: { level: 9 }
```

</details>

<details>
<summary><strong>CPE methods</strong></summary>

```js
client.sendCpeHandshake(appName, extensions)
client.sendExtInfo(appName, count)
client.sendExtEntry(extName, version)
client.sendSetClickDistance(distance)
client.sendCustomBlockSupportLevel(level)
client.sendHoldThis(blockToHold, preventChange)
client.sendSetTextHotKey(label, action, keyCode, keyMods)
client.sendExtAddPlayerName(nameId, playerName, listName, groupName, groupRank)
client.sendExtAddEntity2(entityId, inGameName, skinName, x, y, z, yaw, pitch)
client.sendExtRemovePlayerName(nameId)
client.sendEnvSetColor(variable, r, g, b)
client.sendMakeSelection(id, label, x1, y1, z1, x2, y2, z2, r, g, b, a)
client.sendRemoveSelection(selectionId)
client.sendSetBlockPermission(blockType, allowPlace, allowDestroy)
client.sendChangeModel(entityId, modelName)
client.sendSetMapEnvAppearance(textureUrl, sideBlock, edgeBlock, sideLevel)
client.sendEnvSetWeatherType(weatherType)
client.sendHackControl(flying, noClip, speeding, spawnControl, thirdPerson, jumpHeight)
client.sendDefineBlock(def)
client.sendDefineBlockExt(def)
client.sendRemoveBlockDefinition(blockId)
client.sendBulkBlockUpdate(updates)
client.sendSetTextColor(code, r, g, b, a)
client.sendSetMapEnvUrl(textureUrl)
client.sendSetMapEnvProperty(property, value)
client.sendSetEntityProperty(entityId, propertyType, value)
client.sendTwoWayPing(direction, data)
client.sendSetInventoryOrder(order, blockType)
```

</details>

### Receiving packets

```js
client.on('packet', (p) => {
  // p.name — 'identification' | 'setBlock' | 'position' | 'message' | …
  // p.id   — raw packet byte ID
  console.log(p.name, p);
});

client.on('end',   ()    => console.log('disconnected'));
client.on('error', (err) => console.error(err));

// Low-level raw write:
client.writeBuffer(someBuffer);
```

---

## `createClient` & `ClassiCubeClient`

### Factory (auto-connects)

```js
const { createClient } = require('classic-node-protocol');

const bot = createClient({
  host:           'example.com',
  port:           25565,
  connectTimeout: 10000, // ms (default 10 s)
  pingInterval:   0,     // ms between client pings (0 = off)
});
```

### Manual connect

```js
const { ClassiCubeClient } = require('classic-node-protocol');

const client = new ClassiCubeClient({ connectTimeout: 5000 });
await client.connect('localhost', 25565);
client.sendIdentification('MyBot', '-');
```

### Events

| Event | Callback | When |
|---|---|---|
| `'connect'` | `()` | TCP handshake complete |
| `'end'` | `()` | Connection closed |
| `'error'` | `(err)` | Socket or decode error |
| `'packet'` | `(packet)` | Any decoded server packet |
| `'level'` | `({ blocks, xSize, ySize, zSize })` | Full map assembled & decompressed |
| `'levelProgress'` | `(percent: number)` | Map download progress 0–100 |
| `'ping'` | `()` | Ping interval tick |

### Methods

```js
// Core
bot.sendIdentification(username, verificationKey, unused)
// verificationKey: '-' offline | real MPPass for online-mode servers
// unused: 0x00 vanilla | 0x42 (CPE_MAGIC) to signal CPE support

bot.sendSetBlock(x, y, z, mode, blockType) // mode: 0=destroy 1=create
bot.sendPosition(x, y, z, yaw, pitch)      // FShort coordinates
bot.sendPositionBlocks(bx, by, bz, yaw, pitch) // block coords (auto-converts)
bot.sendMessage(message)
bot.disconnect()
bot.writeBuffer(buf)

// CPE
bot.sendCpeIdentification(username, verificationKey, extensions)
bot.sendExtInfo(appName, extensionCount)
bot.sendExtEntry(extName, version)
bot.sendCustomBlockSupportLevel(level)
bot.sendPlayerClicked(button, action, yaw, pitch, targetId, tx, ty, tz, face)
bot.sendTwoWayPing(direction, data)
```

<details>
<summary><strong>Full online-mode bot example</strong></summary>

```js
const { ClassiCubeClient, ClassiCubeAccount } = require('classic-node-protocol');

async function main() {
  const account = new ClassiCubeAccount();
  await account.login('BotUsername', 'BotPassword');

  const server = await account.findServer('Survival');

  const bot = new ClassiCubeClient({ connectTimeout: 8000 });
  await bot.connect(server.ip, server.port);

  bot.sendIdentification(account.username, server.mppass);

  bot.on('level', ({ xSize, ySize, zSize }) => {
    console.log(`Joined! Map: ${xSize}×${ySize}×${zSize}`);
    bot.sendMessage('Hello everyone!');
  });

  bot.on('packet', (p) => {
    if (p.name === 'message')    console.log('[Chat]', p.message);
    if (p.name === 'disconnect') console.log('[Kick]', p.reason);
  });
}

main().catch(console.error);
```

</details>

---

## `PacketDecoder`

Streaming packet parser used internally by both `ClassiCubeClient` and `ClientConnection`. Available standalone for custom transports.

```js
const { PacketDecoder } = require('classic-node-protocol');

// direction 'client' → you're the server, parsing packets FROM a client
// direction 'server' → you're the client, parsing packets FROM the server
const decoder = new PacketDecoder('server');

decoder.on('packet', (packet) => console.log(packet.name, packet));
decoder.on('error',  (err)    => console.error('Decode error:', err.message));

// Feed raw TCP data — handles fragmentation & coalescing automatically
socket.on('data', (chunk) => decoder.receive(chunk));

// After CPE negotiation, if ExtPlayerList v2 was agreed upon:
decoder.setExtPlayerListActive(true);
```

---

## `encoder`

Pure functions that return a `Buffer`. No I/O, no side effects.

```js
const { encoder } = require('classic-node-protocol');
// or: const enc = require('classic-node-protocol/encoder');
```

### Coordinate helpers

```js
encoder.toFShort(blockPos)  // float blocks → integer FShort (×32)
encoder.fromFShort(fshort)  // integer FShort → float blocks (÷32)
// e.g. toFShort(5.5) → 176
```

### String helpers

```js
encoder.writeString(buf, offset, str)  // write 64-byte padded ASCII string
encoder.readString(buf, offset)        // read 64-byte ASCII, trim trailing spaces
```

### Server → Client

<details>
<summary><strong>All server-side encoders</strong></summary>

```js
encoder.encodeServerIdentification(serverName, motd, userType)
// userType: 0x00=normal, 0x64=operator → 131 bytes

encoder.encodePing()
// → 1 byte

encoder.encodeLevelInitialize()
// → 1 byte

encoder.encodeLevelDataChunk(chunkData, percentComplete)
// chunkData: Buffer ≤1024 bytes  |  percent: 0–100 → 1028 bytes

encoder.encodeLevelFinalize(xSize, ySize, zSize)
// → 7 bytes

encoder.encodeServerSetBlock(x, y, z, blockType)
// → 8 bytes

encoder.encodeSpawnPlayer(playerId, playerName, x, y, z, yaw, pitch)
// x/y/z: FShort  |  playerId: -1 (0xFF) = spawn as self → 74 bytes

encoder.encodePosition(playerId, x, y, z, yaw, pitch)
// Absolute teleport, FShort → 10 bytes

encoder.encodePositionOrientation(playerId, dx, dy, dz, yaw, pitch)
// Relative (signed byte deltas) + absolute rotation → 7 bytes

encoder.encodePositionUpdate(playerId, dx, dy, dz)
// Relative position only → 5 bytes

encoder.encodeOrientationUpdate(playerId, yaw, pitch)
// → 4 bytes

encoder.encodeDespawnPlayer(playerId)
// → 2 bytes

encoder.encodeServerMessage(playerId, message)
// playerId 0xFF = server message → 66 bytes

encoder.encodeDisconnect(reason)
// → 65 bytes

encoder.encodeUpdateUserType(userType)
// 0x00=normal, 0x64=op → 2 bytes
```

</details>

### Client → Server

<details>
<summary><strong>All client-side encoders</strong></summary>

```js
encoder.encodeClientIdentification(username, verificationKey, unused)
// unused: 0x00 vanilla | 0x42 CPE → 131 bytes

encoder.encodeClientSetBlock(x, y, z, mode, blockType)
// mode: 0=destroy 1=create → 9 bytes

encoder.encodeClientPosition(x, y, z, yaw, pitch)
// Player ID auto-set to 0xFF (self) → 10 bytes

encoder.encodeClientMessage(message)
// → 66 bytes
```

</details>

---

## `protocol` — Constants

```js
const {
  protocol,
  BLOCKS, BLOCK_NAMES,
  BLOCK_MODE, USER_TYPE, CPE_MAGIC,
  CPE_EXTENSIONS, CPE_PACKETS,
} = require('classic-node-protocol');

// or: const protocol = require('classic-node-protocol/protocol');
```

### Block types — all 50 Classic blocks

<details>
<summary><strong>Full BLOCKS reference (IDs 0–49)</strong></summary>

| Constant | ID | Constant | ID |
|---|---|---|---|
| `BLOCKS.AIR` | 0 | `BLOCKS.RED_CLOTH` | 21 |
| `BLOCKS.STONE` | 1 | `BLOCKS.ORANGE_CLOTH` | 22 |
| `BLOCKS.GRASS` | 2 | `BLOCKS.YELLOW_CLOTH` | 23 |
| `BLOCKS.DIRT` | 3 | `BLOCKS.LIME_CLOTH` | 24 |
| `BLOCKS.COBBLESTONE` | 4 | `BLOCKS.GREEN_CLOTH` | 25 |
| `BLOCKS.WOOD_PLANKS` | 5 | `BLOCKS.TEAL_CLOTH` | 26 |
| `BLOCKS.SAPLING` | 6 | `BLOCKS.AQUA_CLOTH` | 27 |
| `BLOCKS.BEDROCK` | 7 | `BLOCKS.CYAN_CLOTH` | 28 |
| `BLOCKS.WATER_FLOWING` | 8 | `BLOCKS.BLUE_CLOTH` | 29 |
| `BLOCKS.WATER` | 9 | `BLOCKS.INDIGO_CLOTH` | 30 |
| `BLOCKS.LAVA_FLOWING` | 10 | `BLOCKS.VIOLET_CLOTH` | 31 |
| `BLOCKS.LAVA` | 11 | `BLOCKS.MAGENTA_CLOTH` | 32 |
| `BLOCKS.SAND` | 12 | `BLOCKS.PINK_CLOTH` | 33 |
| `BLOCKS.GRAVEL` | 13 | `BLOCKS.BLACK_CLOTH` | 34 |
| `BLOCKS.GOLD_ORE` | 14 | `BLOCKS.GRAY_CLOTH` | 35 |
| `BLOCKS.IRON_ORE` | 15 | `BLOCKS.WHITE_CLOTH` | 36 |
| `BLOCKS.COAL_ORE` | 16 | `BLOCKS.DANDELION` | 37 |
| `BLOCKS.LOG` | 17 | `BLOCKS.ROSE` | 38 |
| `BLOCKS.LEAVES` | 18 | `BLOCKS.BROWN_MUSHROOM` | 39 |
| `BLOCKS.SPONGE` | 19 | `BLOCKS.RED_MUSHROOM` | 40 |
| `BLOCKS.GLASS` | 20 | `BLOCKS.GOLD_BLOCK` | 41 |
| | | `BLOCKS.IRON_BLOCK` | 42 |
| | | `BLOCKS.DOUBLE_SLAB` | 43 |
| | | `BLOCKS.SLAB` | 44 |
| | | `BLOCKS.BRICK` | 45 |
| | | `BLOCKS.TNT` | 46 |
| | | `BLOCKS.BOOKSHELF` | 47 |
| | | `BLOCKS.MOSSY_COBBLE` | 48 |
| | | `BLOCKS.OBSIDIAN` | 49 |

</details>

Reverse lookup (ID → name string):

```js
BLOCK_NAMES[2]  // → 'GRASS'
BLOCK_NAMES[49] // → 'OBSIDIAN'
```

### Other constants

```js
BLOCK_MODE.DESTROY  // 0x00
BLOCK_MODE.CREATE   // 0x01

USER_TYPE.NORMAL    // 0x00
USER_TYPE.OP        // 0x64

CPE_MAGIC           // 0x42 — send as `unused` byte to signal CPE support

// Packet ID tables:
protocol.CLIENT_PACKETS         // { IDENTIFICATION: 0x00, SET_BLOCK: 0x05, … }
protocol.SERVER_PACKETS         // { PING: 0x01, LEVEL_INITIALIZE: 0x02, … }
protocol.CPE_PACKETS            // { EXT_INFO: 0x10, EXT_ENTRY: 0x11, … }
protocol.CLIENT_PACKET_SIZES    // { 0x00: 131, 0x05: 9, … }
protocol.SERVER_PACKET_SIZES    // { 0x00: 131, 0x01: 1, … }
protocol.CPE_SERVER_PACKET_SIZES
protocol.CPE_CLIENT_PACKET_SIZES
protocol.STRING_LENGTH          // 64
protocol.PROTOCOL_VERSION       // 7
```

### CPE extension registry

All 26 known extensions with their canonical version numbers:

```js
CPE_EXTENSIONS = {
  ClickDistance:       1,   CustomBlocks:       1,
  HeldBlock:           1,   EmoteFix:           1,
  TextHotKey:          1,   ExtPlayerList:      2,
  EnvColors:           1,   SelectionCuboid:    1,
  BlockPermissions:    1,   ChangeModel:        1,
  EnvMapAppearance:    2,   EnvWeatherType:     1,
  HackControl:         1,   MessageTypes:       1,
  PlayerClick:         1,   LongerMessages:     1,
  FullCP437:           1,   BlockDefinitions:   1,
  BlockDefinitionsExt: 2,   BulkBlockUpdate:    1,
  TextColors:          1,   EnvMapAspect:       1,
  EntityProperty:      1,   ExtEntityPositions: 1,
  TwoWayPing:          1,   InventoryOrder:     1,
}
```

---

## `level` — Maps & Compression

```js
const { level } = require('classic-node-protocol');
// or: const level = require('classic-node-protocol/level');
```

### Coordinate formula

Classic maps use the layout: **`index = x + z×xSize + y×xSize×zSize`** (X fastest, Y = up).

```js
level.blockIndex(x, z, y, xSize, zSize) // → buffer index
```

### Built-in generators

```js
// Classic flat world
level.buildFlatMap(xSize, ySize, zSize, groundY?)
// groundY default: Math.floor(ySize / 2)
// Layout: bedrock at y=0, dirt below surface, grass at surface, air above

// Hollow sphere centered in the map
level.buildSphereMap(xSize, ySize, zSize, radius?, shell?, blockType?)
// radius default: min(dim)/2 - 2  |  shell default: 2  |  block default: STONE

// Checkerboard floor + stone ceiling (good for testing)
level.buildCheckerMap(xSize, ySize, zSize)
// y=0: alternating WHITE_CLOTH / BLACK_CLOTH  |  y=max: STONE  |  rest: AIR

// Custom generator with callback
level.buildMap(xSize, ySize, zSize, (x, y, z) => blockType)
```

**Example — custom world:**

```js
const { level, BLOCKS } = require('classic-node-protocol');

const world = level.buildMap(64, 64, 64, (x, y, z) => {
  if (y === 0)  return BLOCKS.BEDROCK;
  if (y < 30)   return BLOCKS.DIRT;
  if (y === 30) return BLOCKS.GRASS;
  if (y > 50)   return Math.random() < 0.01 ? BLOCKS.STONE : BLOCKS.AIR;
  return BLOCKS.AIR;
});
```

### Server-side compression pipeline

```js
// All-in-one (recommended):
const chunks = await level.prepareLevel(blocks, { level: 6 }); // zlib options
// chunks = [{ chunk: Buffer, percent: number }, ...]

// Manual steps if needed:
const gzipped = await level.compressLevel(blocks, { level: 9 });
const chunks  = level.chunkLevel(gzipped);

// Manual send (what client.sendLevel() does internally):
client.sendLevelInitialize();
for (const { chunk, percent } of chunks) {
  client.sendLevelDataChunk(chunk, percent);
}
client.sendLevelFinalize(xSize, ySize, zSize);
```

### Client-side reassembly

`ClassiCubeClient` handles this automatically and emits `'level'`. For custom use:

```js
const assembler = new level.LevelAssembler();

// on levelInitialize:
assembler.reset();

// on each levelDataChunk:
assembler.push(p.chunkData, p.chunkLength);
console.log(assembler.byteLength, 'bytes buffered');

// on levelFinalize:
const blocks = await assembler.decompress(); // → raw block Buffer
```

### Raw zlib helpers

```js
const compressed = await level.gzip(buffer, { level: 9 });
const original   = await level.gunzip(compressed);
```

---

## `auth` — Authentication & Heartbeat

```js
const {
  ClassiCubeAuth,
  ClassiCubeAccount,
  generateSalt,
  computeMPPass,
  verifyMPPass,
} = require('classic-node-protocol');
```

### Standalone MPPass helpers

```js
// Cryptographically random 16-char alphanumeric salt
const salt = generateSalt();     // 'aB3xQzLm9rTpWkYn'
const salt = generateSalt(32);   // longer

// MPPass = lowercase MD5 hex of (salt + username)
const expected = computeMPPass(salt, 'PlayerName');

// Verify (case-insensitive)
const ok = verifyMPPass(salt, 'PlayerName', submittedMppass); // → boolean
```

### `ClassiCubeAuth` — server-side auth & heartbeat

```js
const auth = new ClassiCubeAuth({
  name:       'My ClassiCube Server', // required — shown in server list
  port:       25565,                  // required
  maxPlayers: 20,
  public:     true,
  software:   'classic-node-protocol',
  web:        false,
  salt:       'optionalCustomSalt',   // auto-generated if omitted
});

auth.salt      // the salt string in use
auth.serverUrl // last known classicube.net play URL | null
```

**Verify a player on join:**

```js
server.on('connection', (client) => {
  client.on('packet', (p) => {
    if (p.name === 'identification') {
      if (!auth.verify(p.username, p.verificationKey)) {
        client.disconnect('Invalid login. Please connect via classicube.net');
        return;
      }
      // ✅ verified — proceed
    }
  });
});
```

**Register on the public server list:**

```js
// Sends first heartbeat immediately, then every 45 s
const { url, error } = await auth.startHeartbeat(() => server.playerCount);

if (error) console.warn('Heartbeat warning:', error);
// Server still works on LAN if port isn't forwarded.
else        console.log('Server URL:', url);
// → https://www.classicube.net/server/play/abc123hash/

// Single manual heartbeat:
await auth.sendHeartbeat(playerCount); // → { url, error }

// Stop the interval:
auth.stopHeartbeat();

// Debug — what MPPass would you expect from a player?
auth.computeExpected('PlayerName'); // → 32-char hex
```

### `ClassiCubeAccount` — bot login

```js
const account = new ClassiCubeAccount();

const { verified } = await account.login('BotUsername', 'BotPassword');
// verified=false if account email isn't confirmed (mppasses will be empty)

account.username  // confirmed username
account.loggedIn  // boolean
account.verified  // boolean
```

**Find and connect to a server:**

```js
const servers = await account.getServers();      // ClassiCubeServerInfo[]
const server  = await account.getServer('hash'); // by hash
const server  = await account.findServer('Survival'); // by name substring
```

**`ClassiCubeServerInfo` shape:**

```js
{
  hash:       'abc123hash',
  name:       'My Survival Server',
  ip:         '123.45.67.89',
  port:       25565,
  mppass:     'd3ad8ee0f...',   // use as verificationKey
  software:   'MCGalaxy',
  players:    5,
  maxPlayers: 32,
  online:     true,
  playUrl:    'https://www.classicube.net/server/play/abc123hash/'
}
```

**Auth error reference:**

| Code | Meaning | Fatal |
|---|---|---|
| `token` | Invalid CSRF token | ✅ |
| `username` | Account not found | ✅ |
| `password` | Wrong password | ✅ |
| `verification` | Email not verified — login succeeds but mppasses are empty | ⚠️ warning |

---

## `cpe` — Classic Protocol Extensions

```js
const { cpe, CPE_EXTENSIONS, CPE_PACKETS } = require('classic-node-protocol');
```

### CPE negotiation flow

CPE is negotiated right after the client's identification packet (when `unused === 0x42`).

**Server side:**

```js
server.on('connection', (client) => {
  client.on('packet', (p) => {
    if (p.name === 'identification' && p.unused === 0x42) {
      client.supportsCpe = true;
      client.sendCpeHandshake('MyServer', ['EnvColors', 'HackControl']);
    }
    if (p.name === 'extEntry') {
      client.extensions.add(p.extName);
    }
  });
});
```

**Client side:**

```js
bot.sendCpeIdentification('BotName', '-', ['EnvColors', 'HackControl']);
// pass no extension array to advertise ALL known extensions
```

### Extension examples

<details>
<summary><strong>ClickDistance</strong></summary>

```js
client.sendSetClickDistance(160) // 5 blocks (default = 160 units)
client.sendSetClickDistance(32)  // 1 block range
```

</details>

<details>
<summary><strong>HeldBlock</strong></summary>

```js
client.sendHoldThis(BLOCKS.STONE, 0) // give stone, allow switching
client.sendHoldThis(BLOCKS.STONE, 1) // lock to stone
client.sendHoldThis(0, 0)            // hide hand
```

</details>

<details>
<summary><strong>EnvColors</strong></summary>

```js
const { ENV_COLOR } = require('classic-node-protocol');
// ENV_COLOR: SKY=0  CLOUD=1  FOG=2  AMBIENT=3  SUNLIGHT=4

client.sendEnvSetColor(ENV_COLOR.SKY,      135, 206, 235); // sky blue
client.sendEnvSetColor(ENV_COLOR.CLOUD,    255, 255, 255);
client.sendEnvSetColor(ENV_COLOR.FOG,      180, 180, 180);
client.sendEnvSetColor(ENV_COLOR.AMBIENT,   40,  40,  40);
client.sendEnvSetColor(ENV_COLOR.SUNLIGHT, 255, 255, 200);
client.sendEnvSetColor(ENV_COLOR.SKY, -1, -1, -1);        // reset to default
```

</details>

<details>
<summary><strong>SelectionCuboid</strong></summary>

```js
client.sendMakeSelection(
  0,           // selectionId 0–255
  'My Region', // label shown on hover
  0, 0, 0,     // start corner (x1, y1, z1)
  10, 10, 10,  // end corner   (x2, y2, z2)
  255, 0, 0,   // color RGB
  128          // alpha 0–255
);
client.sendRemoveSelection(0);
```

</details>

<details>
<summary><strong>HackControl</strong></summary>

> ⚠️ ClassiCube only processes this packet after the first `LevelDataChunk` is sent.

```js
client.sendHackControl(
  1,  // flying    (1=allowed, 0=denied)
  1,  // noclip
  1,  // speeding
  1,  // spawn control
  1,  // third-person view
  -1  // jump height in player-space units (-1 = default)
);
```

</details>

<details>
<summary><strong>EnvWeatherType</strong></summary>

```js
const { WEATHER } = require('classic-node-protocol');
// WEATHER.SUNNY=0  WEATHER.RAINING=1  WEATHER.SNOWING=2

client.sendEnvSetWeatherType(WEATHER.RAINING);
```

</details>

<details>
<summary><strong>EnvMapAspect</strong></summary>

```js
const { MAP_ENV_PROPERTY } = require('classic-node-protocol');
// SIDE_BLOCK=0 EDGE_BLOCK=1 EDGE_HEIGHT=2 CLOUD_HEIGHT=3
// MAX_FOG=4    CLOUD_SPEED=5 WEATHER_SPEED=6 WEATHER_FADE=7
// EXP_FOG=8   SIDE_OFFSET=9

client.sendSetMapEnvUrl('https://example.com/textures.zip');
client.sendSetMapEnvUrl(''); // reset to default

client.sendSetMapEnvProperty(MAP_ENV_PROPERTY.CLOUD_HEIGHT, 128);
client.sendSetMapEnvProperty(MAP_ENV_PROPERTY.EDGE_HEIGHT,  32);
client.sendSetMapEnvProperty(MAP_ENV_PROPERTY.MAX_FOG,      512);
```

</details>

<details>
<summary><strong>BulkBlockUpdate</strong></summary>

```js
// Update up to 256 blocks in a single packet
client.sendBulkBlockUpdate([
  { index: level.blockIndex(5, 3, 10, 64, 64), blockType: BLOCKS.GOLD_BLOCK },
  { index: level.blockIndex(6, 3, 10, 64, 64), blockType: BLOCKS.GOLD_BLOCK },
  { index: level.blockIndex(7, 3, 10, 64, 64), blockType: BLOCKS.IRON_BLOCK },
]);
```

</details>

<details>
<summary><strong>ExtPlayerList</strong></summary>

```js
// Add to tab list
client.sendExtAddPlayerName(nameId, 'PlayerName', '&aPlayerName', 'Admins', 0);

// Spawn with custom skin
client.sendExtAddEntity2(entityId, 'PlayerName', 'SkinName', x, y, z, yaw, pitch);

// Remove from tab list
client.sendExtRemovePlayerName(nameId);
```

</details>

<details>
<summary><strong>BlockDefinitions</strong></summary>

```js
client.sendDefineBlock({
  blockId:        50,
  name:           'Glowstone',
  solidity:       1,   // 0=walk-through 1=solid 2=liquid
  movementSpeed:  128, // 0–255 (128 = normal)
  topTexture:     18,
  sideTexture:    18,
  bottomTexture:  18,
  transmitsLight: 0,
  walkSound:      1,
  fullBright:     1,   // 1 = glows
  shape:          0,
  blockDraw:      0,   // 0=opaque 1=transparent 2=translucent 3=gas
  fogDensity:     0,
  fogR: 0, fogG: 0, fogB: 0,
});

client.sendRemoveBlockDefinition(50);
```

</details>

<details>
<summary><strong>EntityProperty</strong></summary>

```js
const { ENTITY_PROPERTY } = require('classic-node-protocol');
// ROT_X=0 ROT_Y=1 ROT_Z=2  SCALE_X=3 SCALE_Y=4 SCALE_Z=5

client.sendSetEntityProperty(entityId, ENTITY_PROPERTY.SCALE_X, 64); // 2× scale
client.sendSetEntityProperty(entityId, ENTITY_PROPERTY.ROT_Z,   45);
```

</details>

<details>
<summary><strong>TwoWayPing</strong></summary>

```js
// Server pings → client echoes back
client.sendTwoWayPing(0, 1234); // direction=0 server→client

bot.on('packet', (p) => {
  if (p.name === 'twoWayPing' && p.direction === 0) {
    bot.sendTwoWayPing(1, p.data); // echo back
  }
});
```

</details>

<details>
<summary><strong>TextHotKey / ChangeModel / BlockPermissions / InventoryOrder / TextColors</strong></summary>

```js
// TextHotKey
client.sendSetTextHotKey('Say Hello', '/say Hello\n', 0x48, 0);
// keyMods: 0=none 1=Ctrl 2=Shift 4=Alt (combinable)

// ChangeModel
client.sendChangeModel(entityId, 'chicken');
client.sendChangeModel(entityId, 'humanoid'); // reset

// BlockPermissions
client.sendSetBlockPermission(BLOCKS.BEDROCK, 0, 0); // no place, no break
client.sendSetBlockPermission(BLOCKS.GRASS,   1, 1); // allow both

// InventoryOrder
client.sendSetInventoryOrder(0, BLOCKS.STONE);
client.sendSetInventoryOrder(1, BLOCKS.GRASS);

// TextColors — redefine a color code
client.sendSetTextColor(0x63, 255, 20, 147, 255); // &c → hot pink
```

</details>

### Low-level CPE encoding

All encoders/decoders are also directly available on the `cpe` module:

```js
cpe.encodeExtInfo(appName, extensionCount)       // → 67-byte Buffer
cpe.encodeExtEntry(extName, version)             // → 69-byte Buffer
cpe.buildExtensionHandshake(appName, extensions) // → Buffer[] (ready to write)

cpe.decodeExtInfo(buf)   // → { name: 'extInfo', appName, extensionCount }
cpe.decodeExtEntry(buf)  // → { name: 'extEntry', extName, version }
```

---

## `jugadorUUID`

Generates deterministic MD5-based (v3-style) UUIDs from usernames, matching ClassiCube's player identification format.

```js
const { jugadorUUID } = require('classic-node-protocol');

jugadorUUID.generarUUID('PlayerName')
// → '550e8400-e29b-31d4-a716-446655440000'
// Same input always → same UUID

jugadorUUID.validarUUID('550e8400-e29b-31d4-a716-446655440000')
// → true

jugadorUUID.uuidToHex('550e8400-e29b-31d4-a716-446655440000')
// → '550e8400e29b31d4a716446655440000'

jugadorUUID.hexToUUID('550e8400e29b31d4a716446655440000')
// → '550e8400-e29b-31d4-a716-446655440000'
```

---

## TypeScript

Full type definitions ship at `types/index.d.ts`. No `@types/` package needed.

```ts
import {
  createServer,
  createClient,
  ClassiCubeServer,
  ClassiCubeClient,
  ClientConnection,
  PacketDecoder,
  ClassiCubeAuth,
  ClassiCubeAccount,
  ClassiCubeServerInfo,
  BLOCKS,
  BLOCK_NAMES,
  BLOCK_MODE,
  USER_TYPE,
  CPE_MAGIC,
  CPE_EXTENSIONS,
  ENV_COLOR,
  WEATHER,
  MAP_ENV_PROPERTY,
  ENTITY_PROPERTY,
} from 'classic-node-protocol';

const server: ClassiCubeServer = createServer({ port: 25565 });

server.on('connection', async (client: ClientConnection) => {
  const blocks = level.buildFlatMap(64, 64, 64);
  await client.sendLevel(blocks, 64, 64, 64);
});
```

---

## Protocol Reference

### FShort — fixed-point coordinates

```
1 block = 32 units
fshort  = blockPosition × 32
block   = fshort ÷ 32
```

Methods like `spawnAt()` and `sendPositionBlocks()` convert automatically. All raw encoder functions (`encodeSpawnPlayer`, `encodePosition`, etc.) expect FShort values.

### Packet byte sizes

| Packet | ID | Size | Direction |
|---|---|---|---|
| Identification | `0x00` | 131 | Both |
| Ping | `0x01` | 1 | S→C |
| LevelInitialize | `0x02` | 1 | S→C |
| LevelDataChunk | `0x03` | 1028 | S→C |
| LevelFinalize | `0x04` | 7 | S→C |
| SetBlock (client) | `0x05` | 9 | C→S |
| SetBlock (server) | `0x06` | 8 | S→C |
| SpawnPlayer | `0x07` | 74 | S→C |
| Position | `0x08` | 10 | Both |
| PositionOrientation | `0x09` | 7 | S→C |
| PositionUpdate | `0x0A` | 5 | S→C |
| OrientationUpdate | `0x0B` | 4 | S→C |
| DespawnPlayer | `0x0C` | 2 | S→C |
| Message | `0x0D` | 66 | Both |
| Disconnect | `0x0E` | 65 | S→C |
| UpdateUserType | `0x0F` | 2 | S→C |

### Map memory layout

```
index = x + (z × xSize) + (y × xSize × zSize)

  X → East       (increases East)
  Y → Up         (increases upward)
  Z → South      (increases South)

Spawn tip: place players at y = groundY + 1 (one block above the surface)
```

---

<div align="center">

**MIT License** · © Rosendo Torres  
[npm](https://www.npmjs.com/package/classic-node-protocol) · [GitHub](https://github.com/rosendotorre/classic-node-protocol) · [Issues](https://github.com/rosendotorre/classic-node-protocol/issues)

</div>

