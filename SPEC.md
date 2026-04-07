# File Procedure Call v1.0
FPC is a protocol designed for file access over a reliable, sequential and robust bytestreams, Mainly intended for embedded systems wanting to expose their resources in a file-like API.  
# Document revision 1
# WARNING: This document is currently marked as incomplete

# Data types used in this document  
u32 = Unsigned 32 bit integer  
u16 = Unsigned 16 bit integer  
u8 = Unsigned 8 bit integer  
String = An array of u8's terminated by a u8 whoose value is 0.  
nid = u32
Blob = An array of opaque bytes which is not terminated.


# General header structure  
This is the base header definition that may not change accross versions.  
(All integers are little endian unless specified otherwise.)  
0..4 u32 Total frame size  
4..6 u16 Major version  
6..7 u8  Minor version  
7..8 u8  Frame Type  
(The rest is dependent on the frame type.)  


# Versioning  
FPC is versioned according to this format: `vX.Y`.  
X is the major version (maximum is 2^16 - 1)  
Y is the minor version (maximum is 2^8 - 1)  

Major and Minor versions are counted up as new releases are available.

Major versions may be incompatible with previous major versions.  
Minor versions must be a superset of previous minor versions. (For example, a frame which has a version of v1.0 is compatible with a v1.5 frame, but not vice versa.)  

# General architecture
The general architecture of FPC is a "Request and wait" architecture.  

Client sends server a request and waits for the server to respond with either success or error.   

There are exceptions to this dynamic however, as the server can send responses without the client's request such as:
1) Client timing out
2) A notification/event being raised
3) Some transport error the server may detect

# Maximum Transmission Unit (MTU)
The maximum transmission unit defines the maximum FPC frame size a server may handle
An FPC server MUST have at least 32 bytes of MTU, however a larger MTU of ~128 bytes is recommended for meaningful operation. 

# Node ID
A node ID is an ID unique to every node. It may not change for the lifetime of the node.  
A node ID is a u32.  
The highest bit of the node ID is used to indicate if the node is a file, or directory node.
Node IDs are NOT namespaced to a given client, a NID accross diffirent connections and clients point to the same node.  

# Nodes
FPC exposes nodes (RPC endpoints) that may be either a file, or a directory.  

## Events
file nodes may generate events, The generated events are as follows:

CHANGE: The change event may only be generated from nodes which are marked as volatile, It signifies that the underlying data of the node has changed.
LOCK: The lock event is generated when the node that generated it is locked
UNLOCK: The unlock event is generated when the node that generated it is unlocked.

Events in frames uses a bitmask of 8 bits:
bit 7: CHANGE
bit 6: LOCK
bit 5: UNLOCK 

By default, a client does not listen for any event(s) on any node(s).

# Concurrency
Nodes, by default, allow concurrent access. However, client(s) may lock nodes if they do not want other clients to interfere.  
If two (or more) clients are operating on the same node without locking it, the behavior is undefined.

## Attributes
Each node has attributes that alter its behavior, which are documented below:
Atomic: The atomic asserts that all read and write operations may NOT happen partially.  
Seekable: The seekable attribute asserts that the read and write operations with offset arguements may be used.  
Fifo: The fifo attribute asserts that all read and write operations happen in a FIFO way. (Writes are appending, Reads are popping from the head)  
Read: The read attribute asserts that the node can be read data from.  
Write: The write attribute asserts that the node can be written data to.  
Static: The static attribute asserts that the node's size may not be altered and the node may not disappear.
Volatile: The volatile attribute asserts that the node's data may change from external sources or the data is not persistent accross device power cycles.  

Attributes in frames use a bitmask, and said bitmask is documented as follows:
(The bitmask uses an u16.)
Atomic      bit 15
Seekable    bit 14
FIFO        bit 13
Read        bit 12
Write       bit 11
Static      bit 10
Volatile    bit 9
UNUSED      bit 8
UNUSED      bit 7
UNUSED      bit 6
UNUSED      bit 5
UNUSED      bit 4
UNUSED      bit 3
UNUSED      bit 2
UNUSED      bit 1
UNUSED      bit 0

# Paths
in FPC v1.0, a "Path" is a string that fully addresses a node, along with its parental hiearchy.
The slash ("/") character is used as the path seperator
For example, lets take the path string "/gpio/digital/1".
From that string, theres a parentless node named "gpio" which is a directory node.
There is a directory node named "digital" whoose parent is a node named "gpio".
And there is a node named "1" (Node type unknown) whoose parent is a directory node named "digital" whoose parent is a directory node named "gpio".


# Frame type
FPC uses frame types to describe the operation(s) it may perform.  
The highest bit of the frame type is used to indicate if it is a response to a previous operation or not
(All numbers are in decimal unless specified otherwise, and the frame types are typed as requesting.)
(The headers each RPC defines are appended to the end of the general headers.)
(All integer data types used in the frames are little-endian unless specified otherwise)  

## No Operation
The No Operation frame request an operation to do nothing, successfully.  
It does not describe its own headers, so the general header structure is used directly without modification.  
Frame type: 0

## GetNID
The GetNID frame type requests the Node ID of a node at a given path.

### Request headers:  
0..4 u32 Path string size
4..X String Path

### Response headers:  
0..4 nid NodeID

Frame type: 1
## Error
This is a special frame that only the server may _respond_ with.  
As it may only be used for responses, it does not have definitions for a request frame.

### Response headers:  
0..4 u32 Error code

Frame type: 2

## GetPath
The GetPath frame's request requests the server to emit the path of the node at the given NID.  

### Request headers:
0..4 nid Node

### Response headers:  
0..4 u32 Path Size
4..X String Path

Frame type: 3  

## Read
The read frame, as the name suggests reads data from nodes, specifically; File nodes.
Offset field is ignored if the node is not marked as seekable, or if the node is marked as a FIFO.
(Other attributes alter this frame's behavior aswell)

### Request headers:  
0..4 nid Node
4..8 u32 Amount of bytes to read
8..12 u32 Offset

### Response Headers:
0..4 u32 Blob size
4..X blob Bytes read

Frame type: 4

## Write
The write frame, as the name suggests writes data to nodes, specifically: File nodes.
Offset field is ignored if the node is not marked as seekable, or if the node is marked as a FIFO.
(Other attributes alter behavior of this frame aswell)

### Request headers
0..4 nid Node
4..8 u32 Offset
8..12 u32 Blob size
12..X Blob Bytes to write

### Response headers
0..4 u32 Amount of bytes written

Frame type: 5

## Get
The get frame atomically gets entire file contents, however by its nature the get frame is heavily constrained and its response may not exceed the MTU of the server. (ie. the transaction will fail if the underlying file node's data size is larger than the server's MTU)
(Node attributes also alter the behavior of this frame)

### Request headers
0..4 nid Node

### Response headers
0..4 u32 Blob size
4..X Blob Bytes read

Frame type: 6

## Set
The set frame is the writing cousion of the Get frame. It sets entire file contents (truncating the previous data). However by its nature, set frames are constrained to be equal to or less than the server's MTU.
(Node attributes may alter the behavior of this frame)   

### Request headers
0..4 nid Node
4..8 u32 Blob size
8..X Blob Bytes to write

### Response headers
No response headers, merely communicates success

Frame type: 7


## Stat
The stat frame gets attributes, Data size and other metadata about Nodes; Specifically file nodes.
(Node attributes may alter the behavior of this )

### Request Headers
0..4 nid Node

### Response headers
0..4 u32 Data size (Signifies how much data the file has)
4..6 u16 Attributes (bitmask of the attributes the file has)
6..7 u8  Events (Bitmask of the events the file may generate)

Frame type: 8

## Get Children Amount
The get children amount frame gets the amount of children a directory node has.  

### Request headers
0..4 nid Node

### Response headers
0..4 u32 Children amount

Frame type: 9

## Get child NID
The get child nid frame gets the NID of a child of a directory node.  

### Request headers
0..4 nid Node
4..8 u32 'th child. (This field specifies which child to get the NID of. If this field is the zero, it gets the NID of the first child the node has. If its set to one, it gets the NID of the second child the node has, and so on.)  

### Response Headers
0..4 nid Child Node

Frame type: 10


## Get parent NID
This frame gets the NID of the parent of the given node.

### Request headers
0..4 nid Node

### Response headers
0..4 nid Parent Node

Frame type: 11

## GetMTU
This frame gets the maximum transmission unit size the server can handle.

### Request headers
None.

### Response headers
0..4 u32 MTU

Frame type: 12

## Event
This frame is a special frame used only by the server to raise events to the client.

### Request headers
None. This frame may not be requested.

### Response headers
0..1 u8 Events bitmask
1..5 nid Node (Node that generated the event)

Frame type: 13

## Notify
This frame sets what events a client wants to listen for. (The event bitmask is used as a toggle, all zero bits means the client does not want to listen for any events)

### Request headers
0..4 nid Node
4..5 u8 Events bitmask

### Response headers
0..1 u8 Events bitmask (The events the client is currently listening for on the node)

Frame type: 14

## Lock
This frame locks a node into an exclusive resource of the client that called it.

### Request headers
0..4 nid Node

### Response headers
None. Merely communicates success.

Frame type: 15

## Unlock
This frame unlocks a node from an exclusive resource of the client that called it. (Client must own lock beforehand)

### Request headers
0..4 nid Node

### Response headers
None. Merely communicates success.

Frame type: 16

TODO: read+, write+, get+, set+, notify+, stat+, get_children_amount+, get_child_nid+, get_parent_nid+, GetMTU+, event+, error+, lock+, unlock+
TODO: Error codes
TODO: List read, write, get and set behavior on diffirent file attributes.
TODO: Version negotiation