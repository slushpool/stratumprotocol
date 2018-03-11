# Stratum protocol extensions


The initial motivation for specifying protocol extensions was a need 
to allow miners to do so called "version rolling", changing the value 
in the first field of the Bitcoin block header.

Version rolling is a backwards incompatible change to the stratum protocol
because the miner couldn't communicate different version values to
the server in the original version of the stratum protocol. Similarly,
a server couldn't communicate safe bits for rolling to a miner. So
both miners and pools need to implement some protocol extension to
support version rolling.

Typically, if a miner sends an unknown message to a server, the server
closes the connection (not all implementations do that but some
do). So it's not very safe to try and send unknown messages to the
server.

We can use this opportunity to make one backwards incompatible
change to the protocol to support multiple extensions in the
future. In a way that a miner can advertise it's capabilities and at
the same time it can request some needed features from the server.

It is preferable that the same mechanism for feature negotiation can
be used for not yet known features. It should be easy to implement in
the mining software too.

We introduce one new message to the stratum protocol 
(**"mining.configure"** ) which handles the initial
configuration/negotiation of features in a generic way. 

Each extension has it's unique string name, so called **extension
code**.

Currently, the following extensions are defined: 

- **"version-rolling"**
- **"minimum-difficulty"**
- **"subscribe-extranonce"**
- **"promise-protocol"**
- **"binary-protocol"**
- _more to be published_


### Additional data types

The following names are used as type aliases, making the message
description easier.

- **TMask** - case independent hexadecimal string of length 8,
  encoding an unsigned 32-bit integer (~```[0-9a-fA-F]{8}```)
	 
- **TExtensionCode** - non-empty string with a value equal to the name of
  some protocol extension.

- **TExtensionResult** - *Boolean* / *String*.
  ```true``` = The requested feature is supported and its configuration
         understood and applied.
  ```false``` = The feature is not supported or unknown.
  *String* = Error message containing information about what went wrong.


## Request "mining.configure"

This message (JSON RPC Request) SHOULD be the **first message** sent
by the miner after the connection with the server is established. The client
uses this message to advertise it's features and to request/allow some
protocol extensions.

The reason for it being the first is that we want the implementation and
possible interactions to be as easy and simple as possible. An extension
can define explicitly what a repeated configuration of that
extension means.

Each extension code provides a namespace for it's parameters
and return values. By convention, the names are formed from
extension codes by adding "." and a parameter name. The same applies
for the return values, which are transferred in a result map. 
E.g. "version-rolling.mask" is the name of the parameter "mask" of
the extension "version-rolling".

**Parameters**:

 - **extensions** (required, List of *TExtensionCode*)

   Each string in the list MUST be a valid extension code. The meaning
   of each code is described independently as part of the extension
   definition.

   A miner SHOULD advertise all it's available features.

 - **extension-parameters** (required, *Map of (String -> Any)*)

   Parameters of the requested/allowed extensions from the **extensions**
   parameter.


**Return value**:

 - *Map of (String -> Any)*

   Each code from the **extensions** list MUST have a defined return value
   (_TExtensionCode_ -> _TExtensionResult_). This way the miner knows
   if the extension is activated or not. 
   E.g. ```{"version-rolling":false}``` for unsupported version   rolling.

   Some extensions need additional information to be delivered to the
   miner. The return value map is used for this purpose.


Example request (new-lines added):
```
{"method": "mining.configure",
 "id": 1, 
 "params": [["minimum-difficulty", "version-rolling"],
            {"minimum-difficulty.value": 2048,
	     "version-rolling.mask": "00fff000", "version-rolling.min-bit-count": 2}]}
```

(The miner requests extensions ```"version-rolling"``` and 
```"minimum-difficulty"```. It sets the parameters according to the extensions'
definitions.)

Example result (new-lines added):
```
{"error": null,
 "id": 1,
 "result": {"version-rolling": true,
            "version-rolling.mask": "007f8000",
	    "minimum-difficulty": true}}
```

# Defined extensions

## Extension "version-rolling"

This extension allows the miner to change the value of some bits in the
version field in the block header. Currently there are no standard bits
used for version rolling so they need to be negotiated between a
miner and a server.

A miner sends the server a mask describing bits which the miner is
capable of changing. 1 = changeable bit, 0 = not changeable (```miner_mask```)
and a minimum number of bits that it needs for efficient version rolling.

A server typically allows you to change only some of the version bits
(```server_mask```) and the rest of the version bits are
fixed. E.g. because the block needs to be valid or some signaling is
in place.

The server responds to the configuration message by sending a mask
with common bits intersection of the miner's mask and its a mask
(```response = server_mask & miner_mask```)

Example request (a miner capable of changing any 2 bits from a 12-bit mask):
```
{"method": "mining.configure", "id": 1, "params": [["version-rolling"], {"version-rolling.mask": "00fff000", "version-rolling.min-bit-count": 2}]}
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
   send the mask, in this case a default full mask is used.
   
**Extension return values**:

 - **"version-rolling"** (required, *TExtensionResult*)
 
   When responded with ```true```, the server will expect a new parameter of
   **"mining.submit"**, see later.
 
 - **"version-rolling.mask"** (required, *TMask*)
  
   Bits set to 1 are allowed to be changed by the miner. If a miner
   changes bits with mask value 0, the server will reject the submit.
   
   The server SHOULD return the largest mask possible (as many bits
   set to 1 as possible). This can be useful in a mining proxy setup
   when a proxy needs to negotiate the best mask for it's future
   clients. There is a BIP(TBD) describing available nVersion
   bits. The server should pick a mask that preferably covers all
   bits specified in the BIP.

 - **"version-rolling.min-bit-count"** (required, *TMask*)

   The miner also provides a minimum number of bits that it needs for
   efficient version rolling in hardware. Note that this parameter
   provides important diagnostic information to the pool server. If
   the requested bit count exceeds the pool servers limit, the miner 
   always has the chance to operate in a degraded mode without using 
   full hashing power. The pool server should NOT terminate the 
   miners connection if this rare mismatch case occurs.

### Notification **"mining.set_version_mask"**

Server notifies the miner about a new mask valid for the
connection. This message can be sent at any time after the successful 
setup of the version rolling extensions by the "mining.configure" message. 
The new mask is valid **immediately**, so that the server doesn't wait 
for the next job.

**TODO**: Is it really better to use the mask immediately or "next
  job" approach would be better?

**Parameters**:

 - *mask* (required, *TMask*): The meaning is the same as the
    **"version-rolling.mask"** return parameter.

Example:
```
{"params":["00003000"], "id":null, "method": "mining.set_version_mask"}

```

### Changes in request **"mining.submit"**

Immediately after successful activation of the version-rolling extension
(result to "mining.configure" sent by server), the server MUST accept
an additional parameter of the message **"mining.submit"**. The client MUST
send one additional parameter, **version_bits** (6th parameter, after
*worker_name*, *job_id*, *extranonce2*, *ntime* and *nonce*).


**Additional parameters**:

- *version_bits* (required, *TMask*) - Version bits set by the miner.

  Miner can set only bits corresponding to the set bits in the last
  received mask from the server either as response to
  "mining.configure" or "mining.set_version_mask" notification
  (```last_mask```). This must hold: ```version_bits & ~last_mask ==  0```.
  
  The server computes nVersion from the submit like this:
  ``` nVersion = (job_version & ~last_mask) | (version_bits & last_mask)```, 
  where ```job_version``` is the block version that was sent to the miner 
  for this job.


## Extension "minimum-difficulty"

This extension allows the miner to request a minimum difficulty for the
connection. It solves a problem in the original stratum protocol where 
there was no way to communicate a hard limit for the connection.

**Extension parameters**:
 - **"minimum-difficulty.value"** (required, *Integer/Float*, >= 0)
 
   The minimum difficulty value acceptable for the miner/connection.
   The value can be 0 for essentially disabling the feature.

**Extension return values**:
 - **"minimum-difficulty"** (required, *TExtensionResult*)
   
   Whether the minimum difficulty was accepted or not.


This extension can be configured multiple times by calling
**"mining.configure"** with "minimum-difficulty" code again. 


## Extension "subscribe-extranonce"

Parameter-less extension. The miner advertises it's ability to receive
the **"mining.set_extranonce"** message (useful for hash rate
routing scenarios).

## Extension "info"

Miner provides additional text-based information.

**Extension parameters**:
 - **"info.connection-url"** (optional, *String*)
 Exact URL used by the mining software to connect to the stratum server.
 
 - **"info.hw-version"** (optional, *String*)
 Manufacturer specific hardware revision string.
 
 - **"info.sw-version"** (optional, *String*)
 Manufacturer specific software version
 
 - **"info.hw-id"** (optional, *String*)
 Unique  identifier of the mining device


### Notification **mining.set_extranonce**:
 - TBD

Extension "binary-protocol"
---------------------------
 - To be published

Extension "promise-protocol"
----------------------------
 - To be published
