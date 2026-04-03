
<div align="center">

# classic-node-protocol

**Minecraft Classic v7 / ClassiCube protocol library for Node.js**

Zero-dependency · Full client & server · CPE support · TypeScript types

[![Node.js](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org)
[![Version](https://img.shields.io/badge/version-2.2.5-blue)](#)
[![License](https://img.shields.io/badge/license-see%20LICENSE-lightgrey)](#)

</div>

---

## Table of Contents

- [Install](#install)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [API Reference](#api-reference)
  - [createServer(options?)](#createserveroptions)
  - [createClient(options?)](#createclientoptions)
  - [ClassiCubeServer](#classicubeserver)
  - [ClassiCubeClient](#classicubeclient)
  - [ClientConnection](#clientconnection)
  - [PacketDecoder](#packetdecoder)
  - [encoder](#encoder)
  - [level](#level)
  - [cpe](#cpe)
  - [jugadorUUID](#jugadoruuid)
  - [protocol constants](#protocol-constants)
- [CPE System](#cpe-system)
- [FShort Coordinates](#fshort-coordinates)
- [Blocks Reference](#blocks-reference)
- [Examples](#examples)

---

## Install

```bash
npm install classic-node-protocol
```

Requires **Node.js >= 18**. No external dependencies.

---

## Quick Start

**Server:**
```js
const { createServer, level } = require('classic-node-protocol');

const srv = createServer({ port: 25565 });
const blocks = level.buildFlatMap(64, 32, 64);

srv.on('connection', async client => {
  client.on('packet', async p => {
    if (p.name === 'identification') {
      client.username = p.username;
      client.sendIdentification('My Server', 'Welcome!');
      await client.sendLevel(blocks, 64, 32, 64);
      client.spawnAt(0xFF, client.username, 32, 17, 32);
    }
  });
});
```

**Client / Bot:**
```js
const { createClient } = require('classic-node-protocol');

const bot = createClient({ host: 'localhost', port: 25565 });
bot.on('connect', () => bot.sendIdentification('MyBot'));
bot.on('packet',  p  => console.log(p));
bot.on('level',   l  => console.log('Map received:', l.xSize, l.ySize, l.zSize));
```

---

## Architecture

```
index.js            <- Entry point, re-exports everything
lib/
  protocol.js       <- Packet IDs, sizes, block IDs, CPE extension list
  encoder.js        <- Build raw Buffers for every packet type
  decoder.js        <- Streaming packet parser (PacketDecoder)
  client.js         <- ClassiCubeClient  (TCP client)
  server.js         <- ClassiCubeServer + ClientConnection  (TCP server)
  level.js          <- Map compression / decompression / generators
  cpe.js            <- All CPE packet encoders & decoders
UUID/
  jugadorUUID.js    <- MD5-based player UUID helpers
types/
  index.d.ts        <- TypeScript declarations
```

Data flow:
```
Socket bytes --> PacketDecoder.receive() --> emit('packet', {name, ...fields})
                                                    ^
              encoder.*() --> Buffer --> socket.write()
```

---

## API Reference

---

### `createServer(options?)`

Creates and returns a `ClassiCubeServer`.
If `port` is given, starts listening immediately (unless `autoListen: false`).

```js
const { createServer } = require('classic-node-protocol');
const srv = createServer({ port: 25565, maxClients: 20 });
```

| Option         | Type      | Default     | Description                                                   |
|----------------|-----------|-------------|---------------------------------------------------------------|
| `port`         | `number`  | —           | TCP port to listen on. Omit to call `listen()` manually.      |
| `host`         | `string`  | `'0.0.0.0'` | Network interface to bind.                                    |
| `autoListen`   | `boolean` | `true`      | Set `false` to delay `listen()`.                              |
| `maxClients`   | `number`  | `0`         | Max simultaneous connections. `0` = unlimited.                |
| `pingInterval` | `number`  | `0`         | Milliseconds between automatic pings to all clients. `0` = off. |

---

### `createClient(options?)`

Creates and returns a `ClassiCubeClient`.
If both `host` and `port` are given, connects immediately.

```js
const { createClient } = require('classic-node-protocol');
const bot = createClient({ host: 'play.example.com', port: 25565 });
```

| Option           | Type     | Default | Description                                          |
|------------------|----------|---------|------------------------------------------------------|
| `host`           | `string` | —       | Server hostname or IP.                               |
| `port`           | `number` | —       | Server TCP port.                                     |
| `connectTimeout` | `number` | `10000` | ms before the TCP connect attempt times out.         |
| `pingInterval`   | `number` | `0`     | ms between automatic ping emissions. `0` = off.      |

---

### `ClassiCubeServer`

TCP server that manages multiple client connections.

#### Events

| Event           | Arguments                  | When it fires                                        |
|-----------------|----------------------------|------------------------------------------------------|
| `'listening'`   | `{ port, host }`           | Server is bound and ready to accept connections.     |
| `'connection'`  | `ClientConnection`         | A new client connected.                              |
| `'disconnect'`  | `ClientConnection`         | A client disconnected cleanly.                       |
| `'clientError'` | `ClientConnection, Error`  | A per-client socket error occurred.                  |
| `'error'`       | `Error`                    | A server-level TCP error occurred.                   |

#### Properties

| Property      | Type                             | Description                                              |
|---------------|----------------------------------|----------------------------------------------------------|
| `clients`     | `Map<number, ClientConnection>`  | All currently connected clients, keyed by numeric ID.    |
| `playerCount` | `number`                         | Number of active connections right now.                  |

#### Methods

```js
// Start listening. Returns a Promise that resolves when bound.
await srv.listen(25565, '0.0.0.0');

// Stop accepting new connections. Returns a Promise.
await srv.close();

// Disconnect every client with a reason, then close the server.
await srv.shutdown('Server is restarting');

// Send a raw Buffer to every connected client.
srv.broadcast(buffer);

// Broadcast a chat message from the server (senderId = 0xFF).
srv.broadcastMessage('Hello everyone!');

// Send a raw Buffer to every client except the one with the given ID.
// Useful to relay player movement without echoing back to the sender.
srv.broadcastExcept(clientId, buffer);

// Relay a chat message from one player to all others.
// Uses the sender's client.id as the in-game sender ID.
srv.relayMessage(fromClientId, 'Hi all');

// Send a vanilla Ping packet to every connected client.
srv.pingAll();
```

---

### `ClassiCubeClient`

TCP client that connects to a Classic / ClassiCube server.

#### Events

| Event             | Arguments                            | When it fires                                             |
|-------------------|--------------------------------------|-----------------------------------------------------------|
| `'connect'`       | —                                    | TCP handshake complete.                                   |
| `'end'`           | —                                    | Connection closed.                                        |
| `'packet'`        | `object`                             | A packet was received and parsed.                         |
| `'level'`         | `{ blocks, xSize, ySize, zSize }`    | Full map received and gunzip'd. `blocks` is a raw Buffer. |
| `'levelProgress'` | `number` (0–100)                     | Map download progress percent.                            |
| `'error'`         | `Error`                              | Network or parse error.                                   |

#### Connection methods

```js
// Open the TCP connection. Returns a Promise.
// Rejects if the connection times out or fails.
await bot.connect('localhost', 25565);

// Close the socket immediately.
bot.disconnect();
```

#### Vanilla send methods

```js
// 0x00 - Send player identification.
// verificationKey: multiplayer hash from ClassiCube API, or '-' for offline mode.
// unused: 0x00 for vanilla, 0x42 (CPE_MAGIC) to signal CPE support.
bot.sendIdentification(username, verificationKey = '-', unused = 0x00);

// 0x05 - Place or break a block.
// mode: 0 = destroy, 1 = create  (use BLOCK_MODE constants).
bot.sendSetBlock(x, y, z, mode, blockType);

// 0x08 - Send position and orientation (FShort coordinates).
bot.sendPosition(x, y, z, yaw, pitch);

// Same as sendPosition but accepts block coordinates and auto-converts to FShort.
bot.sendPositionBlocks(bx, by, bz, yaw, pitch);

// 0x0D - Send a chat message (max 64 chars).
bot.sendMessage('Hello!');
```

#### CPE send methods

```js
// Send full CPE handshake: identification with magic byte + ExtInfo + ExtEntry packets.
// Call this INSTEAD of sendIdentification() when you want CPE support.
// extensions defaults to all 26 known extensions.
bot.sendCpeIdentification(username, verificationKey = '-', extensions = [...]);

// Send an ExtInfo packet manually (declares how many ExtEntry follow).
bot.sendExtInfo(appName, extensionCount);

// Send one ExtEntry packet manually (declares support for one extension).
bot.sendExtEntry(extName, version);

// Reply to the server's CustomBlockSupportLevel packet.
bot.sendCustomBlockSupportLevel(level = 1);

// Notify the server that the player clicked.
// button: 0=Left 1=Right 2=Middle | action: 0=Press 1=Release
// targetId: entity ID (-1 if none) | targetFace: 0-5 or 6=none
bot.sendPlayerClicked(button, action, yaw, pitch, targetId, targetX, targetY, targetZ, targetFace);

// Reply to a TwoWayPing from the server.
// direction: 0=server-to-client echo, 1=client-initiated
bot.sendTwoWayPing(direction, data);
```

#### Low-level

```js
// Write a raw Buffer directly to the socket.
bot.writeBuffer(buffer);
```

---

### `ClientConnection`

Represents one connected player on the server side.
Created automatically by `ClassiCubeServer`; received in the `'connection'` event.

#### Properties

| Property        | Type            | Description                                                    |
|-----------------|-----------------|----------------------------------------------------------------|
| `id`            | `number`        | Unique numeric ID assigned by the server at connect time.      |
| `username`      | `string\|null`  | Player name. Set by your app after the `identification` packet. |
| `state`         | `string`        | Connection phase: `'login'` → `'level'` → `'play'`.           |
| `data`          | `object`        | Free-use object for your application's session data.           |
| `extensions`    | `Set<string>`   | CPE extension names mutually agreed on with this client.       |
| `cpeReady`      | `boolean`       | `true` once CPE negotiation is complete.                       |
| `supportsCpe`   | `boolean`       | `true` if the client sent `unused = 0x42` in identification.   |
| `remoteAddress` | `string`        | Client IP address.                                             |
| `remotePort`    | `number`        | Client port.                                                   |
| `isConnected`   | `boolean`       | `false` if the socket has been destroyed.                      |

#### Events

| Event      | Arguments | When it fires                        |
|------------|-----------|--------------------------------------|
| `'packet'` | `object`  | A packet arrived from this client.   |
| `'end'`    | —         | This client disconnected.            |
| `'error'`  | `Error`   | Socket or parse error.               |

#### Vanilla send methods

```js
// 0x00 - Identify the server to this client.
// userType: 0x00 = normal, 0x64 = operator (enables block deletion in vanilla client).
client.sendIdentification(serverName, motd, userType = 0x00);

// 0x01 - Send a keep-alive ping.
client.sendPing();

// -- Map transmission ---------------------------------------------------------

// Send the full map in one async call (compresses, chunks, sends all 3 phases).
// zlibOpts: optional zlib options, e.g. { level: 6 }
await client.sendLevel(blocks, xSize, ySize, zSize, zlibOpts = {});

// Or send the three phases manually:
client.sendLevelInitialize();                       // 0x02 - Begin map; sets state = 'level'
client.sendLevelDataChunk(chunkBuffer, percent);    // 0x03 - One <=1024-byte gzip chunk
client.sendLevelFinalize(xSize, ySize, zSize);      // 0x04 - End of map; sets state = 'play'

// -- Block changes ------------------------------------------------------------

// 0x06 - Notify this client that a block changed.
client.sendSetBlock(x, y, z, blockType);

// -- Player management --------------------------------------------------------

// 0x07 - Spawn a player entity visible to this client (FShort coords).
// playerId: -1 / 0xFF = spawn as this client's own player.
client.sendSpawnPlayer(playerId, playerName, x, y, z, yaw = 0, pitch = 0);

// Same as sendSpawnPlayer but accepts block coords and auto-converts to FShort.
client.spawnAt(playerId, playerName, bx, by, bz, yaw = 0, pitch = 0);

// 0x08 - Teleport a player to an absolute position (FShort coords).
client.sendPosition(playerId, x, y, z, yaw = 0, pitch = 0);

// 0x09 - Move a player by a relative delta AND set absolute rotation.
// dx/dy/dz: signed byte deltas in FShort units (32 units = 1 block).
client.sendPositionOrientation(playerId, dx, dy, dz, yaw, pitch);

// 0x0A - Move a player by a relative delta only (no rotation change).
client.sendPositionUpdate(playerId, dx, dy, dz);

// 0x0B - Update a player's rotation only (no position change).
client.sendOrientationUpdate(playerId, yaw, pitch);

// 0x0C - Remove a player entity from this client's view.
client.sendDespawnPlayer(playerId);

// -- Chat ---------------------------------------------------------------------

// 0x0D - Send a chat line. senderId 0xFF = server message.
client.sendMessage(senderId, message);

// Convenience wrapper: sends with senderId = 0xFF.
client.sendServerMessage(message);

// -- User type ----------------------------------------------------------------

// 0x0F - Change this client's operator status.
// userType: USER_TYPE.NORMAL (0x00) or USER_TYPE.OP (0x64)
client.sendUpdateUserType(userType);

// Convenience wrapper.
client.setOperator(isOp = true);

// -- Disconnect ---------------------------------------------------------------

// 0x0E - Send a disconnect reason and destroy the socket.
client.disconnect(reason = 'Disconnected');

// -- Low-level ----------------------------------------------------------------
client.writeBuffer(buffer);
```

#### CPE send methods

```js
// Send full CPE handshake to this client.
// Call immediately after detecting supportsCpe = true, BEFORE sendIdentification().
// extensions defaults to all 26 known CPE extensions.
client.sendCpeHandshake(appName = 'classic-node-protocol', extensions = [...]);

// Send ExtInfo packet manually.
client.sendExtInfo(appName, extensionCount);

// Send one ExtEntry packet manually.
client.sendExtEntry(extName, version);

// ClickDistance - set how far this client can interact with blocks.
// distance in FShort units (160 = 5 blocks, the default).
client.sendSetClickDistance(distance);

// CustomBlocks - tell the client which extra block IDs are supported.
// supportLevel: 1 = IDs 50-65 enabled.
client.sendCustomBlockSupportLevel(level = 1);

// HeldBlock - force this client to hold a specific block.
// preventChange: 0 = player can switch, 1 = locked.
client.sendHoldThis(blockToHold, preventChange = 0);

// TextHotKey - define a client-side keyboard shortcut.
// action: text to type; append '\n' to auto-send.
// keyMods: 0=None 1=Ctrl 2=Shift 4=Alt (bitwise OR to combine).
client.sendSetTextHotKey(label, action, keyCode, keyMods = 0);

// ExtPlayerList - add an entry to the tab-list.
// nameId must be unique; groupRank controls sort order within groupName.
client.sendExtAddPlayerName(nameId, playerName, listName, groupName, groupRank = 0);

// ExtPlayerList - remove a tab-list entry.
client.sendExtRemovePlayerName(nameId);

// ExtPlayerList v2 - spawn an entity with a separate skin name or skin URL.
// skinName: player name OR full https:// URL to a .png skin file.
client.sendExtAddEntity2(entityId, inGameName, skinName, x, y, z, yaw = 0, pitch = 0);

// EnvColors - change one sky/fog/light color variable.
// variable: use ENV_COLOR constants. Pass r=-1,g=-1,b=-1 to reset to default.
client.sendEnvSetColor(variable, r, g, b);

// SelectionCuboid - draw a colored highlight box around a region.
// x1/y1/z1 and x2/y2/z2 are block coords. a = opacity (0-255).
client.sendMakeSelection(selectionId, label, x1, y1, z1, x2, y2, z2, r, g, b, a);

// SelectionCuboid - remove a highlighted region.
client.sendRemoveSelection(selectionId);

// BlockPermissions - allow or deny placing/breaking a specific block type.
// allowPlace / allowDestroy: 1 = allowed, 0 = denied.
client.sendSetBlockPermission(blockType, allowPlace, allowDestroy);

// ChangeModel - change this entity's visual model.
// modelName examples: 'humanoid', 'creeper', 'chicken', 'spider', 'zombie'
client.sendChangeModel(entityId, modelName);

// EnvMapAppearance (legacy v2) - set world texture, border blocks, water level.
// Prefer sendSetMapEnvUrl + sendSetMapEnvProperty for modern clients.
client.sendSetMapEnvAppearance(textureUrl, sideBlock, edgeBlock, sideLevel);

// EnvWeatherType - change the weather.
// Use WEATHER constants: WEATHER.SUNNY / WEATHER.RAINING / WEATHER.SNOWING
client.sendEnvSetWeatherType(weatherType);

// HackControl - enable or disable individual client movement abilities.
// All flags: 0 = disabled, 1 = enabled.
// jumpHeight: FShort units (32/block); -1 resets to default.
// WARNING: ClassiCube ignores this packet before the first LevelDataChunk.
client.sendHackControl(flying, noClip, speeding, spawnControl, thirdPersonView, jumpHeight = -1);

// BlockDefinitions - define a new custom block (v1, blockId should be 50-65).
client.sendDefineBlock(def);

// BlockDefinitions - define a custom block with extended hitbox + fog (v2).
client.sendDefineBlockExt(def);

// BlockDefinitions - remove a previously defined custom block.
client.sendRemoveBlockDefinition(blockId);

// BulkBlockUpdate - update up to 256 blocks in a single packet.
// updates: Array of { index: number, blockType: number }
// index = x + z*xSize + y*xSize*zSize
client.sendBulkBlockUpdate(updates);

// TextColors - redefine the RGBA color that a color-code character maps to.
// code: ASCII code of the char (e.g. 0x31 for '1', 0x61 for 'a').
client.sendSetTextColor(code, r, g, b, a = 255);

// EnvMapAspect - set the texture pack URL.
client.sendSetMapEnvUrl(textureUrl);

// EnvMapAspect - set one map environment property.
// Use MAP_ENV_PROPERTY constants for the property argument.
client.sendSetMapEnvProperty(property, value);

// EntityProperty - set rotation or scale of an entity.
// Use ENTITY_PROPERTY constants for propertyType.
// value is a signed Int32 (fixed-point angle or scale factor).
client.sendSetEntityProperty(entityId, propertyType, value);

// TwoWayPing - send a ping that the client must echo back.
// direction: 0 = server-initiated, 1 = client-initiated.
client.sendTwoWayPing(direction, data);

// InventoryOrder - set which block appears in a given hotbar slot.
client.sendSetInventoryOrder(order, blockType);
```

---

### `PacketDecoder`

Streaming packet parser. Extends `EventEmitter`.
Used internally by both `ClassiCubeClient` and `ClientConnection`, but can be used standalone.

```js
const { PacketDecoder } = require('classic-node-protocol');

// direction:
//   'client' -> parsing packets SENT BY the client (server perspective)
//   'server' -> parsing packets SENT BY the server (client perspective)
const decoder = new PacketDecoder('client');

decoder.on('packet', packet => console.log(packet));
decoder.on('error',  err    => console.error(err));

// Feed raw socket data. Handles partial and multi-packet chunks automatically.
socket.on('data', data => decoder.receive(data));

// Call after CPE negotiation if ExtPlayerList v2 was agreed on.
// Required to correctly disambiguate packet ID 0x1D
// (ExtAddEntity2 and ChangeModel share the same ID).
decoder.setExtPlayerListActive(true);
```

Parsed packets are plain JS objects. Every packet has at minimum:
- `name` — human-readable name e.g. `'identification'`, `'setBlock'`, `'message'`
- `id`   — numeric packet ID

Unknown packet IDs emit `'error'` and flush the internal buffer.

---

### `encoder`

Low-level functions that return `Buffer` objects ready to write to a socket.

```js
const encoder = require('classic-node-protocol/encoder');
// or
const { encoder } = require('classic-node-protocol');
```

#### String and coordinate helpers

```js
// Write a 64-byte space-padded ASCII string into buf at offset.
// Returns the next offset position after the string.
encoder.writeString(buf, offset, str);    // -> number

// Read and right-trim a 64-byte ASCII string from buf at offset.
encoder.readString(buf, offset);          // -> string

// Convert block position (float or int) to FShort wire format (* 32).
encoder.toFShort(5);      // -> 160
encoder.toFShort(3.5);    // -> 112

// Convert FShort wire value back to block position.
encoder.fromFShort(160);  // -> 5
encoder.fromFShort(112);  // -> 3.5
```

#### Server -> Client encoders

```js
// 0x00 - Server identification.
// userType: 0x00 = normal, 0x64 = operator.
encoder.encodeServerIdentification(serverName, motd, userType = 0x00);  // -> Buffer (131 bytes)

// 0x01 - Ping / keep-alive.
encoder.encodePing();                                                     // -> Buffer (1 byte)

// 0x02 - Begin map transmission.
encoder.encodeLevelInitialize();                                          // -> Buffer (1 byte)

// 0x03 - One chunk of gzip'd map data.
// chunkData: up to 1024 bytes. percentComplete: 0-100.
encoder.encodeLevelDataChunk(chunkData, percentComplete);                 // -> Buffer (1028 bytes)

// 0x04 - End of map transmission with world dimensions.
encoder.encodeLevelFinalize(xSize, ySize, zSize);                        // -> Buffer (7 bytes)

// 0x06 - Notify a client that one block changed.
encoder.encodeServerSetBlock(x, y, z, blockType);                        // -> Buffer (8 bytes)

// 0x07 - Spawn a player entity. x/y/z are FShort. playerId 0xFF = self.
encoder.encodeSpawnPlayer(playerId, playerName, x, y, z, yaw, pitch);   // -> Buffer (74 bytes)

// 0x08 - Absolute position + orientation (teleport). x/y/z are FShort.
encoder.encodePosition(playerId, x, y, z, yaw, pitch);                  // -> Buffer (10 bytes)

// 0x09 - Relative position delta + absolute rotation.
// dx/dy/dz: signed byte deltas in FShort units.
encoder.encodePositionOrientation(playerId, dx, dy, dz, yaw, pitch);    // -> Buffer (7 bytes)

// 0x0A - Relative position delta only.
encoder.encodePositionUpdate(playerId, dx, dy, dz);                      // -> Buffer (5 bytes)

// 0x0B - Rotation update only.
encoder.encodeOrientationUpdate(playerId, yaw, pitch);                   // -> Buffer (4 bytes)

// 0x0C - Remove a player entity from the client's world.
encoder.encodeDespawnPlayer(playerId);                                    // -> Buffer (2 bytes)

// 0x0D - Chat message. playerId 0xFF = server-sent message.
encoder.encodeServerMessage(playerId, message);                          // -> Buffer (66 bytes)

// 0x0E - Disconnect the client with a reason string.
encoder.encodeDisconnect(reason);                                         // -> Buffer (65 bytes)

// 0x0F - Change the client's operator status.
// userType: 0x00 = normal, 0x64 = operator.
encoder.encodeUpdateUserType(userType);                                   // -> Buffer (2 bytes)
```

#### Client -> Server encoders

```js
// 0x00 - Player identification.
// unused: 0x00 = vanilla, 0x42 = CPE magic byte.
encoder.encodeClientIdentification(username, verificationKey = '-', unused = 0x00);  // -> Buffer (131 bytes)

// 0x05 - Place or break a block. mode: 0 = destroy, 1 = create.
encoder.encodeClientSetBlock(x, y, z, mode, blockType);                  // -> Buffer (9 bytes)

// 0x08 - Send player position + orientation. x/y/z are FShort.
// Player ID is always 0xFF on the client side.
encoder.encodeClientPosition(x, y, z, yaw, pitch);                      // -> Buffer (10 bytes)

// 0x0D - Send a chat message.
encoder.encodeClientMessage(message);                                     // -> Buffer (66 bytes)
```

---

### `level`

Map compression, decompression, and world generation utilities.

```js
const level = require('classic-node-protocol/level');
// or
const { level } = require('classic-node-protocol');
```

#### Server-side pipeline

```js
// Prepend the Classic 4-byte block-count header and gzip-compress the block array.
// opts: optional zlib.gzip options e.g. { level: 6 }
const gzipped = await level.compressLevel(blocks, opts = {});   // -> Buffer

// Split a gzip Buffer into <=1024-byte chunks for LevelDataChunk packets.
// Returns Array<{ chunk: Buffer, percent: number }>.
const chunks = level.chunkLevel(gzipped);

// Convenience: compress + chunk in one call.
const chunks = await level.prepareLevel(blocks, opts = {});     // -> Array<{ chunk, percent }>
```

#### Map generators

```js
// Build a block array from a callback function.
// fn(x, y, z) must return a block type ID (0 = air).
// Memory layout: X varies fastest, then Z, then Y (Y is "up").
const blocks = level.buildMap(xSize, ySize, zSize, (x, y, z) => {
  return y === 0 ? BLOCKS.STONE : BLOCKS.AIR;
});

// Built-in flat world: bedrock at y=0, dirt layer, grass surface, air above.
// groundY defaults to Math.floor(ySize / 2).
const blocks = level.buildFlatMap(256, 64, 256);
const blocks = level.buildFlatMap(256, 64, 256, groundY = 20);

// Hollow sphere centred in the map.
// radius defaults to min(dim)/2 - 2. shell = shell thickness in blocks.
const blocks = level.buildSphereMap(64, 64, 64);
const blocks = level.buildSphereMap(64, 64, 64, radius = 28, shell = 2, blockType = BLOCKS.STONE);

// Checkerboard floor (y=0) + stone ceiling (y=ySize-1), air in between.
const blocks = level.buildCheckerMap(64, 32, 64);
```

#### Block index formula

```js
// Classic layout: index = x + z*xSize + y*xSize*zSize
const idx = level.blockIndex(x, z, y, xSize, zSize);

// Read / write a block type:
const type = blocks[idx];
blocks[idx] = BLOCKS.STONE;
```

#### Client-side reassembly

```js
// LevelAssembler collects LevelDataChunk packets and decompresses the final map.
// NOTE: ClassiCubeClient does this automatically and emits 'level'.
// Use LevelAssembler directly only if you're driving PacketDecoder yourself.
const { LevelAssembler } = level;
const asm = new LevelAssembler();

decoder.on('packet', async p => {
  if (p.name === 'levelInitialize') asm.reset();
  if (p.name === 'levelDataChunk')  asm.push(p.chunkData, p.chunkLength);
  if (p.name === 'levelFinalize') {
    const blocks = await asm.decompress(); // -> raw block Buffer, 4-byte header stripped
  }
});

asm.byteLength;  // -> number - compressed bytes received so far
```

#### Constants

```js
level.CHUNK_SIZE  // -> 1024  (bytes per LevelDataChunk payload slot)
```

---

### `cpe`

Encoders and decoders for all CPE (Classic Protocol Extension) packets.

```js
const { cpe } = require('classic-node-protocol');
```

#### Negotiation (both directions)

```js
// 0x10 - ExtInfo: announces how many ExtEntry packets follow.
cpe.encodeExtInfo(appName, extensionCount);   // -> Buffer (67 bytes)
cpe.decodeExtInfo(buf);
// -> { name: 'extInfo', id: 0x10, appName, extensionCount }

// 0x11 - ExtEntry: declares support for one extension.
cpe.encodeExtEntry(extName, version);         // -> Buffer (69 bytes)
cpe.decodeExtEntry(buf);
// -> { name: 'extEntry', id: 0x11, extName, version }

// Build a complete handshake as an array of Buffers: [ExtInfo, ...ExtEntry].
// extensions: array of names from CPE_EXTENSIONS keys.
cpe.buildExtensionHandshake(appName, extensions);  // -> Buffer[]
```

#### ClickDistance

```js
// 0x12 - Tell the client how far it can interact with blocks.
// distance in FShort units. Default = 160 (5 blocks).
cpe.encodeSetClickDistance(distance);   // -> Buffer (3 bytes)
cpe.decodeSetClickDistance(buf);
// -> { name: 'setClickDistance', id: 0x12, distance }
```

#### CustomBlocks

```js
// 0x13 - Both sides send this to agree on custom block support level.
// supportLevel 1 = IDs 50-65 are available.
cpe.encodeCustomBlockSupportLevel(supportLevel);   // -> Buffer (2 bytes)
cpe.decodeCustomBlockSupportLevel(buf);
// -> { name: 'customBlockSupportLevel', id: 0x13, supportLevel }
```

#### HeldBlock

```js
// 0x14 - Force the client to hold a specific block.
// blockToHold: 0 = hide hand. preventChange: 0 = allow switch, 1 = lock.
cpe.encodeHoldThis(blockToHold, preventChange = 0);   // -> Buffer (3 bytes)
cpe.decodeHoldThis(buf);
// -> { name: 'holdThis', id: 0x14, blockToHold, preventChange }
```

#### TextHotKey

```js
// 0x15 - Define a client-side keyboard shortcut.
// action: text injected when the key is pressed. Append '\n' to auto-submit.
// keyCode: LWJGL key code integer.
// keyMods: 0=None 1=Ctrl 2=Shift 4=Alt (bitwise-combinable).
cpe.encodeSetTextHotKey(label, action, keyCode, keyMods = 0);   // -> Buffer (134 bytes)
cpe.decodeSetTextHotKey(buf);
// -> { name: 'setTextHotKey', id: 0x15, label, action, keyCode, keyMods }
```

#### ExtPlayerList

```js
// 0x16 - Add / update a tab-list entry (ExtPlayerList v2).
// nameId: unique Int16 per entry. groupRank: ascending sort order.
cpe.encodeExtAddPlayerName(nameId, playerName, listName, groupName, groupRank);  // -> Buffer (196 bytes)
cpe.decodeExtAddPlayerName(buf);
// -> { name: 'extAddPlayerName', id: 0x16, nameId, playerName, listName, groupName, groupRank }

// 0x18 - Remove a tab-list entry.
cpe.encodeExtRemovePlayerName(nameId);   // -> Buffer (3 bytes)
cpe.decodeExtRemovePlayerName(buf);
// -> { name: 'extRemovePlayerName', id: 0x18, nameId }

// 0x1D - Extended spawn packet; replaces SpawnPlayer when ExtPlayerList v2 is active.
// skinName: player name OR https:// URL to a .png skin.
cpe.encodeExtAddEntity2(entityId, inGameName, skinName, x, y, z, yaw, pitch);   // -> Buffer (138 bytes)
cpe.decodeExtAddEntity2(buf);
// -> { name: 'extAddEntity2', id: 0x1D, entityId, inGameName, skinName, x, y, z, yaw, pitch }
```

#### EnvColors

```js
// 0x19 - Change one environment color variable.
// variable: use cpe.ENV_COLOR constants.
// r/g/b: 0-255. Pass -1/-1/-1 to reset to default.
cpe.encodeEnvSetColor(variable, r, g, b);   // -> Buffer (8 bytes)
cpe.decodeEnvSetColor(buf);
// -> { name: 'envSetColor', id: 0x19, variable, r, g, b }

cpe.ENV_COLOR  // -> { SKY: 0, CLOUD: 1, FOG: 2, AMBIENT: 3, SUNLIGHT: 4 }
```

#### SelectionCuboid

```js
// 0x1A - Draw a colored highlight box around a block region.
// selectionId: 0-255. Coords are block ints (Int16). a = opacity 0-255.
cpe.encodeMakeSelection(selectionId, label, x1, y1, z1, x2, y2, z2, r, g, b, a);  // -> Buffer (86 bytes)
cpe.decodeMakeSelection(buf);
// -> { name: 'makeSelection', id: 0x1A, selectionId, label, startX, startY, startZ, endX, endY, endZ, r, g, b, a }

// 0x1B - Remove a selection region.
cpe.encodeRemoveSelection(selectionId);   // -> Buffer (2 bytes)
cpe.decodeRemoveSelection(buf);
// -> { name: 'removeSelection', id: 0x1B, selectionId }
```

#### BlockPermissions

```js
// 0x1C - Allow or deny placing/breaking a specific block type.
// allowPlace / allowDestroy: 1 = allowed, 0 = denied.
cpe.encodeSetBlockPermission(blockType, allowPlace, allowDestroy);   // -> Buffer (4 bytes)
cpe.decodeSetBlockPermission(buf);
// -> { name: 'setBlockPermission', id: 0x1C, blockType, allowPlace, allowDestroy }
```

#### ChangeModel

```js
// 0x1D - Change a player entity's visual model.
// WARNING: shares ID 0x1D with ExtAddEntity2. Decoder disambiguates by buffer length.
// Common modelName values: 'humanoid', 'creeper', 'chicken', 'spider', 'zombie'
cpe.encodeChangeModel(entityId, modelName);   // -> Buffer (66 bytes)
```

#### EnvMapAppearance (legacy)

```js
// 0x1E - Set texture URL, side/edge block types, and water level (deprecated v2).
// Prefer EnvMapAspect (encodeSetMapEnvUrl + encodeSetMapEnvProperty) for modern clients.
cpe.encodeSetMapEnvAppearance(textureUrl, sideBlock, edgeBlock, sideLevel);   // -> Buffer (69 bytes)
cpe.decodeSetMapEnvAppearance(buf);
// -> { name: 'setMapEnvAppearance', id: 0x1E, textureUrl, sideBlock, edgeBlock, sideLevel }
```

#### EnvWeatherType

```js
// 0x1F - Change the weather effect.
cpe.encodeEnvSetWeatherType(weatherType);   // -> Buffer (2 bytes)
cpe.decodeEnvSetWeatherType(buf);
// -> { name: 'envSetWeatherType', id: 0x1F, weatherType }

cpe.WEATHER  // -> { SUNNY: 0, RAINING: 1, SNOWING: 2 }
```

#### HackControl

```js
// 0x20 - Enable or disable individual movement abilities on the client.
// All flag args: 0 = disabled, 1 = enabled.
// jumpHeight: FShort units (32 per block); -1 resets to default.
// WARNING: ClassiCube ignores this packet before the first LevelDataChunk.
cpe.encodeHackControl(flying, noClip, speeding, spawnControl, thirdPersonView, jumpHeight);  // -> Buffer (8 bytes)
cpe.decodeHackControl(buf);
// -> { name: 'hackControl', id: 0x20, flying, noClip, speeding, spawnControl, thirdPersonView, jumpHeight }
```

#### BlockDefinitions

```js
// 0x23 - Define a custom block (v1). blockId should be in 50-65.
cpe.encodeDefineBlock(def);    // -> Buffer (80 bytes)
cpe.decodeDefineBlock(buf);
// -> { name: 'defineBlock', id: 0x23, blockId, blockName, solidity, movementSpeed,
//      topTexture, leftTexture, rightTexture, frontTexture, backTexture, bottomTexture,
//      transmitsLight, walkSound, fullBright, shape, blockDraw }

// 0x24 - Remove a previously defined custom block.
cpe.encodeRemoveBlockDefinition(blockId);   // -> Buffer (2 bytes)
cpe.decodeRemoveBlockDefinition(buf);
// -> { name: 'removeBlockDefinition', id: 0x24, blockId }

// 0x25 - Define a custom block with extended hitbox, fog color, and fog density (v2).
cpe.encodeDefineBlockExt(def);   // -> Buffer (85 bytes)
cpe.decodeDefineBlockExt(buf);
// -> { ...same as defineBlock, minX, minY, minZ, maxX, maxY, maxZ, fogDensity }
```

**`def` object shape for DefineBlock / DefineBlockExt:**

| Field            | Type     | Description                                                          |
|------------------|----------|----------------------------------------------------------------------|
| `blockId`        | `number` | ID to assign (50-65 for custom blocks)                               |
| `name`           | `string` | Display name (max 64 chars)                                          |
| `solidity`       | `number` | `0`=walk-through `1`=swim-through `2`=solid                          |
| `movementSpeed`  | `number` | Speed multiplier (0-255)                                             |
| `topTexture`     | `number` | Texture atlas index for top face                                     |
| `leftTexture`    | `number` | Left face texture atlas index                                        |
| `rightTexture`   | `number` | Right face texture atlas index                                       |
| `frontTexture`   | `number` | Front face texture atlas index                                       |
| `backTexture`    | `number` | Back face texture atlas index                                        |
| `bottomTexture`  | `number` | Bottom face texture atlas index                                      |
| `transmitsLight` | `number` | `0`=opaque `1`=transparent                                           |
| `walkSound`      | `number` | Sound ID played when walking on this block                           |
| `fullBright`     | `number` | `0`=normal lighting `1`=always fully lit                             |
| `shape`          | `number` | `0`=cube `16`=sprite                                                 |
| `blockDraw`      | `number` | `0`=opaque `1`=transparent `2`=translucent `3`=sprite                |
| `minX/Y/Z`       | `number` | **(v2 only)** Min hitbox coordinates (0-16, where 16 = 1 block)      |
| `maxX/Y/Z`       | `number` | **(v2 only)** Max hitbox coordinates (0-16)                          |
| `fogDensity`     | `number` | **(v2 only)** Fog density inside this block (0 = no fog)             |

#### BulkBlockUpdate

```js
// 0x26 - Update up to 256 blocks in a single packet.
// updates: Array of { index: number, blockType: number }
// index = level.blockIndex(x, z, y, xSize, zSize)
cpe.encodeBulkBlockUpdate(updates);   // -> Buffer (1282 bytes)
cpe.decodeBulkBlockUpdate(buf);
// -> { name: 'bulkBlockUpdate', id: 0x26, count, updates: [{ index, blockType }] }
```

#### TextColors

```js
// 0x27 - Redefine the RGBA color that a text color-code character maps to.
// code: ASCII code of the character (e.g. 0x31 for '1', 0x61 for 'a').
cpe.encodeSetTextColor(code, r, g, b, a = 255);   // -> Buffer (6 bytes)
cpe.decodeSetTextColor(buf);
// -> { name: 'setTextColor', id: 0x27, r, g, b, a, code }
```

#### EnvMapAspect

```js
// 0x28 - Set the texture pack URL. Pass '' to use the server default.
cpe.encodeSetMapEnvUrl(textureUrl);   // -> Buffer (65 bytes)
cpe.decodeSetMapEnvUrl(buf);
// -> { name: 'setMapEnvUrl', id: 0x28, textureUrl }

// 0x29 - Set one map environment property.
// Use cpe.MAP_ENV_PROPERTY constants for the property argument. value: Int32.
cpe.encodeSetMapEnvProperty(property, value);   // -> Buffer (6 bytes)
cpe.decodeSetMapEnvProperty(buf);
// -> { name: 'setMapEnvProperty', id: 0x29, property, value }

cpe.MAP_ENV_PROPERTY
// -> { SIDE_BLOCK: 0, EDGE_BLOCK: 1, EDGE_HEIGHT: 2, CLOUD_HEIGHT: 3,
//      MAX_FOG: 4, CLOUD_SPEED: 5, WEATHER_SPEED: 6,
//      WEATHER_FADE: 7, EXP_FOG: 8, SIDE_OFFSET: 9 }
```

#### EntityProperty

```js
// 0x2A - Set rotation or scale of an entity.
// Use cpe.ENTITY_PROPERTY constants for propertyType.
// value: signed Int32 (rotation = degrees * 32768/360; scale = fraction * 1000).
cpe.encodeSetEntityProperty(entityId, propertyType, value);   // -> Buffer (7 bytes)
cpe.decodeSetEntityProperty(buf);
// -> { name: 'setEntityProperty', id: 0x2A, entityId, propertyType, value }

cpe.ENTITY_PROPERTY
// -> { ROT_X: 0, ROT_Y: 1, ROT_Z: 2, SCALE_X: 3, SCALE_Y: 4, SCALE_Z: 5 }
```

#### TwoWayPing

```js
// 0x2B - Ping packet echoed in both directions for latency measurement.
// direction: 0 = server-to-client, 1 = client-to-server.
// data: arbitrary UInt16 that must be echoed back unchanged.
cpe.encodeTwoWayPing(direction, data);   // -> Buffer (3 bytes)
cpe.decodeTwoWayPing(buf);
// -> { name: 'twoWayPing', id: 0x2B, direction, data }
```

#### InventoryOrder

```js
// 0x2C - Set which block appears in a given hotbar slot.
// order: slot index. blockType: block ID for that slot.
cpe.encodeSetInventoryOrder(order, blockType);   // -> Buffer (3 bytes)
cpe.decodeSetInventoryOrder(buf);
// -> { name: 'setInventoryOrder', id: 0x2C, order, blockType }
```

#### PlayerClick (Client -> Server)

```js
// 0x22 - Client notifies the server of a mouse button event.
// button: 0=Left 1=Right 2=Middle | action: 0=Press 1=Release
// yaw: UInt16 (0-65535 = 360 degrees) | pitch: Int16
// targetId: entity ID hit by the click, or -1/0xFF if none
// targetX/Y/Z: block coordinates of the clicked face (Int16)
// targetFace: 0=Bottom 1=Top 2=North 3=South 4=East 5=West 6=none
cpe.encodePlayerClicked(button, action, yaw, pitch, targetId, targetX, targetY, targetZ, targetFace);  // -> Buffer (15 bytes)
cpe.decodePlayerClicked(buf);
// -> { name: 'playerClicked', id: 0x22, button, action, yaw, pitch, targetId, targetX, targetY, targetZ, targetFace }
```

---

### `jugadorUUID`

Player UUID generation and manipulation.
ClassiCube uses MD5-based (v3-style) UUIDs derived from usernames.

```js
const { jugadorUUID } = require('classic-node-protocol');
```

```js
// Generate a deterministic UUID from a username.
// The same username always produces the same UUID.
jugadorUUID.generarUUID('Steve');
// -> '5e918d1e-b34c-3f57-a9e5-3d4c0df318a6'  (example)

// Check whether a string is a well-formed UUID.
jugadorUUID.validarUUID('5e918d1e-b34c-3f57-a9e5-3d4c0df318a6');  // -> true
jugadorUUID.validarUUID('not-a-uuid');                              // -> false

// Remove hyphens from a UUID string.
jugadorUUID.uuidToHex('5e918d1e-b34c-3f57-a9e5-3d4c0df318a6');
// -> '5e918d1eb34c3f57a9e53d4c0df318a6'

// Add hyphens back to a 32-char hex string.
jugadorUUID.hexToUUID('5e918d1eb34c3f57a9e53d4c0df318a6');
// -> '5e918d1e-b34c-3f57-a9e5-3d4c0df318a6'
```

---

### `protocol` (constants)

Raw protocol constants for low-level work.

```js
const protocol = require('classic-node-protocol/protocol');
// or destructure from root:
const { BLOCKS, BLOCK_NAMES, BLOCK_MODE, USER_TYPE, CPE_MAGIC, CPE_EXTENSIONS, CPE_PACKETS } = require('classic-node-protocol');
```

| Export                    | Value / Shape | Description |
|---------------------------|---------------|-------------|
| `PROTOCOL_VERSION`        | `0x07`        | Classic protocol version number |
| `STRING_LENGTH`           | `64`          | Bytes per protocol string field |
| `CLIENT_PACKETS`          | `{ IDENTIFICATION: 0x00, SET_BLOCK: 0x05, POSITION: 0x08, MESSAGE: 0x0D }` | Client->Server packet IDs |
| `SERVER_PACKETS`          | `{ IDENTIFICATION: 0x00, PING: 0x01, … }` | Server->Client packet IDs |
| `CLIENT_PACKET_SIZES`     | `{ [id]: bytes }` | Byte size per client packet ID |
| `SERVER_PACKET_SIZES`     | `{ [id]: bytes }` | Byte size per server packet ID |
| `CPE_PACKETS`             | `{ EXT_INFO: 0x10, EXT_ENTRY: 0x11, … }` | CPE packet IDs |
| `CPE_SERVER_PACKET_SIZES` | `{ [id]: bytes }` | Byte size per CPE server packet |
| `CPE_CLIENT_PACKET_SIZES` | `{ [id]: bytes }` | Byte size per CPE client packet |
| `CPE_EXTENSIONS`          | `{ ClickDistance: 1, CustomBlocks: 1, … }` | All 26 extensions with their version numbers |
| `BLOCK_MODE`              | `{ DESTROY: 0x00, CREATE: 0x01 }` | Used in SetBlock packets |
| `USER_TYPE`               | `{ NORMAL: 0x00, OP: 0x64 }` | Operator flag values |
| `CPE_MAGIC`               | `0x42`        | Sent in `unused` byte of identification to signal CPE support |
| `BLOCKS`                  | `{ AIR: 0, STONE: 1, GRASS: 2, … OBSIDIAN: 49 }` | All 50 vanilla block IDs |
| `BLOCK_NAMES`             | `{ 0: 'AIR', 1: 'STONE', … }` | Reverse lookup: ID -> name |

---

## CPE System

CPE (Classic Protocol Extension) is ClassiCube's extension mechanism layered on top of the vanilla Classic protocol.

### Negotiation flow

```
Client                            Server
  |                                 |
  |-- identification (unused=0x42) ->|   Client signals CPE support
  |                                 |
  |<-- ExtInfo (N extensions) ------|   Server announces its extensions
  |<-- ExtEntry x N ---------------|
  |                                 |
  |-- ExtInfo (M extensions) ------>|   Client announces its extensions
  |-- ExtEntry x M ---------------->|
  |                                 |
  |<-- ServerIdentification --------|   Normal login resumes
```

Both sides compare their lists. Only extensions present on both sides may be used.

### Server implementation

```js
srv.on('connection', client => {
  client.on('packet', p => {

    if (p.name === 'identification') {
      client.username    = p.username;
      client.supportsCpe = p.unused === 0x42;

      if (client.supportsCpe) {
        // MUST be sent before sendIdentification()
        client.sendCpeHandshake('MyServer', ['EnvColors', 'HackControl', 'EnvWeatherType']);
      }
      client.sendIdentification('MyServer', 'Welcome');
    }

    if (p.name === 'extEntry') {
      client.extensions.add(p.extName);
    }

    // Use CPE features after the map has been sent
    if (p.name === 'levelFinalize') {
      if (client.extensions.has('EnvColors')) {
        client.sendEnvSetColor(cpe.ENV_COLOR.SKY, 100, 149, 237);
      }
      if (client.extensions.has('HackControl')) {
        client.sendHackControl(0, 0, 1, 1, 1, -1);  // no fly, no noclip
      }
    }
  });
});
```

---

## FShort Coordinates

The Classic protocol represents positions as fixed-point **Int16** values:

```
1 block = 32 units
```

```js
const { encoder } = require('classic-node-protocol');

encoder.toFShort(5);      // -> 160   (block 5)
encoder.toFShort(3.5);    // -> 112   (half-block precision)
encoder.fromFShort(160);  // -> 5
encoder.fromFShort(112);  // -> 3.5

// Spawn a player centered on block y=17:
client.spawnAt(0xFF, 'Player', 32, 17.5, 32);  // auto-converts internally

// Or convert manually:
const y = encoder.toFShort(17.5);  // -> 560
client.sendSpawnPlayer(0xFF, 'Player', encoder.toFShort(32), y, encoder.toFShort(32), 0, 0);
```

---

## Blocks Reference

```js
const { BLOCKS, BLOCK_NAMES } = require('classic-node-protocol');

// IDs 0-49 (vanilla Classic blocks)
BLOCKS.AIR            // 0
BLOCKS.STONE          // 1
BLOCKS.GRASS          // 2
BLOCKS.DIRT           // 3
BLOCKS.COBBLESTONE    // 4
BLOCKS.WOOD_PLANKS    // 5
BLOCKS.SAPLING        // 6
BLOCKS.BEDROCK        // 7
BLOCKS.WATER_FLOWING  // 8
BLOCKS.WATER          // 9
BLOCKS.LAVA_FLOWING   // 10
BLOCKS.LAVA           // 11
BLOCKS.SAND           // 12
BLOCKS.GRAVEL         // 13
BLOCKS.GOLD_ORE       // 14
BLOCKS.IRON_ORE       // 15
BLOCKS.COAL_ORE       // 16
BLOCKS.LOG            // 17
BLOCKS.LEAVES         // 18
BLOCKS.SPONGE         // 19
BLOCKS.GLASS          // 20
// RED_CLOTH (21) through WHITE_CLOTH (36)
BLOCKS.DANDELION      // 37
BLOCKS.ROSE           // 38
BLOCKS.BROWN_MUSHROOM // 39
BLOCKS.RED_MUSHROOM   // 40
BLOCKS.GOLD_BLOCK     // 41
BLOCKS.IRON_BLOCK     // 42
BLOCKS.DOUBLE_SLAB    // 43
BLOCKS.SLAB           // 44
BLOCKS.BRICK          // 45
BLOCKS.TNT            // 46
BLOCKS.BOOKSHELF      // 47
BLOCKS.MOSSY_COBBLE   // 48
BLOCKS.OBSIDIAN       // 49

// Reverse lookup: ID -> name string
BLOCK_NAMES[2]   // -> 'GRASS'
BLOCK_NAMES[49]  // -> 'OBSIDIAN'
```

---

## Examples

### Minimal echo server

```js
const { createServer, level, BLOCKS, encoder } = require('classic-node-protocol');

const srv = createServer({ port: 25565 });
const W = 64, H = 32, D = 64;
const blocks = level.buildFlatMap(W, H, D);

srv.on('connection', async client => {
  client.on('packet', async p => {
    switch (p.name) {
      case 'identification':
        client.username = p.username;
        client.sendIdentification('EchoServer', 'Welcome');
        await client.sendLevel(blocks, W, H, D);
        client.spawnAt(0xFF, client.username, W/2, H/2+1, D/2);
        client.sendServerMessage(`Hello, ${client.username}!`);
        break;

      case 'message':
        srv.broadcastMessage(`<${client.username}> ${p.message}`);
        break;

      case 'setBlock': {
        const type = p.mode === 1 ? p.blockType : BLOCKS.AIR;
        srv.broadcast(encoder.encodeServerSetBlock(p.x, p.y, p.z, type));
        break;
      }
    }
  });

  client.on('end', () => srv.broadcastMessage(`${client.username ?? '?'} left.`));
});
```

### Bot with chat commands

```js
const { createClient, BLOCKS } = require('classic-node-protocol');

const bot = createClient({ host: 'localhost', port: 25565 });
bot.on('connect', () => bot.sendIdentification('CommandBot'));

bot.on('packet', p => {
  if (p.name !== 'message' || !p.message.startsWith('!')) return;
  const [cmd, ...args] = p.message.slice(1).split(' ');

  if (cmd === 'goto') {
    bot.sendPositionBlocks(...args.slice(0,3).map(Number), 0, 0);
  }
  if (cmd === 'place') {
    const [x, y, z] = args.map(Number);
    bot.sendSetBlock(x, y, z, 1, BLOCKS.STONE);
  }
});
```

### CPE server with atmosphere

```js
const { createServer, level, cpe, CPE_MAGIC } = require('classic-node-protocol');

const srv = createServer({ port: 25565 });
const blocks = level.buildFlatMap(128, 64, 128);
const EXTS = ['EnvColors', 'HackControl', 'EnvWeatherType', 'SelectionCuboid'];

srv.on('connection', async client => {
  client.on('packet', async p => {
    if (p.name === 'identification') {
      client.username    = p.username;
      client.supportsCpe = p.unused === CPE_MAGIC;

      if (client.supportsCpe) client.sendCpeHandshake('MyServer', EXTS);
      client.sendIdentification('MyServer', 'CPE Demo');
      await client.sendLevel(blocks, 128, 64, 128);
      client.spawnAt(0xFF, client.username, 64, 33, 64);

      if (client.supportsCpe) {
        client.sendEnvSetColor(cpe.ENV_COLOR.SKY, 80, 0, 130);       // purple sky
        client.sendEnvSetColor(cpe.ENV_COLOR.FOG, 50, 0, 100);       // purple fog
        client.sendEnvSetWeatherType(cpe.WEATHER.SNOWING);            // snow
        client.sendHackControl(0, 0, 1, 1, 1, -1);                   // no fly, no noclip
        client.sendMakeSelection(0, 'Spawn', 60,32,60, 68,34,68, 0,255,0,80);
      }
    }
    if (p.name === 'extEntry') client.extensions.add(p.extName);
  });
});
```

### BulkBlockUpdate for terrain fills

```js
// Update a 16x16 patch of ground with gold blocks in one packet
function fillGold(client, xSize, zSize) {
  const updates = [];
  for (let x = 0; x < 16; x++) {
    for (let z = 0; z < 16; z++) {
      updates.push({
        index:     level.blockIndex(x, z, 0, xSize, 64),
        blockType: BLOCKS.GOLD_BLOCK,
      });
    }
  }
  // Protocol limit is 256 updates per packet
  for (let i = 0; i < updates.length; i += 256) {
    client.sendBulkBlockUpdate(updates.slice(i, i + 256));
  }
}
```

### PacketDecoder standalone

```js
const net = require('net');
const { PacketDecoder, encoder } = require('classic-node-protocol');

const socket = net.createConnection({ host: 'localhost', port: 25565 }, () => {
  socket.write(encoder.encodeClientIdentification('RawBot'));
});

// 'server' = we are the client, parsing packets sent BY the server
const decoder = new PacketDecoder('server');
socket.on('data', data => decoder.receive(data));

decoder.on('packet', p => {
  if (p.name === 'identification') console.log('Connected to:', p.serverName);
  if (p.name === 'message')        console.log('[chat]', p.message);
});

decoder.on('error', err => console.error('Parse error:', err.message));
```

---

## License

See the `LICENSE` file included in the package.
