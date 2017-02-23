
mixnet pki
==========

phase 1
```````

In order to produce a proof of concept mix network as soon as possible
a rather simple key server can be used where the correctness of the
cryptographic construction would be the primary concern.

phase 2
```````

Eliminate the SPOF (single point of failure); Tor Project's directory
authority design is somewhat decentralized in that it uses a small set
of directory authority servers to negotiate a network consensus
document. The ``dir-spec.txt`` file from the ``torspec`` git repo
discusses their design in detail:

https://git.torproject.org/torspec.git

A simple mix network would require something considerably less complicated
than Tor's dir auth scheme.


simple mixnet keyserver
-----------------------

a mixnet key server could perhaps have these responsibilities:

- listen for identity announcements and updates from mix nodes
- perform connectivity tests to mix nodes
- consensus process for selecting the mix net node to advertise
- provide a mix node consensus set to clients who wish to download it


keyserver
`````````

design goals:
 - do not leak identity keys over the network
 - authenticated key updates
 - forward secrecy
 - symmetrically sized call and response to prevent traffic
   amplification attacks

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
~~~~~~~~~~~~~~~~~~~~~~

S   : keyserver identity key
S'  : keyserver ephemeral key
C   : client identity key
C'  : client ephemeral key
N   : new client identity key
R   : client mix routing key (curve25519 public key)
Q   : new client mix routing key (curve25519 public key)

Box : Box( private_key, public_key)[ data_to_encrypt ]

* uses DJB's NaCl crypto library; here are the pynacl docs: https://pynacl.readthedocs.io/


envelope types with fields
~~~~~~~~~~~~~~~~~~~~~~~~~~

RequestKey
 - C'
 - Nonce
 - Box( C', S )[ padding ]

RequestKeyResponse
 - S'
 - Nonce
 - Box( S, C` )[ S' ]

MixIntroduction
 - C'
 - S'
 - Nonce
 - Box( C', S' )[ 
     - C 
     - Box( C, S' )[ C' ]
     - Box( C, S' )[ R ]
   ]

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
     - Box( C, S' )[ C' ]
     - Box( C, S' )[ N ]
     - Box( N, S' )[ Q ]
   ]

MixUpdateAck
 - S'
 - C'
 - Nonce
 - Box( S', N )[ padding ]




valid state transitions
~~~~~~~~~~~~~~~~~~~~~~~



acknowledgements
----------------

This keyserver design was inspired by Jonathan Moore's unfinished halite crypto library:
https://bitbucket.org/0x0000/halite/

