# WebRTC

| Lifecycle Stage | Maturity      | Status | Latest Revision |
|-----------------|---------------|--------|-----------------|
| 1A              | Working Draft | Active |                 |

Authors: [@mxinden]

Interest Group: [@marten-seemann]

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [WebRTC](#webrtc)
    - [Motivation](#motivation)
    - [Requirements](#requirements)
    - [Addressing](#addressing)
    - [Connection Establishment](#connection-establishment)
        - [Browser to public Server](#browser-to-public-server)
            - [Open Questions](#open-questions)
        - [Browser to Browser](#browser-to-browser)
            - [Open Questions](#open-questions-1)
    - [Multiplexing](#multiplexing)
        - [Ordering](#ordering)
    - [Connection Security](#connection-security)
        - [Open Questions](#open-questions-2)
    - [General Open Questions](#general-open-questions)
    - [Previous, ongoing and related work](#previous-ongoing-and-related-work)
- [FAQ](#faq)

<!-- markdown-toc end -->

## Motivation

1. **No need for trusted TLS certificates.** Enable browsers to connect to
   public server nodes without those server nodes providing a TLS certificate
   within the browsers trustchain. Note that we can not do this today with our
   Websocket transport as the browser requires the remote to have a trusted TLS
   certificate. Nor can we establish a plain TCP or QUIC connection from within
   a browser.

2. **Hole punching in the browser**: Enable two browsers or a browser and a
   server node to connect even though one or both are behind a NAT / firewall.

## Requirements

- Loading a remote nodes certificate into ones browser trust-store is not an
  option, i.e. doesn't scale.

- No dependency on central STUN and/or TURN servers.

## Addressing

WebRTC multiaddresses are composed of an IP and UDP address component, followed
by `/webrtc` and a multihash of the certificate that the node uses.

Examples:
- `/ip4/1.2.3.4/udp/1234/webrtc/certhash/<hash>/p2p/<peer-id>`
- `/ip6/fe80::1ff:fe23:4567:890a/udp/1234/webrtc/certhash/<hash>/p2p/<peer-id>`

The TLS certificate fingerprint in `/certhash` is a
[multibase](https://github.com/multiformats/multibase) encoded
[multihash](https://github.com/multiformats/multihash).

For compatibility implementations MUST support hash algorithm
[`sha-256`](https://github.com/multiformats/multihash) and base encoding
[`base64url`](https://github.com/multiformats/multibase). Implementations MAY
support other hash algorithms and base encodings, but they may not be able to
connect to all other nodes.

## Connection Establishment

### Browser to public Server

Scenario: Browser _A_ wants to connect to server node _B_ where _B_ is publicly
reachable but _B_ does not have a TLS certificate trusted by _A_.

As a preparation browser _A_ [generates a
certificate](https://www.w3.org/TR/webrtc/#dom-rtcpeerconnection-generatecertificate)
and [gets the certificate's
fingerprint](https://www.w3.org/TR/webrtc/#dom-rtccertificate-getfingerprints).

1. Browser _A_ discovers server node _B_'s multiaddr, containing _B_'s IP, UDP
  port, TLS certificate fingerprint and libp2p peer ID (e.g.
  `/ip6/2001:db8::/udp/1234/webrtc/certhash/<hash>/p2p/<peer-id>`),
  through some external mechanism.

2. _A_ instantiates a `RTCPeerConnection`, passing its local certificate as a
   parameter. See
   [`RTCPeerConnection()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/RTCPeerConnection).

3. _A_ constructs _B_'s SDP offer locally based on _B_'s multiaddr and sets it
   via
   [`RTCPeerConnection.setRemoteDescription()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/setRemoteDescription).

4. _A_ creates a local offer via
   [`RTCPeerConnection.createOffer()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/createOffer).
   _A_ generates a random string and sets that string as the username (_ufrag_
   or _username fragment_) and password on the SDP of the local offer. Finally
   _A_ sets the modified offer via
   [`RTCPeerConnection.setLocalDescription()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/setLocalDescription).

   Note that this process, oftentimes referred to as "SDP munging" is disallowed
   by the specification, but not enforced across the major browsers (Safari,
   Firefox, Chrome) due to use-cases in the wild. See also
   https://bugs.chromium.org/p/chromium/issues/detail?id=823036

5. _A_ establishes the connection to _B_. The random string used as a _username_
   and _password_ can be used by _B_ to identify the connection, i.e.
   demultiplex incoming UDP datagrams per incoming connection. _B_ uses the same
   random string for the username and password in the STUN message from _B_ to
   _A_.

6. _B_ does not know the TLS fingerprint of _A_. _B_ upgrades the incoming
   connection from _A_ as an _insecure_ connection, learning _A_'s TLS
   fingerprint through the WebRTC DTLS handshake. At this point the DTLS
   handshake provides confidentiality and integrity but not authenticity.

7. Messages on an `RTCDataChannel` are framed using the message framing
   mechanism described in [Multiplexing](#multiplexing).

8. The remote is authenticated via an additional Noise handshake. See
   [Connection Security](#connection-security).

#### Open Questions

- Is the fact that the server accepts STUN messages from the client prone to
  attacks?

    - Can an attacker launch an **amplification attack** with the STUN endpoint
      of the server?

      The QUIC protocol defends against amplification attacks by requiring:

      > an endpoint MUST limit the amount of data it sends to the unvalidated
      > address to three times the amount of data received from that address.

      https://datatracker.ietf.org/doc/html/rfc9000#section-8

      For WebRTC in libp2p one could require the client (_A_) to add additional
      bytes to its STUN message, e.g. in the STUN username and password, thus
      making an amplification attack less attractive.

    - Can a client run a **DOS attack** by sending many STUN messages with
      different ufrags using different UDP source ports, forcing the server to
      allocate a new peer connection for each? Would rate limiting suffice to
      defend against this attack?

  See also:
  - https://datatracker.ietf.org/doc/html/rfc5389#section-16.2.1
  - https://datatracker.ietf.org/doc/html/rfc5389#section-16.1.2

- Do the major WebRTC server implementations support using the same UDP port for
  multiple WebRTC connections, thus requiring multiplexing multiple WebRTC
  connections on the same UDP port?

  This is related to [ICE Lite](https://www.rfc-editor.org/rfc/rfc5245), having
  a host only advertise a single address, namely the host address, which is
  assumed to be public.

  The [Go WebRTC](https://github.com/pion/webrtc/) implementation and the [Rust
  WebRTC](https://github.com/webrtc-rs/webrtc) implementation support this. It
  is unclear whether this is supported in any of the NodeJS implementations.

- Do the major (Go / Rust / ...) WebRTC implementations allow us to accept a
  WebRTC connection from a remote node without previously receiving an SDP
  packet from such host?

- Can _Browser_ generate a _valid_ SDP packet for the remote node based on the
  remote's Multiaddr, where that Multiaddr contains the IP, UDP port and TLS
  certificate fingerprint (e.g.
  `/ip6/2001:db8::/udp/1234/webrtc/certhash/<hash>/p2p/<peer-id>`)? _Valid_ in
  the sense that this generated SDP packet can then be used to establish a
  WebRTC connection to the remote.

  Yes.

### Browser to Browser

Scenario: Browser _A_ wants to connect to Browser node _B_ with the help of
server node _R_.

- Replace STUN with libp2p's identify and AutoNAT
  - https://github.com/libp2p/specs/tree/master/identify
  - https://github.com/libp2p/specs/blob/master/autonat/README.md
- Replace TURN with libp2p's Circuit Relay v2
  - https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md
- Use DCUtR over Circuit Relay v2 to transmit SDP information
  1. Transform ICE candidates in SDP to multiaddresses.
  2. Transmit the set of multiaddresses to the remote via DCUtR.
  3. Transform the set of multiaddresses back to the remotes SDP.
  4. https://github.com/libp2p/specs/blob/master/relay/DCUtR.md

#### Open Questions

- Can a browser know upfront its UDP port which it is listening for incoming
  connections on? Does the browser reuse the UDP port across many WebRTC
  connections? If that is the case one could connect to any public node, with
  the remote telling the local node what port it is perceived on.

  No, a browser uses a new UDP port for each `RTCPeerConnection`.

- Can _Browser_ control the lifecycle of its local TLS certificate, i.e. can
  _Browser_ use the same TLS certificate for multiple WebRTC connections?

  Yes. For the lifetime of the page, one can generate a certificate once and
  reuse it across connections. See also
  https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/RTCPeerConnection#using_certificates

- Can two _Browsers_ exchange their SDP packets via a third server node using
  Circuit Relay v2 and DCUtR? Instead of exchanging the original SDP packets,
  could they exchange their multiaddr and construct the remote's SDP packet
  based on it?

## Multiplexing

The WebRTC browser APIs do not support half-closing of streams nor resets of the
sending part of streams.
[`RTCDataChannel.close()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel/close)
flushes the remaining messages and closes the local write and read side. After
calling `RTCDataChannel.close()` one can no longer read from the channel. This
lack of functionality is problematic, given that libp2p protocols running on top
of transport protocols, like WebRTC, expect to be able to half-close or reset a
stream. See [Connection Establishment in
libp2p](https://github.com/libp2p/specs/blob/master/connections/README.md#definitions).

To support half-closing and resets of streams, libp2p WebRTC uses message
framing. Messages on a `RTCDataChannel` are embedded into the Protobuf message
below and sent on the `RTCDataChannel` prefixed with the message length in
bytes, encoded as an unsigned variable length integer as defined by the
[multiformats unsigned-varint spec][uvarint-spec].

It is an adaptation from the [QUIC RFC]. When in doubt on the semantics of
these messages, consult the [QUIC RFC].

``` proto
syntax = "proto2";

package webrtc.pb;

message Message {
  enum Flag {
    // The sender will no longer send messages on the stream.
    FIN = 0;
    // The sender will no longer read messages on the stream. Incoming data is
    // being discarded on receipt.
    STOP_SENDING = 1;
    // The sender abruptly terminates the sending part of the stream. The
    // receiver can discard any data that it already received on that stream.
    RESET_STREAM = 2;
  }

  optional Flag flag=1;

  optional bytes message = 2;
}
```

Note that "a STOP_SENDING frame requests that the receiving endpoint send a
RESET_STREAM frame.". See [QUIC RFC - 3.5 Solicited State
Transitions](https://www.rfc-editor.org/rfc/rfc9000.html#section-3.5).

Encoded messages including their length prefix MUST NOT exceed 16kiB to support
all major browsers. See ["Understanding message size
limits"](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_data_channels#understanding_message_size_limits).
Implementations MAY choose to send smaller messages, e.g. to reduce delays
sending _flagged_ messages.

### Ordering

Implementations MAY expose an unordered byte stream abstraction to the user by
overriding the default value of `ordered` `true` to `false` when creating a new
data channel via
[`RTCPeerConnection.createDataChannel`](https://www.w3.org/TR/webrtc/#dom-peerconnection-createdatachannel).

## Connection Security

Note that the below uses the message framing described in
[multiplexing](#multiplexing).

While WebRTC offers confidentiality and integrity via TLS, one still needs to
authenticate the remote peer by its libp2p identity.

After [Connection Establishment](#connection-establishment):

1. _A_ opens a WebRTC datachannel.

2. _A_ starts a Noise `XX` handshake using _A_'s and _B_'s libp2p identity. See
   [noise-libp2p](https://github.com/libp2p/specs/tree/master/noise).

   Instead of exchanging the TLS certificate fingerprints on the established
   Noise channel once the Noise handshake succeeded, _A_ and _B_ use the [Noise
   Prologue](https://noiseprotocol.org/noise.html#prologue) mechanism, thus
   saving one round trip.

   More specifically _A_ and _B_ set the Noise _Prologue_ to
   `libp2p-webrtc-noise:<FINGERPRINTS>` before starting the actual Noise
   handshake. `<FINGERPRINTS>` is the concatenation of the two TLS
   fingerprints of _A_ and _B_ in their multihash byte representation, sorted in
   ascending order.

3. On success of the authentication handshake, the used datachannel is
   closed and the plain WebRTC connection is used with its multiplexing
   capabilities via datachannels. See [Multiplexing](#multiplexing).

Note: WebRTC supports different hash functions to hash the TLS certificate (see
https://datatracker.ietf.org/doc/html/rfc8122#section-5). The hash function used
in WebRTC and the hash function used in the multiaddr `/certhash` component MUST
be the same. On mismatch the final Noise handshake MUST fail.

### Open Questions

- Can a _Browser_ access the fingerprint of its TLS certificate?

  Chrome allows you to access the fingerprint of any locally-created certificate
  directly via `RTCCertificate#getFingerprints`. Firefox does not allow you to
  do so. Browser compatibility can be found
  [here](https://developer.mozilla.org/en-US/docs/Web/API/RTCCertificate). In
  practice, this is not an issue since the fingerprint is embedded in the local
  SDP string.

- Is the above proposed additional handshake secure? See also alternative
  proposed Handshake for
  [WebTransport](https://github.com/libp2p/specs/pull/404).

- Would it be more efficient for _B_ to initiate the Noise handshake? In other
  words, who is able to write on an established WebRTC connection first? _A_ or
  _B_?

- On the server side, can one derive the TLS certificate in a deterministic way
  based on a node's libp2p private key? Benefit would be that a node only needs
  to persist the libp2p private key and not the TLS key material while still
  maintaining a fixed TLS certificate fingerprint.

## General Open Questions

- Should libp2p's WebRTC stack limit itself to using UDP only, or support WebRTC
  on top of both UDP and TCP?

- Is the fact that Firefox does not allow a WebRTC to `localhost` an issue? See
  https://bugzilla.mozilla.org/show_bug.cgi?id=831926.

## Previous, ongoing and related work

- Proof of concept for the server side in rust-libp2p:
  https://github.com/libp2p/rust-libp2p/pull/2622

- Proof of concept for the server side (native) and the client side (Rust in
  WASM): https://github.com/wngr/libp2p-webrtc

- WebRTC using STUN and TURN: https://github.com/libp2p/js-libp2p-webrtc-star

# FAQ

- _Why exchange the TLS certificate fingerprint in the multiaddr? Why not
  base it on the libp2p public key?_

  Browsers do not allow loading a custom certificate. One can only generate a
  certificate via
  [rtcpeerconnection-generatecertificate](https://www.w3.org/TR/webrtc/#dom-rtcpeerconnection-generatecertificate).

- _Why not embed the peer ID in the TLS certificate, thus rendering the
  additional "peer certificate" exchange obsolete?_

  Browsers do not allow editing the properties of the TLS certificate.

- _How about distributing the multiaddr in a signed peer record, thus rendering
  the additional "peer certificate" exchange obsolete?_

  Signed peer records are not yet rolled out across the many libp2p protocols.
  Making the libp2p WebRTC protocol dependent on the former is not deemed worth
  it at this point in time. Later versions of the libp2p WebRTC protocol might
  adopt this optimization.

  Note, one can role out a new version of the libp2p WebRTC protocol through a
  new multiaddr protocol, e.g. `/webrtc-2`.

- _Why exchange fingerprints in an additional authentication handshake on top of
  an established WebRTC connection? Why not only exchange signatures of ones TLS
  fingerprints signed with ones libp2p private key on the plain WebRTC
  connection?_

  Once _A_ and _B_ established a WebRTC connection, _A_ sends
  `signature_libp2p_a(fingerprint_a)` to _B_ and vice versa. While this has the
  benefit of only requring two messages, thus one round trip, it is prone to a
  key compromise and replay attack. Say that _E_ is able to attain
  `signature_libp2p_a(fingerprint_a)` and somehow compromise _A_'s TLS private
  key, _E_ can now impersonate _A_ without knowing _A_'s libp2p private key.

  If one requires the signatures to contain both fingerprints, e.g.
  `signature_libp2p_a(fingerprint_a, fingerprint_b)`, the above attack still
  works, just that _E_ can only impersonate _A_ when talking to _B_.

  Adding a cryptographic identifier of the unique connection (i.e. session) to
  the signature (`signature_libp2p_a(fingerprint_a, fingerprint_b,
  connection_identifier)`) would protect against this attack. To the best of our
  knowledge the browser does not give us access to such identifier.

- _Why use Protobuf for WebRTC message framing. Why not use our own,
  potentially smaller encoding schema?_

  The Protobuf framing adds an overhead of 5 bytes. The unsigned-varint prefix
  adds another 2 bytes. On a large message the overhead is negligible (`(5
  bytes + 2 bytes) / (16384 bytes - 7 bytes) = 0.000427246`). On a small
  message, e.g. a multistream-select message with ~40 bytes the overhead is high
  (`(5 bytes + 2 bytes) / 40 bytes = 0.175`) but likely irrelevant.

  Using Protobuf allows us to evolve the protocol in a backwards compatibile way
  going forward. Using Protobuf is consistent with the many other libp2p
  protocols. These benefits outweigh the drawback of additional overhead.

[QUIC RFC]: https://www.rfc-editor.org/rfc/rfc9000.html