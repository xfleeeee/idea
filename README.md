Value-Scaled Spend Delay (VSD)

Abstract

This proposal introduces a single consensus rule: spending a confirmed UTXO whose value meets or exceeds a threshold requires a relative height delay that scales with the UTXO's value. Small transactions are unaffected. The rule is a pure soft fork (it only reduces the set of valid transactions), changes no data format, adds no new cryptographic primitive, and introduces no new opcode. Its purpose is to convert the confirmation of a large-value transfer from "within one block (~10 minutes)" into "after N blocks", thereby giving a legitimate holder a discovery-and-response window if a key is compromised — without disrupting day-to-day transaction flow.

This proposal is intentionally partial and conservative. It is not a substitute for quantum-resistant signatures (e.g. a future P2MR/PQ-signature soft fork) or for public-key-hiding output types (P2WPKH/P2TR). It is an additive, behavior-layer defense intended to buy time in the common, recoverable scenario: a key is stolen or broken, but the holder is alive and monitoring.
