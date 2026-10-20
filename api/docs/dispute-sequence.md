# `flag_dispute` / `resolve_dispute` — sequence diagram

How a commitment is frozen into `Disputed` and later released (PRD must-have
"dispute flagging with defined escalation"; contract `flag_dispute()`,
`resolve_dispute()`, `expire_dispute_window()`). The two halves are
different mechanisms:

- **Flag** is single-signer: one party builds, signs and submits an
  ordinary transaction. Nothing is staged.
- **Resolve** needs all three parties' auth, so it uses the same staged
  propose / sign / finalize mechanism as `cancel` and `reassign_buyer` — see
  `../README.md`'s "Staged multi-party signing" section and
  [`reassign-buyer-sequence.md`](./reassign-buyer-sequence.md) for the
  step-by-step detail of the shared machinery. `../HANDOFF.md` records what
  is live-verified.

**Checked against the code, not from memory:** `src/server.ts` (routes
`POST /commitments/:contractId/tx/flag-dispute`,
`POST|GET /commitments/:contractId/tx/resolve-dispute/propose`,
`POST /commitments/:contractId/tx/propose/:proposalId/sign`,
`POST /transactions/submit`), `src/stellar/multiParty.ts`,
`src/db/pendingMultisigProposals.ts`, and the frontend caller
`buyer-app/src/components/DisputeSection.tsx` (identical in `coop-pwa` and
`warehouse-app`). The contract side (`HarvestLock-Contracts`) is described
as `README.md` and `HANDOFF.md` describe it; it is not in this repo. If any
of these change, update this file in the same change.

## Diagram

```mermaid
sequenceDiagram
    autonumber
    actor F as Flagging party (buyer / cooperative / warehouse operator)
    actor P1 as Second party (wallet)
    actor P2 as Third party (wallet)
    participant API as HarvestLock API (Fastify)
    participant DB as Postgres (pending_multisig_proposals)
    participant RPC as Stellar RPC / escrow contract

    rect rgb(255, 244, 238)
    note over F,RPC: Flag — single signer, no staging
    F->>API: POST /commitments/:id/tx/flag-dispute<br/>{ flagger, sourcePublicKey }
    API->>API: validate contract ID and both G-addresses (400 if malformed)<br/>NOT checked against buyer/coop/warehouse here
    API->>RPC: buildInvokeTransaction: flag_dispute(flagger)
    RPC-->>API: simulated, unsigned XDR
    API-->>F: { xdr }
    F->>F: Freighter signTransaction(xdr) — ordinary envelope signature
    F->>API: POST /transactions/submit { xdr, refreshContractId }
    API->>RPC: sendTransaction + poll getTransaction
    RPC->>RPC: require flagger's auth; flagger must be a party (else NotAParty);<br/>status in Locked..Delivered; status := Disputed, remember pre-dispute status
    RPC-->>API: SUCCESS { hash }
    API->>RPC: getCommitment — best-effort cache refresh
    API->>DB: upsert mirrored commitment — best-effort
    API-->>F: { status:"SUCCESS", hash }
    note over API,RPC: Every other lifecycle call is now blocked by the contract's status check
    end

    rect rgb(238, 246, 255)
    note over F,RPC: Resolve — propose (any ONE of the three parties may start)
    F->>API: POST /commitments/:id/tx/resolve-dispute/propose<br/>{ proposerPublicKey }
    API->>DB: findActiveProposal(contractId, "resolve_dispute")
    alt a resolve_dispute proposal is already active
        DB-->>API: existing row
        API-->>F: 200 existing proposal (idempotent)
    else none
        API->>RPC: getCommitment(contractId) — live read
        RPC-->>API: commitment
        API->>API: proposer must be buyer, cooperative or warehouse operator, else 403
        API->>RPC: simulate resolve_dispute(), source = proposer
        RPC-->>API: 3 auth entries (1 source-account + 2 address entries)
        API->>API: set signatureExpirationLedger on each non-source entry (~24 h)
        API->>DB: INSERT proposal (status=pending, expires_at=now+24 h)
        API-->>F: 201 { id, status:"pending", pending_entries:[P1, P2] }
    end
    end

    rect rgb(245, 245, 245)
    note over P1,P2: Sign — each remaining party, own device, own wallet
    P1->>API: GET /commitments/:id/tx/resolve-dispute/propose (UI polls every 10 s)
    API-->>P1: { proposal: { pending_entries:[...] } }
    P1->>P1: Freighter signAuthEntry(own entry_xdr)
    P1->>API: POST /commitments/:id/tx/propose/:proposalId/sign<br/>{ signerPublicKey, signedEntryXdr }
    API->>DB: SELECT ... FOR UPDATE; mark P1's entry signed
    API-->>P1: { status:"pending", pending_entries:[P2] }
    P2->>P2: Freighter signAuthEntry(own entry_xdr)
    P2->>API: POST .../tx/propose/:proposalId/sign
    API->>DB: SELECT ... FOR UPDATE; mark P2's entry signed
    note over API: every non-source entry signed
    API->>API: rebuild tx from stored func_xdr + sorobanData + signed entries<br/>(no re-simulation)
    API->>DB: status=ready, final_xdr = rebuilt unsigned XDR
    API-->>P2: { status:"ready", ready_xdr }
    end

    rect rgb(238, 255, 240)
    note over F,RPC: Finalize — proposer signs the envelope and submits
    F->>API: GET proposal (polls) — sees status "ready"
    F->>F: Freighter signTransaction(ready_xdr)
    F->>API: POST /transactions/submit<br/>{ xdr, refreshContractId, completeProposalId }
    API->>RPC: sendTransaction + poll getTransaction
    RPC->>RPC: verify all three parties' auth;<br/>status := Disputed -> resumed (normal processing)
    RPC-->>API: SUCCESS { hash }
    API->>DB: markCompleted(proposalId) — best-effort
    API->>RPC: getCommitment — best-effort cache refresh
    API-->>F: { status:"SUCCESS", hash }
    end

    rect rgb(255, 250, 230)
    note over F,RPC: Escalation if the parties never agree — no signatures needed
    F->>API: POST /commitments/:id/tx/expire_dispute_window { sourcePublicKey }<br/>(generic no-arg route; the method name is the literal underscored name)
    API-->>F: { xdr }
    F->>API: sign, then POST /transactions/submit { xdr, refreshContractId }
    API->>RPC: sendTransaction (anyone may be the source — permissionless)
    RPC->>RPC: only succeeds once the contract's own dispute deadline has passed;<br/>restores the same pre-dispute status
    end
```

GitHub renders Mermaid fenced blocks natively; the same block works in the
VS Code Markdown preview (with Mermaid support) and at <https://mermaid.live>.

## Where each piece lives

| Step | What | Where |
|---|---|---|
| Flag | Builds unsigned `flag_dispute(flagger)` XDR. Validates address format only; `flagger` being a party is enforced on-chain (`NotAParty`). | `src/server.ts` `POST /commitments/:contractId/tx/flag-dispute` |
| Flag | Frontend: build → `signTransactionXdr` → `submitTx(signedXdr, contractId)`. Button only shows for a connected party when status is `Locked`…`Delivered`. | `DisputeSection.tsx` `handleFlag`, `DISPUTABLE_STATUSES` |
| Propose | Idempotent (200 existing / 201 new). Any of buyer, cooperative, warehouse operator may propose — unlike `reassign_buyer`, which is current-buyer only. 403 otherwise, from a live chain read. | `src/server.ts` `POST .../tx/resolve-dispute/propose` |
| Propose | Active proposal lookup for polling: `{ proposal: … \| null }`. | `src/server.ts` `GET .../tx/resolve-dispute/propose` |
| Sign | Method-agnostic route shared with `cancel` / `reassign_buyer`. Rejects non-UUID `proposalId`, malformed XDR, expired proposals, non-`pending` status, wrong signer or a repeat signature (all 400). | `src/server.ts` `POST .../tx/propose/:proposalId/sign`, `signProposalEntry` |
| Ready | When the last entry is signed the API rebuilds the transaction (deliberately no re-simulation) and flips the row to `ready`. | `finalizeMultiPartyProposal` (`src/stellar/multiParty.ts`), `markReady` |
| Finalize | Proposer signs `ready_xdr` classically and submits with `completeProposalId`; cache refresh and `markCompleted` are best-effort and never mask a landed transaction. | `POST /transactions/submit`, `markCompleted` |
| Escalation | `expire_dispute_window` is permissionless and no-arg, so it goes through the generic `POST .../tx/:method` route (`expire_dispute_window`, underscored). It restores the pre-dispute status after the contract's deadline. | `NO_ARG_METHODS` in `src/server.ts`; `../README.md` "Dispute flagging" |

## Edge cases and checks

- **Flag authorization is on-chain only.** The API does not compare
  `flagger` to the commitment's parties; the contract rejects non-parties
  with `NotAParty`. The frontend additionally hides the button for
  non-parties.
- **Reachable states for flagging:** `Locked` through `Delivered`; never
  `Draft` (nothing escrowed) or a terminal status. Simulation fails
  otherwise and the contract error is surfaced.
- **Idempotent propose:** a second propose while a `resolve_dispute`
  proposal is active returns the existing one. `cancel` and
  `reassign_buyer` proposals are separate rows and do not collide.
- **Expiry:** proposals expire after 24 h; signing an expired proposal is
  rejected with "propose a fresh one". The auth entries also carry a
  ~24 h on-chain `signatureExpirationLedger`.
- **Pending entries:** `pending_entries` lists only still-unsigned
  entries, so once you've signed you no longer appear in it. The UI tracks
  that with local `justApproved` state.
- **No arbitration.** Resolving needs unanimous consent; nothing decides
  who was right. `expire_dispute_window` exists so one party refusing to
  sign cannot freeze the commitment forever. It does not pause other
  deadlines already running on the commitment (see
  `HarvestLock-Contracts/HANDOFF.md`, Deployment 10).
- **Alternative to staging:** `resolve_dispute` is also in the generic
  no-arg method set, so three parties can sign the same envelope in turn
  instead of using the staged flow.
