# File Procedure Call Core v0
##### Document revision: 1

This document ("Core") documents the FPC Core (v0.0 and above minor releases). and architecture that may not change accross FPC versions.
The FPC Core is used for version and MTU negotion before FPC can be used. (Protocol discovery)
The FPC Core assumes that it is running on top of a bytestream that:
Gurantees delivery of bytes
Gurantees non-corruption of bytes
And is connection oriented.   

# Abstract
This document aims to be a guide for future FPC versions by defining common points accross FPC versions. (AKA A meta-spec for FPC).


# Versioning
All FPC versions have both a major, and minor version.

A major version is a version that may be incompatible with previous and/or future major versions.
A major version is an u16 on the wire. A major version can be at maximum, 65535.
Major version 0 is reserved for Core.

A minor version is a version that must be backwards compatible (AKA a superset) of minor version before it.
a minor version is an u8 on the wire. A minor version can be at maximum, 255.

# FPC Headers
The headers (and wire format) defined as below may NOT change accross FPC versions.
(On the positions, left side is inclusive while the right side isn't. Position 0 comes first (transmitted first). Zig-style types are used)
(position, data type, name, explanation)
(All integers are little-endian unless noted otherwise.)

0..4, u32, Frame_len, This header defines the total lenght of the entire transmission. (Including itself).
4..6, u16, Major_Ver, This header specifies which major version this frame conforms to.
6..7, u8,  Minor_Ver, This header specifies which minor version this frame conforms to.
7..8, u8,  Frame_Type, This header specifies the type of the frame. This header may also be used to specify what operation the frame is doing.

## Frame Type
The frame type defines what the frame's type is; It is a u8 on the wire
The highest bit is used as the response bit; as a side effect the maximum frame type on the wire is 127.
The frame type(s) may also be used as RPC IDs by FPC versions.


# General Architecture
FPC follows the following general architecture:

## Nodes
Nodes in FPC are RPC endpoints that handle RPC functions related to their class (Node classes are to be defined by FPC versions)

## Server Capabilities
Server capabilities are capabilities servers will advertise to clients to note what they support. Capabilites are to be defined by FPC versions.
Capabilities should NOT be what the server is (Eg a multiplexer) but rather optional features of the specific FPC version (Discovery, Hardware-Atomicity, etc.). 
Due to its nature, FPC versions must implement capability discovery themselves.

## Server multi-versioning
FPC servers must support multiple major versions running on the same connection.  
For example, connection cA has a server that supports v0.X (Core) and v1.0.
The client can issue v0.X frames to discover the limits and the versions the server supports, and then later switch to using v1.0 frames, without a dedicated RPC for version switching.
Conversely, if the client already knows the version(s) the server supports, they may skip version negotion entirely (not using v0.X) and use the frame versions the server supports.

Only the maximum minor version of a major version the server supports may be advertised, to reduce version negotiation times. (This is fine, because minor versions are backwards-compatible.). 

# Core RPCs
FPC core supports the following RPCs:
(On the positions, left side is inclusive while the right side isn't. Position 0 comes first (transmitted first). Zig-style types are used)
(position, data type, name, explanation)
(All integers are little-endian unless noted otherwise.)
(Headers each RPC defines must be appended to the non-changing FPC headers. And their position(s) are relative.)

## 0, GetMTU
This RPC gets the maximum transmission unit (ie the maximum frame lenght a server can handle) a server supports.
By minimum, a server must have 32 bytes of MTU.

### Request headers
No request headers.  

### Response Headers  
0..4, u32, Max_Frame_len, The maximum frame lenght the server can handle.  

### Errors  
This RPC may not fail.  

## 1, GetVersion
This RPC gets the FPC version(s) the server supports.

### Request headers
0..4, u32, n, The n'th version the server supports is being requested

### Response headers
0..1, u8, R_class, The response class the server is responding with. (0 = SUCCESS, 1 = ERROR)
(Following headers are for the SUCCESS response class)
1..3, u16, Major_Ver, The major version of the n'th version the server supports.
3..4, u8,  Minor_Ver, The minor version of the n'th version the server supports.

### Errors
This RPC may error with the following error codes:
0: Request index (n) is out of range.

#### Error response
(This is a continuation of the response headers)
1..5, u32, E_code, The error code the server has returned.