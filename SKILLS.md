# AudioStreamNetCode: agent usage guide

## Purpose and scope

Use this guide to integrate and maintain the AudioStreamNetCode Unity asset. Paths in this guide are relative to the asset root, normally `Assets/AudioStreamNetCode`.

The asset sends Unity audio through Netcode for GameObjects (NGO). It uses the bundled Concentus Opus codec. A host or server sends a separate audio mix to each connected client. A client can also send its AudioSource, microphone, or both to the server.

Use the supplied prefabs as the setup reference. Read the current implementation before changes. [Documentation.md](Documentation.md) contains older Unity and NGO setup notes. [Changelog.md](Changelog.md) contains newer changes. Neither file alone defines the current API or dependencies.

## Dependencies and version checks

The 2.0 changelog gives Unity 6.0 and NGO 2.7.0 as its baseline. Check the target project's Editor version, package manifest, and installed asset code before setup or a dependency change. Do not assume that all Unity and NGO versions are compatible.

Required parts:

| Part | Where to check | Purpose |
| --- | --- | --- |
| NGO and Unity Transport | Project `Packages/manifest.json`, `Prefabs/NetworkManager.prefab` | Network session, player objects, RPCs, transport |
| `AudioStreamNetCode` assembly | `Scripts/AudioStreamNetCode.asmdef` | References `Unity.Netcode.Runtime`, `AudioStreamSupport`, and `Il2CppSetOptionAttribute` |
| `AudioStreamSupport` assembly | `Support/AudioStreamSupport.asmdef` | Shared audio, buffers, input, and runtime information; also has a GUID assembly reference |
| Concentus | `Libs/Concentus/Concentus.dll` | Managed Opus implementation |
| IL2CPP attributes | A project folder containing Unity's IL2CPP attribute source and its assembly definition | Assembly named `Il2CppSetOptionAttribute` with Unity's attribute source |
| Input System | Project manifest and `Support/UserInput.cs` | Demo input supports old input or the new Input System through conditional code |

For a new project, follow [initial-import README](<../!!README1ST!! INITIAL IMPORT ERRORS.md>) for the missing IL2CPP attribute assembly. Copy the attribute source from the installed Unity Editor and use the required assembly name. Do not remove attribute references to hide import errors. Some codec and mixing methods use the attributes without an `ENABLE_IL2CPP` condition, so the assembly is required for script import too.

Check the GUID reference in `AudioStreamSupport.asmdef` against the destination project. Do not assume that copying only `Scripts` is sufficient. Keep relevant support code, libraries, prefabs, resources, and their `.meta` files together. Keep third-party notices and license files with the asset.

## Main files

| File or folder | Responsibility |
| --- | --- |
| `Scripts/AudioStreamNetCodeServer.cs` | Singleton, session start/stop, connection events, downstream packet dispatch |
| `Scripts/AudioStreamNetCodeServer+Tx.cs` | Creates and removes one server TX object per client |
| `Scripts/AudioStreamNetCodeServer+Rx.cs` | Creates client RX objects on the server and publishes receiver snapshots |
| `Scripts/AudioStreamNetCodeServer_TX.cs` | Mixes server capture and other clients for one destination client |
| `Scripts/AudioStreamNetCodeServer_RX.cs` | Owns one client's server-side decoder |
| `Scripts/AudioStreamNetCodeClient.cs` | Player NetworkBehaviour, stream metadata, owner controls, RPCs |
| `Scripts/AudioStreamNetCodeClientTx.cs` | Mixes the client's source and microphone captures into its encoder |
| `Scripts/AudioStreamEncoder.cs` | PCM capture, Opus worker, outgoing packet queue |
| `Scripts/AudioStreamDecoder.cs` | Incoming packet queue, Opus worker, playback and decoded capture |
| `Scripts/AudioStreamUnityMicrophone.cs` | Microphone device selection and recording lifecycle |
| `Support/AudioSourceCaptureBuffer.cs` | Captures audio before local monitoring gain |
| `Prefabs/` | Complete component order, references, and network registration |
| `Demo/AudioStreamNetCodeDemo.unity` and `.cs` | Session controls, IP setup, audio controls, and usage examples |

## Scene setup

Use the supplied prefabs as the reference for component order and required scene wiring.

1. Add `Prefabs/NetworkManager.prefab` and `Prefabs/AudioStreamNetCodeServer.prefab` to the scene. Keep a single active NetworkManager and a single asset server controller per process. The server controller is needed in the client process too.
2. Keep the NetworkManager's UnityTransport reference. The current server code expects UnityTransport with the direct `UnityTransport` protocol.
3. Keep `Prefabs/AudioStreamNetCodeClient#{0}.prefab` as Player Prefab and keep the associated `NetworkPrefabsList.asset` registration. Let NGO spawn the player objects.
4. Keep the server controller's `audioStreamNetCodeServer_TX_prefab` and `audioStreamNetCodeServer_RX_Prefab` references.
5. Assign the desired clip to the server AudioSource, or use the existing capture pattern on a GameObject with an AudioListener. Keep upstream effects above `AudioSourceCaptureBuffer` so capture receives their output.
6. Choose `automaticallyStart`: `HOST`, `SERVER`, `CLIENT`, or `NONE`. The supplied server prefab stores `NONE`, although the C# field initializer is `HOST`. Inspect scene overrides too.
7. Set the transport address and port before starting the session. The supplied NetworkManager uses destination `127.0.0.1`, port `7777`, and server listen address `0.0.0.0`.
8. Choose `use2DRecevingAudioSources` before starting the host/server. Preserve this exact field spelling in code.
9. Keep an active AudioListener and the required audio processing path. A server process must process audio to produce this stream; a network connection alone is not sufficient proof of audio operation.

For two processes on one computer, the client destination is `127.0.0.1`. For another computer, use the server's reachable IP address. `0.0.0.0` is a server bind address, not the client destination. This asset does not provide automatic NAT traversal.

### Required client hierarchy

The client root has the NetworkObject, `AudioStreamNetCodeClient`, `AudioStreamDecoder`, and receiving AudioSource. Its `clientTx` reference points to the child with `AudioStreamNetCodeClientTx`, an AudioSource, and `AudioStreamEncoder`.

That TX component has two required capture references:

- `audioSourceCapture`: the `ClientTxAudioSource` child with an AudioSource and `AudioSourceCaptureBuffer`. Set the client's outgoing clip here.
- `microphoneCapture`: the `ClientTxUnityMicrophone` child with an AudioSource, `AudioStreamUnityMicrophone`, and `AudioSourceCaptureBuffer`. Set `microphoneDeviceId` here. It is an index into `Microphone.devices`, not a persistent device identifier.

Both capture references remain required even when one transmission option is off. Preserve the prefab's component order: source or mixer processing must run before capture or encoding; monitoring gain must run after capture. Demo graphics and transform movement are separate from the audio path. If removing demo parts, inspect their references first.

## Session API

Namespace: `AudioStreamNetCode`. Configure the transport and scene references before these calls. Use the server component's wrappers for host/server startup so source audio starts too.

```csharp
// server is an already wired AudioStreamNetCodeServer reference.
server.StartHost();                 // Server plus a local client.
// OR
server.StartServer();               // Server without a local client.
// OR, in the client process:
Unity.Netcode.NetworkManager.Singleton.StartClient();

// Stop the session and the server audio objects.
server.StopNetworkAndAudio();
```

These are alternative startup calls. Do not run them all. With explicit startup, set `automaticallyStart` to `AUTOSTART.NONE` first. Avoid adding a second startup path through raw `NetworkManager.StartHost()` or `StartServer()`.

`AudioStreamNetCodeServer.Singleton` is set in `OnEnable`, and the server GameObject uses `DontDestroyOnLoad`. Its `Awake` accesses `NetworkManager.Singleton`. Preserve the supplied initialization setup; do not access these singletons from an arbitrary earlier `Awake`.

`OnClientConnected` and `OnClientDisconnected` are Unity events with a `ulong` client ID. Callbacks can run in both client and server processes. Check the network role before server-only work. See the demo's `OnClientConnected` handler.

## Client transmission API

Change the transmission NetworkVariables on the spawned local player's component. Their write permission is `Owner`. Run the following only after the local player is spawned, from code that owns that player:

```csharp
var player = Unity.Netcode.NetworkManager.Singleton.LocalClient.PlayerObject;
var client = player.GetComponent<AudioStreamNetCodeClient>();

client.transmitClientAudioSource.Value = true;
client.transmitClientMicrophone.Value = false;
```

Set either or both values to `true`. Set both to `false` to stop transmission. The owner's `Update` starts and stops capture and sends the stream lifecycle RPCs. Do not call the start/stop/packet RPCs directly for normal integration. Do not write these owner variables on another client's player from server code.

The microphone uses Unity's Microphone API. The user must verify device availability, OS permissions, and platform behavior. The asset does not establish those conditions through network setup.

## Audio route and network behavior

```text
Server source/effects -> capture -------------------------+
                                                         |
Client A source + microphone -> encoder -> server decoder +-> mix for B -> encoder -> B decoder
Client B source + microphone -> encoder -> server decoder +-> mix for A -> encoder -> A decoder
```

Each destination mix contains server audio and all transmitting clients except that destination's own audio. Preserve this exclusion to prevent network return of the client's own signal. It is not acoustic echo cancellation.

- Server TX objects are children of the server controller. Server RX objects are children of the corresponding client player objects.
- The server sends downstream audio with `OnServerPacketRpc`, targeted to the player owner with server invoke permission and unreliable delivery.
- Clients send upstream packets with unreliable `OnClientPacket_ServerRpc`. Stream start/stop RPCs use reliable delivery. Preserve this split.
- `Update` dispatches the packets already queued at the start of each pass. It can send multiple packets per frame. Do not reduce it to one packet per frame or drain an unbounded stream of new arrivals.
- The server has one downstream mix and encoder per client. Account for per-client CPU cost and transport bandwidth when increasing client counts.
- The current code checks for direct UnityTransport. Relay integration and other transports need code work and separate verification. Do not promise support from the older documentation's transport suggestions. Any service policy quoted there must be checked against current official terms before use.

## Audio settings and limits

| Setting | Use and constraint |
| --- | --- |
| `AudioStreamEncoder.bitrate` | Range 6-510; applied as `bitrate * 1024` bits per second |
| `complexity` | Range 0-10; higher values increase codec work |
| `opusApplicationType` | `AUDIO`, `RESTRICTED_LOWDELAY`, or `VOIP`; selected when the encoder starts |
| `rate` | `CBR`, `VBR`, or `Constrained_VBR`; applied during Update |
| `frameSize` | 120, 240, 480, 960, 1920, or 2880 samples per channel; default 960 |
| `encoderThreadPriority`, `decoderThreadPriority` | Worker priorities; keep the defaults unless measurement supports a change |
| `use2DRecevingAudioSources` | Select before session start; passed to downstream and server-side receiving decoders |

The codec constants are 48000 Hz and two channels. The encoder checks that the Unity default speaker mode is mono or stereo. It writes callback PCM into the encoder buffer without an explicit input resampler or mono-to-stereo conversion. Do not infer arbitrary rate/channel conversion from the codec constants. Use 48 kHz stereo as the initial verification setup; other configurations need a specific code audit and manual audio checks.

At the codec rate, 960 samples is 20 ms per frame. This is not total network latency. Total delay also includes DSP buffers, main-thread dispatch, queues, the network, and playback.

In 3D playback mode the decoder supplies a streaming AudioClip through `PCMReaderCallback`. In lower-latency 2D mode it reads decoded samples in `OnAudioFilterRead` and uses a resampler based on the source and output rates. The 2D option removes receiver spatial playback behavior; it does not remove effects already captured at the source. Do not consume the decoder buffer from both callback paths in 3D mode.

Packet queues use `ThreadSafeQueue<byte[]>` with a configured capacity of 100 and drop new packets after their threshold. Do not describe these queues as a jitter buffer or loss recovery system. Encoder FEC is enabled, but the current decoder loop does not implement the commented packet-loss path.

Keep packets within transport and network limits when changing frame size or bitrate. The supplied transport stores `m_MaxPayloadSize: 6144`; this does not establish the usable MTU of a real network.

### Volume and monitoring

- Server `Volume` controls the source AudioSource, or AudioListener volume when no AudioSource is present. It affects the captured source path.
- Server `MonitoringVolume` controls its capture component's post-capture gain. A value of zero silences local monitoring without removing captured audio.
- `ClientsVolume` writes the AudioSource volume of existing server TX objects. `ClientsMonitoringVolume` writes their encoder monitoring gain. The setters do not apply those values to future TX objects; inspect prefab defaults and connection timing when using them.
- Client source AudioSource volume and microphone `Gain` change the audio sent upstream.
- Encoder `monitoringVolume` acts after samples enter the encoder input buffer. Decoder `monitoringVolume` acts after decoded audio is copied to `captureBuffer` for further mixing.
- The local player's receiving AudioSource volume controls local playback. The server source and TX/RX prefabs have monitoring gains set to zero in several places by design. Do not treat all zero monitoring values as a fault.

The demo's server encoder window edits one existing TX encoder, selected from `serverTX`; it does not broadcast settings to every encoder. Configure the TX prefab for startup defaults. If all active streams must change, explicitly apply the supported settings to all existing encoders and define how later connections receive the same configuration.

## Thread and lifecycle constraints for changes

Audio callbacks, codec workers, and the main thread share buffers. Trace the complete path before changing a queue, capture, or shutdown method.

- Preserve buffer locks, the encoder settings lock, and worker stop/join behavior.
- Keep Unity scene and network operations on the main thread. Do not move RPC calls into audio callbacks or codec workers.
- The server publishes an array snapshot of RX objects for the audio thread through `srx`. Do not enumerate the mutable server RX dictionary from the audio thread.
- Keep callback work small. Do not add logging, network I/O, or component creation to audio callbacks.
- Capture buffers must match the callback block length. `AudioSourceCaptureBuffer` allocates its buffer from Update after observing that length. Do not replace this with an assumed DSP block size.
- Treat decoder capture and local monitoring as separate stages. Preserve capture in both 2D and 3D modes.
- Preserve disconnect cleanup and stream stop behavior. Do not change shutdown semantics or suppress lifecycle failures as an unrelated cleanup.

## Troubleshooting sequence

1. For missing namespaces or assemblies, inspect the installed packages, asmdef references, Concentus import, and the IL2CPP attribute assembly.
2. For no connection, inspect client destination, server bind address, port, firewall, Player Prefab registration, and whether the chosen role started.
3. For a connected but silent client, trace source playback, callback capture, server TX encoder queue, owner-targeted RPC, decoder queue, and receiving AudioSource. Check source volume and local monitoring separately.
4. For missing client audio, inspect ownership, transmission flags, the two capture references, microphone state, reliable start/stop messages, and creation of the server RX object.
5. For stutter or delay, inspect DSP settings, frame size, main-thread frame time, queue growth, per-client codec work, transport limits, and network loss. Do not assume that higher worker priority fixes the cause.
6. For incorrect spatial playback, compare the startup 2D flag, receiver AudioSource settings, source effects, listener position, and demo transform sync. Check both decoder callback paths before changing code.