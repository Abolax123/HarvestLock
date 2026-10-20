# `cancel` propose / sign / finalize — sequence diagram

How a mutual cancellation (contract `cancel()`, buyer + cooperative auth) is
completed when the two parties are on separate devices and no script or
process ever holds more than one private key. It is the two-party case of the
same staged mechanism `reassign_buyer` (three parties) and `resolve_dispute`
(three parties) use — see `../README.md`'s "Staged multi-party signing"
section for the shared design, `reassign-buyer-sequence.md` for the
three-party walk, and `../HANDOFF.md` for what's live-verified.

**Checked against the code, not from memory:** `src/server.ts` (routes),
`src/stellar/multiParty.ts` (propose/finalize), `src/db/pendingMultisigProposals.ts`
(Postgres staging) and `src/stellar/tx.ts` (submit) as of 20 Oct 2026. If any
of those change, update this file in the same change.

## Diagram

Either the buyer or the cooperative may propose. The proposer becomes the
transaction source, so their own auth entry is `source_account` and is
satisfied by their classic envelope signature; the **other** party is the only
entry that needs a wallet `signAuthEntry`. `P` is the proposer, `O` the other
party (buyer/cooperative in either order).

```mermaid
sequenceDiagram
    autonumber
    actor P as Proposer (buyer or cooperative)
    participant API as HarvestLock API (Fastify)
    participant DB as Postgres (pending_multisig_proposals)
    participant RPC as Stellar RPC / escrow contract
    actor O as Other party (cooperative or buyer)

    rect rgb(238, 246, 255)
    note over P,RPC: Propose — buyer or cooperative may start this
    P->>API: POST /commitments/:id/tx/cancel/propose<br/>{ proposerPublicKey }
    API->>API: validate contract ID and G-address (400 if malformed)
    API->>DB: findActiveProposal(contractId, "cancel")
    alt a cancel proposal is already active
        DB-->>API: existing row
        API-->>P: 200 existing proposal (idempotent — no duplicate staged)
    else none
        API->>RPC: getCommitment(contractId) — live read
        RPC-->>API: commitment (buyer, cooperative)
        API->>API: proposerPublicKey must equal buyer or cooperative, else 403
        API->>RPC: getAccount(proposer) + simulateTransaction(cancel(), source = proposer)
        RPC-->>API: simulation success + resource footprint + 2 auth entries<br/>(1 source-account + 1 address entry)
        API->>RPC: getLatestLedger
        RPC-->>API: latest ledger sequence
        API->>API: set signatureExpirationLedger = latest + 17,280 (~24 h)<br/>on the non-source entry, server-side, before any XDR reaches a wallet
        API->>DB: INSERT proposal (status=pending, expires_at=now+24 h,<br/>func_xdr, soroban_data_xdr, auth_entries)
        DB-->>API: row
        API-->>P: 201 { id, status:"pending", pending_entries:[other party] }
    end
    end

    rect rgb(245, 245, 245)
    note over O,API: Sign — the other party, own device, own request, own wallet
    O->>API: GET /commitments/:id/tx/cancel/propose (UI polls every 10 s)
    API->>DB: findActiveProposal
    DB-->>API: row
    API-->>O: { proposal: { pending_entries:[ {address, entry_xdr} ] } }

    O->>O: Freighter signAuthEntry(my entry_xdr) — NOT signTransaction
    O->>API: POST /commitments/:id/tx/propose/:proposalId/sign<br/>{ signerPublicKey, signedEntryXdr }
    API->>API: validate proposalId (UUID), signer key, signedEntryXdr parses as an auth entry (400 otherwise)
    API->>DB: getProposalById — must match contract, be unexpired and "pending" (400 otherwise)
    API->>DB: SELECT ... FOR UPDATE; find this signer's unsigned entry; UPDATE auth_entries
    DB-->>API: row (allSigned = true — the only non-source entry is now signed)
    API->>RPC: getAccount(proposer) — fresh, propose/finalize can be hours apart
    RPC-->>API: current sequence number
    API->>API: rebuild the tx from func_xdr + the original sorobanData +<br/>the signed entries (no re-simulation — that would mint fresh nonces)
    API->>DB: markReady(status=ready, final_xdr = rebuilt unsigned XDR)
    API-->>O: { status:"ready", ready_xdr }
    end

    rect rgb(238, 255, 240)
    note over P,RPC: Finalize — an ordinary signed transaction, existing endpoint
    P->>API: GET proposal (polls) — sees status "ready" and ready_xdr
    P->>P: Freighter signTransaction(ready_xdr) — ordinary envelope signature;<br/>satisfies the proposer's source-account auth entry
    P->>API: POST /transactions/submit<br/>{ xdr, refreshContractId, completeProposalId }
    API->>RPC: sendTransaction + poll getTransaction
    RPC->>RPC: verify both parties' auth, escrow unwinds, status := Cancelled
    RPC-->>API: SUCCESS { hash }
    API->>RPC: getCommitment(refreshContractId) — best-effort cache refresh
    API->>DB: upsert mirrored commitment + reputation consequences — best-effort
    API->>DB: markCompleted(proposalId) — best-effort
    API-->>P: { status:"SUCCESS", hash }
    end
```

GitHub renders Mermaid fenced blocks natively; the same block works in the
VS Code Markdown preview (with Mermaid support) and at <https://mermaid.live>.

## Step-by-step, with where each piece lives

| # | Step | Where |
|---|---|---|
| 1 | Buyer or cooperative proposes with `{ proposerPublicKey }` (`cancel` takes no arguments). | `src/server.ts` `POST /commitments/:contractId/tx/cancel/propose` |
| 2 | An existing active `cancel` proposal is returned as-is (200); otherwise a new one is inserted (201). `reassign_buyer` and `resolve_dispute` proposals are separate rows and don't block it. | `findActiveProposal`, `createProposal` (`src/db/pendingMultisigProposals.ts`) |
| 3 | Proposer must be the commitment's buyer **or** cooperative (403 otherwise), checked against a live chain read — unlike `reassign_buyer`, which only the current buyer may start. | `src/server.ts` cancel propose handler |
| 4 | Simulation is sourced from the proposer, so the proposer's auth entry is `source_account` and never handed to a wallet; that leaves exactly one pending entry (the other party). | `buildMultiPartyProposal` (`src/stellar/multiParty.ts`) |
| 5 | `signatureExpirationLedger` is set **server-side** on the non-source entry (`VALID_LEDGERS_AHEAD` = 17,280 ledgers ≈ 24 h), because Freighter's `signAuthEntry(entryXdr)` takes no expiration argument. | `src/stellar/multiParty.ts` |
| 6 | The row stores the function XDR, the original `sorobanData`, and the entries; TTL is 24 h. | `MULTISIG_PROPOSAL_TTL_MS` (`src/server.ts`), migration `004_generalize_multisig_proposals.sql` |
| 7 | Any viewer polls `GET .../cancel/propose`, which returns `{ proposal: … \| null }`; `pending_entries` lists only unsigned address entries. | `serializeProposal` (`src/server.ts`) |
| 8 | The other party signs their `entry_xdr` with `signAuthEntry` and posts it to the method-agnostic sign route. | `POST .../tx/propose/:proposalId/sign` (`src/server.ts`) |
| 9 | Signing is a locked read-modify-write (`SELECT … FOR UPDATE`); a wrong signer or repeat signature is rejected (400). | `signProposalEntry` (`src/db/pendingMultisigProposals.ts`) |
| 10 | With the single pending entry signed, the API rebuilds the transaction (deliberately **not** re-simulating) and flips the proposal to `ready`. | `finalizeMultiPartyProposal` (`src/stellar/multiParty.ts`), `markReady` |
| 11 | The proposer signs `ready_xdr` classically and submits via `/transactions/submit` with `completeProposalId`; the proposal is then marked `completed`. | `POST /transactions/submit` (`src/server.ts`), `markCompleted` |
| 12 | On-chain, both parties' auth is verified and the commitment is cancelled. | `cancel` in `HarvestLock-Contracts/contracts/escrow/src/lib.rs` |

## Edge cases and checks (all enforced in code, not just documented)

- **Who may propose:** the buyer or the cooperative — 403 for anyone else.
- **Idempotent propose:** proposing again while a `cancel` proposal is active
  returns the existing one (200), even if the second caller is the other party.
  So whoever proposed first is the transaction source and must do the final
  `signTransaction`; the other party only ever signs an auth entry.
- **Wrong signer / double sign:** 400 — the signer must have an unsigned
  entry on *this* proposal.
- **Bad input:** non-UUID `proposalId`, malformed keys, or a `signedEntryXdr`
  that isn't a valid base64 auth entry are rejected with 400 before any write.
- **Expiry:** proposals expire after 24 h (`expires_at`); signing an expired
  proposal is rejected with "propose a fresh one". The auth entry also carries
  a ~24 h on-chain `signatureExpirationLedger`.
- **Not awaiting a signature:** signing a proposal that is already `ready` or
  `completed` is a 400.
- **Reachable states:** `Draft` through `ReadyForDelivery`; from other
  statuses, simulation fails at propose time and the contract error surfaces.
- **Failure surfaces:** `refreshContractId` / `completeProposalId` side
  effects after a successful submit are best-effort and logged, never masking
  a landed transaction.
