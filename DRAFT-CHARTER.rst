===============================
WebTransport (WEBTRANS) Charter
===============================

Chairs:
  TBD

Applications and Real-Time Area Directors:

- Barry Leiba <barryleiba@computer.org>
- Alexey Melnikov <aamelnikov@fastmail.fm>
- Adam Roach <adam@nostrum.com>

Applications and Real-Time Area Advisor:
  Barry Leiba <barryleiba@computer.org>

Mailing Lists:

  | General Discussion: webtransport@ietf.org
  | To Subscribe: https://www.ietf.org/mailman/listinfo/webtransport
  | Archive: https://mailarchive.ietf.org/arch/browse/webtransport/

Description of Working Group:

  The WebTransport working group will define new protocols or protocol
  extensions in order to support the development of the W3C WebTransport API
  https://wicg.github.io/web-transport;.

  These protocols will support:

  - Reliable bidirectional and unidirectional communication that provides
    greater efficiency than Websockets (e.g. removal of head-of-line blocking).
  - Unreliable datagram communication, functionality not available in
    Websockets.
  - Origin checks to allow supporting the Web's origin-based security model.
  
  The WebTransport working group will define three variants:
  
  - A protocol directly running over QUIC with its own ALPN.
  - A protocol that runs multiplexed with HTTP/3.
  - Fallback protocols that can be used when QUIC or UDP are not available.
  
  The group will pay attention to security issues arising from the above
  scenarios so as to ensure against creation of new modes of attack, as well as
  to ensure that security issues addressed in the design of Websockets remain
  addressed in the new work.
  
  To assist in the coordination with W3C, the group will initially develop an
  overview document containing use cases and requirements in order to clarify
  the goals of the effort.  Feedback will also be solicited at various points
  along the way in order to ensure the best possible match between the protocol
  extensions and the needs of the W3C WebTransport API. The clarity and
  interoperability of specifications will be confirmed via test events and
  hackathons.
  
  The group will also coordinate with other working groups within the IETF (e.g.
  QUIC, HTTPBIS) as appropriate.

Goals and Milestones:

- March 2020 - Adopt a WebTransport Overview draft as a WG work item
- March 2020 - Adopt a draft on WebTransport over QUIC as a WG work item
- March 2020 - Adopt a draft on WebTransport over HTTP/3 as a WG work item
- March 2020 - Adopt a draft on HTTP/2 fallback mechanism as a WG work item
- March 2020 - Adopt a draft on a QUIC fallback mechanism as a WG work item
- August 2020 - Issue WG last call of the WebTransport Overview document.
- November 2020 - Issue WG last call on WebTransport over QUIC
- November 2020 - Issue WG last call on QUIC fallback mechanism
- February 2021 - Issue WG last call on WebTransport over HTTP/3
- February 2021 - Issue WG last call on HTTP/2 fallback mechanism
