
============
 mixnet pki
============

- phase 1: In order to produce a proof of concept mix network as soon
  as possible a rather simple key server can be used where the
  correctness of the cryptographic construction would be the primary
  concern.

- phase 2: Eliminate the SPOF (single point of failure); Tor Project's
  directory authority design is somewhat decentralized in that it uses a
  small set of directory authority servers to negotiate a network
  consensus document. The ``dir-spec.txt`` file from the ``torspec`` git
  repo discusses their design in detail:

  https://git.torproject.org/torspec.git

  A simple mix network would require something considerably less
  complicated than Tor's dir auth scheme.


simple mixnet keyserver
=======================

a mixnet key server could perhaps have these responsibilities:

- listen for identity announcements and updates from mix nodes
- perform connectivity tests to mix nodes
- consensus process for selecting the mix net node to advertise
- provide a mix node consensus set to clients who wish to download it


keyserver
---------

design goals:
 - do not leak identity keys over the network
 - authenticated key updates
 - forward secrecy
 - symmetrically sized call and response to prevent traffic
   amplification attacks

TODO:
  - investigate making the protocol call and responses take
    equal amounts of work to produce in terms of memory and CPU usage

The following cryptographic scheme is used for a mix network where
each mix has a signing key and a routing key. The routing key is a
curve25519 key whereas the singing key is an ed25519 key. When the mix
performs a key rotation both the signing key and the routing key are
rotated.

In this context forward secrecy means that if a mix routing key or
signing key is compromised it means loss of confidentiality until the
next key rotatation.

Symmetrically sized call and response is easily achieved by encrypted
padding, for example the MixIntroduction will be the same size as
MixIntroductionAck which uses padding.


notation for key types
----------------------

- S   : keyserver identity key
- S'  : keyserver ephemeral key
- C   : client identity key
- C'  : client ephemeral key
- N   : new client identity key
- R   : client mix curve25519 routing key
- Q   : new client mix curve25519 routing key
- R_private : private curve25519 routing key
- Q_private: new private curve25519 routing key
- G   : curve25519 generator value


notation for cryptographic operations
-------------------------------------

- Box : Box( private_key, public_key)[ data_to_encrypt ]
- scalar_mult : scalar_mult(base, exponent)

* uses DJB's NaCl crypto library; here are the pynacl docs: https://pynacl.readthedocs.io/


envelope types with fields
--------------------------

RequestKey
 - C'
 - Nonce
 - Box( C', S )[ padding ]

RequestKeyResponse
 - S'
 - Nonce
 - Box( S, C` )[ S' ]

Query
 - C'
 - S'
 - Nonce
 - Box( C', S' )[
     - C
     - Box( C, S' )[
       - C'
       - "query payload"
       - padding ] ]

QueryResponse
 - S'
 - C'
 - Nonce
 - Box( S', C' )[
   - "query response payload"
   - padding ]

AnonymousQuery
 - C'
 - S'
 - Nonce
 - Box( C', S' )[
   - "query payload"
   - padding ]

MixIntroduction
 - C'
 - S'
 - Nonce
 - Box( C', S' )[ 
     - C 
     - Box( C, S' )[
       - C'
       - R
       - scalar_mult(scalar_mult(generator, make_curve(S')), R_private) ] ]

MixIntroductionAck
 - S'
 - C'
 - Nonce
 - Box( S', C' )[ padding ]

MixUpdate
 - C'
 - S'
 - Nonce
 - Box( C', S' )[ 
     - C 
     - Box( C, S' )[
       - C'
       - scalar_mult(scalar_mult(generator, make_curve(Q)), R_private)
       - N
       - Box( N, S' )[
         - scalar_mult(scalar_mult(generator, make_curve(S')), Q_private) ] ] ]

MixUpdateAck
 - S'
 - C'
 - Nonce
 - Box( S', N )[ padding ]

UnknownSession
 - R'
 - Nonce
 - Box( R', S )[ padding ]


possible channel states
-----------------------

- STATE_CLOSED          : No matchin channel exists
- STATE_DISCONNECTED    : Channel exists but no session exists
- STATE_HALF_SESSION    : Local ephermal key exists but no remote ephermal key is known
- STATE_CONNECTED       : Local and remote ephermal keys are known
- STATE_HALF_INTRODUCED : Introduction sent but no acknowledgement was received
- STATE_INTRODUCED      : Introduction send and acknowledgement received
- STATE_HALF_UPDATED    : A mix has sent it's key rotation update to the keyserver
- STATE_UPDATED         : A mix has received an ACK after sending a key rotation update to the keyserver
- STATE_HALF_QUERIED    : A mix has sent it's query to the keyserver
- STATE_QUERIED         : A mix has received a QueryResponse after sending a query


error codes
-----------

- SUCCESS                 : opperation was successfull
- ERROR_UNKNOWN_CHANNEL   : the referenced channel id is unknown
- ERROR_DISCONNECTED      : the referenced session id is unknown
- ERROR_NOT_DISCONNECTED  : the opperation is only valid on disconnected channels


valid state transitions for API calls
-------------------------------------

**client state transitions**

- mix introduction:

  - STATE_CLOSED -> STATE_HALF_SESSION: client sends RequestKey
  - STATE_HALF_SESSION -> STATE_CONNECTED: client receives a valid RequestKeyResponse
  - STATE_CONNECTED -> HALF_INTRODUCED: client sends MixIntroduction
  - STATE_HALF_INTRODUCED -> STATE_INTRODUCED: client receives a valid MixIntroductionAck

- mix update:

  - STATE_CLOSED -> STATE_HALF_SESSION: client sends RequestKey
  - STATE_HALF_SESSION -> STATE_CONNECTED: client receives a valid RequestKeyResponse
  - STATE_CONNECTED -> STATE_HALF_UPDATED: client sends MixUpdate
  - STATE_HALF_UPDATED -> STATE_UPDATED: client received a valid MixUpdateAck

- mix query:

  - STATE_CLOSED -> STATE_HALF_SESSION: client sends RequestKey
  - STATE_HALF_SESSION -> STATE_CONNECTED: client receives a valid RequestKeyResponse
  - STATE_CONNECTED -> STATE_HALF_QUERIED: client sends a Query
  - STATE_HALF_QUERIED -> STATED_QUERED: client received QueryResponse

- anonymous query:

  - STATE_CLOSED -> STATE_HALF_SESSION: client sends RequestKey
  - STATE_HALF_SESSION -> STATE_CONNECTED: client receives a valid RequestKeyResponse
  - STATE_CONNECTED -> STATE_HALF_QUERIED: client sends an AnonymousQuery
  - STATE_HALF_QUERIED -> STATED_QUERED: client received QueryResponse



acknowledgements
----------------

This keyserver design was inspired by Jonathan Moore's unfinished halite crypto library:
https://bitbucket.org/0x0000/halite/

