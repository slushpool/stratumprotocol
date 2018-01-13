Stratum protocol extensions
===========================

(Adding version rolling support to stratum)

Initial motivation for specifying some general support for stratum
protocol extensions was a need to allow miners to do so called
"version rolling", changing value in the first field of the Bitcoin
block header.

Version rolling is backwards incompatible change to stratum protocol
because miner couldn't communicate different block version value to
the server in the original version of the stratum protocol. Similarly,
a server couldn't communicate safe bits for rolling to a miner. So
both miners and pools need to implement some protocol extension to
support version rolling.

Typically, if a miner sends an unknown message to a server, the server
closes the connection (not all implementations do that but some
do). So it is not very safe to try to send unknown messages to
servers.

We can use the opportunity to make one backward incompatible
change to the protocol to support multiple extensions in the
future. In a way that a miner can advertise its cababilities and at
the same time it can request some needed features from server.

It is preferrable that the same mechanism for feature negotiation can
be used for not yet known features. It should be easy to implement in
the mining software too.

We introduce one new message to the stratum protocol (
**"mining.configure"** ) which handles the initial
configuration/negotiation of features in a generic way. So that adding
features in the future can be done without a necessity to add

Each extension has its unique string name, so called **extension
code**.

Currently, the following extensions are defined: 

- **"version-rolling"**
- **"sp-telemetry"**
- **"minimum-difficulty"**
- **"promise-protocol"**
- **"binary-protocol"**


Additional data types
---------------------

The following names are used as type aliases, making the message
description easier.

- **TMask** - case independent hexadecimal string of length 8,
  encoding unsigned 32-bit integer (~```[0-9a-fA-F]{8}```)
	 
- **TExtensionCode** - non-empty string with a value equal to a name of
  some protocol extension.

- **TExtensionResult** - ```true``` / ```true``` / *String*.
  ```true``` = The requested feature is supported and its configuration
         understood and applied.
  ```false``` = The feature is not supported or unknown.
  *String* = Error message containing information about what went wrong.


Request "mining.configure"
--------------------------

This message (JSON RPC Request) is the **first message** sent by a
miner after connection with a server is established. The client uses
the message to advertise its features and to request/allow some
protocol extensions.

The reason for being the first is that we want the implementation and
possible interactions as easy as possible.

Example request (new-lines added):
```
{"method": "mining.configure",
 "id": 1, 
 "params": [["sp-telemetry", "version-rolling"],
            {"sp-telemetry.version": 1,
			 "version-rolling.mask": "00fff000"}]}
```

(The miner requests extension ```"version-rolling"``` and allows extension
```"sp-telemetry"```. It sets parameters according to the extensions'
definitions.)

Example result (new-lines added):
```
{"error": null,
 "id": 1,
 "result": {"version-rolling": true,
            "version-rolling.mask": "007f8000",
			"sp-telemetry": true}}
```

**Parameters**:

 - **requested-extensions** (required, List of *TExtensionCode*)
 
   Each string must be a valid extension code. The meaning of each
   code is described independently as part of the extension
   definition.
   
 - **extension-parameters** (required, *Map of (String -> Any)*)
 

Each extension code provide a namespace for its parameters and
extension return values. By convention, extension parameter names are
formed from extension codes by adding "." and parameter name. The same
applies for the return values, which are transfered in a map
too. E.g. "version-rolling.mask" is a parameter to extension
"version-rolling".
   

**Retrun value**: _Map of (String -> Any)_
    
  Each code from **requested-extensions** list SHOULD have a defined
  return value (_TExtensionCode_ -> _TExtensionResult_). This way


Defined extensions
==================

Extension "version-rolling"
---------------------------

This extension allows to miner to change value of some bits in the
version field in a block header. Currently there are no standard bits
used for version rolling so they need to be negotioated between a
miner and a server.

A miner sends to the server a mask describing bits which the miner is
capable to change. 1 = changable bit, 0 = not changable (```miner_mask```).

A server typically allows to change only some of the version bits
(```server_mask```) and the rest of the version bits are
fixed. E.g. because the block needs to be valid or some signalling is
in place.

Ther server responds to the configuration message by sending a mask
with common bits intersection of the miner's mask and its a mask
(```response = server_mask & miner_mask```)

Example request:
```
{"method": "mining.configure", "id": 1, "params": [["version-rolling"], {"version-rolling.mask": "00fff000"}]}
```

Example result (success):
```
{"error": null, "id": 1, "result": {"version-rolling": true, "version-rolling.mask": "007f8000"}}
```

Example result (unknown extension):
```
{"error": null, "id": 1, "result": {"version-rolling": false}}
```

**Extension parameters**:

 - **"version-rolling.mask"** (optional, *TMask*, default value
   ```"ffffffff"```) 
   
   Bits set to 1 can be changed by the miner. This value is expected
   to be stable for the whole mining session. A miner doesn't have to
   sent the mask, in such case a default full mask is expected.
   
**Extension return values**:

 - **"version-rolling"** (required, *TExtensionResult*)
 
   When responded ```true```, the server will accept new parameter of
   **"mining.submit"**, see later.
 
 - **"version-rolling.mask"** (required, *TMask*)
  
   Bits set to 1 are allowed to be changed by the mininer. If a miner
   will change bits with mask value 0, the server will reject the submit.
   
   The server SHOULD return the largest mask possible (as many bits
   set to 1 as possible). This can be useful in a mining proxy setup
   when a proxy needs to negotiate the best mask for it's future
   clients.

Notification **"mining.set_version_mask"**:

Server notifies the miner about a new mask valid for the
connection. This message can be sent any time after successfull setup
of the version rolling by "mining.configure" message. The new mask is
valid **immediatelly**, so that the server doesn't wait for the next
job.

**TODO**: Is it really better to use the mask immediatelly or "next
  job" approach would be better?

**Parameters**:

 - *mask* (required, *TMask*): The meaning is the same as
    **"version-rolling.mask"** return parameter.

Example:
```
{"params":["00003000"], "id":null, "method": "mining.set_version_mask"}

```

Changes in request **"mining.submit"**:

Immediatelly after successful activation of version-rolling extension
(result message to "mining.configure" sent by server), the server MUST
accept one additional parameter of the message "mining.submit". Client
MUST send one additional parameter, **version_bits** (6th, after
*worker_name*, *job_id*, *extranonce2*, *ntime* and *nonce*).


**Additional parameter to "mining.submit"**:

- *version_bits* (required, *TMask*) - 

  Miner can set only bits corresponding to the set bits in the last
  recevied mask from the server either as response to
  "mining.configure" or "mining.set_version_mask" notification
  (```last_mask```): ```version_bits & ~last_mask == 0```.
  
  nVersion for the submit is computed by the server as follows:
  ``` nVersion = (job_version & ~last_mask) | (version_bits & last_mask)```, 
  where ```job_version``` is a block version sent to miner as part
  of job with id ```job_id```.


Extension "minimum-difficulty"
------------------------------

This extension allows miner to request a minimum difficutly for the
connected machine. It solves a problem in the original stratum
protocol when a reaction to a "mining.subscribe" is sending a mining
job but there is no

**Extension parameters**:
   - **minimum-difficulty** (required, *Integer/Float*, >= 0) 

**Extension return values**:
   - **minimum-difficulty** (required, *TExtensionResult*)


Extension "subscribe-extranonce"
--------------------------------

Parameter-less extension. Miner advertises its capability of receiving
message **"mining.set_extranonce"** message (useful for hash rate
routing scenarios).

Notification **mining.set_extranonce**:
 - TBD

Extension "sp-telemetry"
------------------------
 - To be publised.

Extension "binary-protocol"
---------------------------
 - To be published

Extension "promise-protocol"
----------------------------
 - To be published
