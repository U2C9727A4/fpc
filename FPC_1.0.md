# File Procedure Call v1.0
###### Document revision 1

# Abstract
FPC v1.0 is a synchronus protocol intended for file-like access to embedded device resources over a reliable bytestream.

# v1.0 Notes
This document is deliberately small because the author is lazy.

# How to read this document
Please read the FPC Core v0.X document aswell, as it defines common points accross FPC versions, and this document uses terminology from it.
All integers in this document are little-endian on the wire.

# Transport assumptions
FPC v1.0 assumes a transport layer that is:
1) Reliable (Does not corrupt bytes during transit)
2) Sequential (All bytes arrive as they are sent)
3) Connection oriented (A connection state is kept)

# Nodes
Nodes in FPC are RPC endpoints.

All nodes have a node class, which may be:
0) Unassigned (Does not require a capability) (Node Class 0)
1) File Node (Requires Basic IO capability) (Node Class 1)

Nodes classes are a u8 on the wire.

## Class Unassigned
The unassigned class nodes do not implement any RPCs, and are simply dummy endpoints.

## Class File
The File class nodes implement file-like RPCs that cause them to mimic files.
The file class also implement attributes.

### Attributes
Attributes are a u8 bitmask on the wire.
File class nodes may have the following attributes:
Read. (bit 0). This attribute asserts that the node can be read data from.
Write. (bit 1). This attribute asserts that the node can be written data to.
Appending. (bit 2). This attribute asserts that all writes happen appendingly to the node. (This *must* coexist with the write and resizing attribute)
Volatile. (bit 3). This attribute asserts that the data the node has may change without intervention from FPC. (An example would be a file that represents the state of a digital input pin).  
Resizing. (bit 4). This attribute asserts that the node's data may be resized.


## Node Addressing
As per version v1.0, All nodes have a single address, called a "NID".
An NID may only point to a single node, and must be valid for the lifetime of that node.
For servers with multiple clients, the same NID must point to the same node.

A NID is a 32 bit unsigned integer.

# General Architecture
FPC v1.X is a request-response protocol that may not generate unsolicited responses, request-response is strict.  
All requests block until their responses are emitted.  

# Errors
Any RPC call can fail. When one does, an error response is sent, instead of the pairing RPC response.

## Error codes
Error codes are a unsigned 32 bit integer on the wire.
Error codes after 50000 is free to use for RPC-specific errors. (These error codes are namespaced to the RPCs.)
Error codes before 50000 is reserved for internal FPC errors.

Error codes ranging from 0 to 1000 is for transport errors.
Error codes ranging from 1000 to 2000 are internal errors.
Error codes ranging from 2000 to 3000 are client errors.

0: Timeout. (The transport timed out while the server was awaiting data)  

1000: Unknown Irrecoverable Error.  
1001: Out Of Memory.
1002: Hardware Fault.

2000: Frame beyond MTU limits.
2001: Unknown RPCID.



# Capabilities
FPC v1.0 has the following capabilities (Capabilities are a u8 bitmask on the wire.):
1) Basic IO (bit 0)

If a client calls an RPC that is implemented by a capability the server does not have, the server can respond do it with an unknown RPC error. (Or as the author puts it "Sorry pal, I don't know this guy!")

## Basic IO
This capability actives:
File class nodes

Read RPC (RPCID 20)
Write RPC (RCPID 21)
Stat RPC (RPCID 22)
Resize RPC (RPCID 23)

# RPCs
FPC v1.0 implements the following RPCs, along with their RPCIDs (Refer to FPC Core on what RPCID is.).  
(Headers defined by these RPCs in both request and responses must be appended to shared headers as defined by FPC Core.)

RPC IDs 0 to 20 are allocated for FPC v1.0's metadata needs, and may not be used by Node Classes, and these allocated RPCs *must* be implemented, regardless of the capabilities the server has. 

## GetCapabilities (RCPID 0)
This RPC gets the capabilities the server has.  

### Request headers
None.  

### Response headers  
0..1, u8, capabilities. (this is a bitmask that represents the capabilities this server has.)  

### Error codes
This RPC may not error. (This does not mean the server cannot fail when producing this result, it just means that the RPC itself cannot error).

## GetNodeClass (RPCID 1)
This RPC gets the node class of a node.

### Request headers
0..4, u32, NID. The address of the node to get the class of.

### Response Headers
0..1, u8, Class. The Class of the node at NID.

### Error Codes
This RPC may error with the following codes:

50000: Node at NID does not exist.

## Error (RPCID 2)
This is a special "RPC" that may not be requested by clients. It is simply a placeholder for failing RPCs.

### Request Headers
This RPC may not be requested, Requesting this RPC must error with "Unknown RPC" error.

### Response Headers
0..4, u32, Error. (The error code)

## Read RPC (RPCID 20)
This RPC is specific to the File class of nodes.
This RPC reads a blob of data from the given file node

### Request Headers
0..4, u32, NID. the address of the node to read data from
4..8, u32, amount. The amount of bytes to read
8..12, u32, Offset. The offset from start to read bytes from

### Response Headers
0..4, u32, amount. The amount of bytes that were read.
4..(4+X), Blob, Data. The data that was read.

### Error Codes
This RPC may error with the following error codes:
50000: Node at NID does not exist.
50001: Node at NID is not in the file class.
50002: The node does not support this operation. (Read attribute not set).
50003: Offset beyond limits.  

51000: The response won't fit into MTU limits. (This is not a failure in the real sense, it just means you've requested too much data, and must lower it.)

## Write RPC (RPCID 21)
This RPC is specific to the file class of nodes.
This RPC writes a binary blob into a node's data.

### Behavior
If the given blob (ie the data to be written) cannot fit into the file after the offset, this RPC will attempt to grow the file in order to make it fit.
If growing fails, it will simply write whatever it can and return.

### Request Headers
0..4, u32, NID. The address of the node to operate on.
4..8, u32, offset. The offset from start to write bytes to
8..12, u32, Blob size. The size (in bytes) of the blob.
12..(12+X), Blob, data. The data to be written.

### Response Headers
0..4, u32, amount. The amount of bytes that were successfully written.

### Error Codes  
This RPC may error with the following error codes:
50000: Node at NID does not exist.
50001: Node at NID is not in the file class.
50002: The node does not support this operation. (Write attribute not set)

## Stat RPC (RPCID 22)
This RPC is specific to file class of nodes.
This RPC gets the attributes, file size (in bytes) of the given file node in NID.

### Request Headers
0..4, u32, NID. The address of the node to operate on.

### Response Headers
0..4, u32, File size. The size of the file (In bytes).
4..5, u8, attributes. The attribute bitmask of the file.

### Error codes
This RPC may error with the following error codes:
50000: Node at NID does not exist.
50001: Node at NID is not in the file class.

## Resize RPC (RPCID 23)
This RPC is specific to the file class of nodes.
This RPC resizes a file.

### Behavior
Given a file node whoose size is 25 before the operation, If the newly requested size is smaller than the original size, bytes at the end (tail) of the file must be truncated out.
If the newly requested size is bigger than the original size, the end (tail) must be grown to the requested size and filled with zeroes.

If the request cannot be satisfied (and thus returns an error), this RPC does nothing.

### Request Headers
0..4, u32, NID. The address of the node to operate on.  
4..8, u32, new size. The newly requested size.  

### Response headers
None. Merely communicates success of operation.

### Error codes
This RPC may error with the following error codes:
50000: Node at NID does not exist.
50001: Node at NID is not in the file class.
50002: Operation failed.
50003: File is not resizeable.

