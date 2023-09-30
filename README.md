# Veilid DEFCON video notes
## Basics
- "Veilid is an open-source peer-to-peer mobile-first network application framework"
- Like Tor + IPFS
- Apps communicate directly to each other, instead of to central servers
- Written in Rust, very portable
- Built on top of standard protocols
- Services to make Veilid work are all within the network (besides a little DNS)
- Nodes and identites are distinct
- Built with context switching in mind (eg: using a cell network and going in a tunnel, moving between different networks)

## Nodes
- No special nodes, everyone runs the same thing
- All Veilid applications are nodes
- One popular app would expand out the network, including for less popular apps
- Two types of nodes
    - veilid-core: standard node for apps
    - veilid-server: node without an app attached, just for those who want to expand the network

## Protocols
- Devs can choose which protocols they want to use depending on need
- UDP
    - Fast, unreliable
- TCP
    - Not as fast, reliable
- Websockets
    - Reliable, support for HTTP/HTTPS delivery
    - Browsers can directly contact any other node on the network

## Network Topology
- https://veilid.com/img/Network-Thumbnail.png
### Basic Bootstrap
- Get a new node into the network
- Only use of DNS, makes requests to find bootstrap.veilid.com (or whatever your bootstrap server is configured as)
- Gets public keys and dial info, points you to a few Veilid nodes
- Your node goes to provided nodes to get routing tables to find other nodes, and then gets their routing tables. This builds up a list of nodes
- Applies a distance metric to each node (XOR public key)
- Nodes are identified by public key
- Once you have bootstrapped once, your node reuses the same list of nodes
### Bootstrapping in depth
- Any node can bootstrap
- Bootstraps "find self"
    - Find nodes "close" to the bootstrap and close to yourself
- Public Address Detection
    - Determines how nodes can connect and if they need help
    - Gets public IP from other nodes it knows
    - Calculates dial info, which is info on how your node can be reached, signed with public key cryto
- Relay Configuration
    - Some nodes may need the help of relays to allow connection (eg: behind a NAT, like most nodes at homes without firewall configuration)
    - Veilid nodes will try to help each other to the best of their ability
    - Relay will be part of the node's dial info
- Peer Minimum Refresh
    - Nodes in your routing table are asked to return nodes "near" you
- Network Class Detection
    - Determine what type of NAT you have, which affects how you can be reached
- Ping Validation
    - Check whether nodes are still alive
    - Nodes are pinged with exponential backoff (nodes that have been up for longer are pinged less frequently)

## Cryptography
- The cryptography suite for the project is called "VLD0"
- Uses a series of known good and vetted tools
### VLD0
- Authentication is Ed25519
    - Elliptic curve25519
    - Fast and secure asymmetric encryption
- Key Exchange is x25519
    - DH function that is part of curve25519
    - DH can be used to generate symmetric key with your target's public key and your secret key
- Encryption is XChaCha20-Poly1305
    - Symmetric stream cipher
    - Has associated data encryption system
    - Allows thing like partial packet encryption and including a header with the key
- Message Digest is BLAKE3
    - Very fast hashing algorithm
- Key Derivation is Argon2
    - Password hashes; slow and resistant to GPU attacks
### Upgrading
- Cryptography will need to be upgraded, ability to upgrade is built into Veilid
- Old versions are supported alongside new ones (until deprecated)
- Typed keys are included so all crypto features are tagged with what crypto system they used
- Migration support means data is written back with newer crypto systems. Data can be upgraded by reading it and writing it back to storage
### Secure Storage
- Local storage: ProtectedStore and TableStore
- ProtectedStore
    - Support for device-level secrets (iOS/MacOS keychain, Android key store, Windows Protected Store, Linux Secret Service, etc)
- TableStore
    - encrypted key-value database on the device
- Nothing needs to be stored unencrypted on disk
- Store private keys in ProtectedStore, keep encrypted data in TableStore
- Enables network-level storage: RecordStore and BlockStore
- RecordStore
    - Distributed hash table storage
- BlockStore
    - Not out at time of writing (2023/09/30)
    - Content-addressable data distribution
    - Take What You Give (give more storage to get more storage)
    - Bittorrent-like sharding (data split across nodes)
    - Like shared cloud storage
### On the Wire
- Everything uses same encryption
- Everything is signed, everything is timestamped

## RPC Protocol
- How do nodes talk to each other?
### RPC Schema
- Schema Language is Cap'n Proto
    - Made for deserialization speed and schema evolution
    - Flexible and well supported in Rust
- "Zero copy deserialization"; uses the same format on the wire and in memory
- RPC supports private routing
- Schema evolution built-in; forward and backward compatibility, version changes won't break nodes
### FindNodeQ/FindNodeA
- (Skipped over in talk)
- RPC defines operations such as pinging, finding nodes, getting/setting distributed hash table values
### Distributed Hash Table
- Basically just search
- Store small amounts of data distributed across the hash space
- Veilid DHT allows storage of larger chunks of data
- About a megabyte per key, across an array of subkeys that can be allocated on a per-writer basis
- DHT keys are signed key pairs
- Public key represents location in the hash
- Private key allows writer to sign the data pushed to the key's value
- Allows multiple writers for parts of the data
- Eg: you can have a conversation in one place on the DHT
- Allows caching
- Related data can be put together, don't have to search the whole network when you want to find a thing

## Private Routing
### Private And Safety Routing
- Both ends have a say in the which nodes are used
- Routes are built in two chunks
    - Sender allocates safety route for sending privacy (compare to Tor guard node)
    - Receiver publishes private route for reciever privacy
- Both chunks of compiled together to make the circuit
### Compiled Route
- Onion-style layered encryption; each node can remove one encryption layer to figure out where the data goes next
### Toward the future
- Per-hop payload keying: reduce the amount of info that is common between packets
- Eliminate Hop Counting: done for debugging, can be removed
- Simlify Directionality: routes are bi-directional, but allocated directionally. Could enforce bi-directionality
    - Make packets go back and forth over the same route rather than over two different routes
- Increase hop count
