---
sidebar_position: 4
title: Notes for contract authors
---

# Notes for contract authors

Earth runs CosmWasm, with IBC. A tx's results (events, msg responses, error
text) are stored for ever and served by every node, so the chain caps their
size. Bytes are counted at their size in a node's JSON answer: a plain ASCII
character counts 1, a `"` or `\` counts 2, a `<`, `>`, `&` or control character
counts 6, and a msg response counts 2 per byte.

- The first 8 KiB of a tx's results is free. Each byte past that costs 20 gas.
- A tx may emit at most 1 MiB of results, and so may each of its top-level
  msgs. Over that, the tx fails with `ErrTxTooLarge` (sdk code 21). (A tx made
  only of IBC relay msgs has no byte cap; gas bounds it.)
- On an IBC contract port, one received packet may emit at most 512 KiB: the
  events plus twice the acknowledgement. Over that, the packet gets an error
  acknowledgement. The same 512 KiB applies to `ibc_packet_ack` and
  `ibc_packet_timeout`, and over it the relayer's msg fails.
- An IBC callback's events are capped at 256 KiB. Over that, the callback
  fails.

**Don't echo a received acknowledgement, or any other data the counterparty
chooses, into your events.** The counterparty sets the ack's size. If
`ibc_packet_ack` emits a large enough ack, it passes 512 KiB and every
`MsgAcknowledgement` for that packet fails. The packet was already received,
so it can't time out either, and anything the contract escrowed for it stays
locked. Emit a hash or a length instead.
