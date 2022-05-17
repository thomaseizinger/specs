# WebRTC

| Lifecycle Stage | Maturity      | Status | Latest Revision |
|-----------------|---------------|--------|-----------------|
| 1A              | Working Draft | Active |                 |

Authors: [@mxinden]

Interest Group: [@marten-seemann]

## Motivation

1. **No need for valid TLS certificates.** Enable browsers to connect to public
   server nodes without those server nodes providing a TLS certificate within
   the browsers trustchain. Note that we can not do this today with our
   Websocket transport.

2. **Hole punching in the browser**: Enable two browsers or a browser and a
   non-public server node to connect.

## Requirements

- Loading a remote nodes certificate into ones browser trust-store is not an
  option, i.e. doesn't scale.

- No dependency on central STUN and/or TURN servers.

## Protocol

Scenario: Browser _A_ wants to connect to server node _B_.

### Setup - Both _A_ and _B_

1. [Generate a
   certificate](https://www.w3.org/TR/webrtc/#dom-rtcpeerconnection-generatecertificate).

2. [Get the certificate's
   fingerprint](https://www.w3.org/TR/webrtc/#dom-rtccertificate-getfingerprints).

3. Sign this fingerprint using the libp2p peer key to generate a "peer cert"

### Connection Establishment

1. Browser _A_ discovers server node _B_'s multiaddr, containing _B_'s IP, UDP
  port and TLS certificate fingerprint (e.g.
  `/ip6/2001:db8::/udp/1234/webrtc/<tls-certificate-fingerprint`), through some
  external mechanism.

2. _A_ create a connection with its local certificate.

3. _A_ constructs _B_'s SDP offer based on _B_'s multiaddr and sets it
   via
   [`RTCPeerConnection.setRemoteDescription()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/setRemoteDescription).

4. _A_ establishes the connection to _B_.

5. _A_ opens a data stream.

6. _A_ and _B_ exchange the "peer certs".

7. [Get the remote peer's TLS
   certificate](https://www.w3.org/TR/webrtc/#dom-rtcdtlstransport-getremotecertificates).

8. Check the remote endpoint's "peer cert".

## Open Questions

_Where _Browser_ is replaced with the major desktop browsers of today (Chrome,
Safari, Edge, Firefox)._

### Direct connections

#### Browser to public server node

- Can _Browser_ generate a _valid_ SDP packet for the remote node based on the
  remote's Multiaddr, where that Multiaddr contains the IP, UDP port and TLS
  certificate fingerprint (e.g.
  `/ip6/2001:db8::/udp/1234/webrtc/<tls-certificate-fingerprint`)? _Valid_ in
  the sense that this generated SDP packet can then be used to establish a
  WebRTC connection to the remote.

- Do the major (Go / Rust / ...) WebRTC implementations allow us to accept a
  WebRTC connection from a remote node without previously receiving an SDP
  packet from such host?

#### Browser to browser or non-public server node

- Can _Browser_ know upfront its UDP port which it is listening for incoming
  connections on?

- Can _Browser_ control the lifecycle of its local TLS certificate, i.e. can
  _Browser_ use the same TLS certificate for multiple WebRTC connections?

- Can two _Browsers_ exchange their SDP packets via a third server node using
  Circuit Relay v2 and DCUtR? Instead of exchanging the original SDP packets,
  could they exchange their multiaddr and construct the remote's SDP packet
  based on it?

### Securing the connection

While WebRTC offers confidentiality and integrity via TLS, one still needs to
authenticate the remote peer by its libp2p identity.

- Can a _Browser_ access the fingerprint of its TLS certificate?

  Chrome allows you to access the fingerprint of any locally-created certificate
  directly via `RTCCertificate#getFingerprints`. Firefox does not allow you to
  do so. Browser compatibility can be found
  [here](https://developer.mozilla.org/en-US/docs/Web/API/RTCCertificate). In
  practice, this is not an issue since the fingerprint is embedded in the local
  SDP string.

- Is the above proposed #protocol secure?

### Multiplexing

- Can we use WebRTC’s data channels in _Browser_ to multiplex a single
  connection, or do we need to run an additional multiplexer (e.g. yamux) on top
  of a WebRTC connection and WebRTC datachannel? In other words, does WebRTC
  provide all functionality of a libp2p muxer like Yamux (e.g. flow control)?

## Previous, ongoing and related work

- Proof of concept for the server side in rust-libp2p:
  https://github.com/libp2p/rust-libp2p/pull/2622

- Proof of concept for the server side (native) and the client side (Rust in
  WASM): https://github.com/wngr/libp2p-webrtc

- WebRTC using STUN and TURN: https://github.com/libp2p/js-libp2p-webrtc-star