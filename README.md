# EscrowCourt — Internet Court Dispute Resolution for x402 Agent Jobs

EscrowCourt is a GenLayer **Intelligent Contract** that acts as the
**verification & disputes layer of the Internet Court** for agentic
commerce jobs paid with **x402** payment requirements: when a client
and a freelancer disagree about whether a deliverable meets the
agreed requirement, the court reads the evidence itself — a public
web page — and GenLayer's validators reach consensus on the ruling.

Built for the GenLayer Portal mission **"Internet Court: Partner
Demo Lab"**. Partner product integrated: **x402** (payment & escrow
layer of the Internet Court stack).

## The problem

Agent-to-agent and client-to-freelancer jobs increasingly follow the
Internet Court stack:

| Layer | Standard / product |
| --- | --- |
| Contracts & obligations | ERC-8183-style job terms |
| Payment & escrow | **x402** payment requirements |
| Verification & disputes | **GenLayer** (this contract) |

The first two layers are mechanical: terms are written down, payment
is escrowed against them. The third layer is not mechanical at all.
"Does this deliverable match what was promised?" is a judgement
call over unstructured evidence — a repository, a published page, a
demo. A classic smart contract cannot even open the evidence page.
EscrowCourt closes that gap.

## How it works

1. **Open a case** — the client records the job on-chain: the
   freelancer's address, the agreed deliverable requirement in plain
   language, and the x402-style payment terms for the job's escrow
   (`scheme: "exact"`, network, asset, amount, `payTo`).
2. **Submit the deliverable** — the freelancer submits a public
   evidence URL plus a short summary.
3. **Request a ruling** — either party calls `request_ruling()`. A
   leader validator fetches the evidence page
   (`gl.nondet.web.render`) and asks an LLM to rule whether the page
   shows the requirement fulfilled (`gl.nondet.exec_prompt`, JSON
   response). Every other validator independently repeats the fetch
   and the judgement, and accepts the leader's ruling only if its
   own ruling matches it **exactly** (`gl.vm.run_nondet_unsafe` with
   a custom validator function) — GenLayer's equivalence principle
   applied to a ruling instead of to free text. The reasoning text
   is never compared, only the normalised verdict:
   `PAY_FREELANCER` or `REFUND_CLIENT`.
4. **Settle** — the ruling and its decisive evidence are stored
   on-chain; the settlement executed against the x402 payment terms
   is recorded (`record_settlement`) and the case closes. Every step
   is appended to a public, append-only history (`get_history`).

The evidence page is treated as untrusted input throughout: page
content is fenced inside the judging prompt as data only, and the
prompt explicitly instructs the model never to follow instructions
found inside the page.

## Contract API

| Method | Type | Description |
| --- | --- | --- |
| `open_case(job_title, freelancer, requirement, asset, network, amount)` | write | Open a dispute-ready job case with x402 payment terms; returns the case id |
| `submit_deliverable(case_id, evidence_url, summary)` | write, freelancer | Submit the deliverable evidence URL |
| `request_ruling(case_id)` | write, either party | Fetch evidence + LLM judgement + validator consensus on the ruling |
| `record_settlement(case_id, settlement_ref)` | write, either party | Record the settlement executed against the ruling; closes the case |
| `get_case(case_id)` | view | Full case record: parties, requirement, payment terms, evidence, ruling, status |
| `get_cases()` | view | Summary list of all cases |
| `get_history()` | view | Append-only log of every event (JSON strings) |
| `get_config()` | view | Court configuration and scope |

Constructor: `EscrowCourt()` — no arguments. The deployer becomes
the contract owner.

Case lifecycle: `OPEN` → `EVIDENCE_SUBMITTED` → `RULED_PAY` /
`RULED_REFUND` → `CLOSED`. After a `REFUND_CLIENT` ruling the
freelancer may submit a new deliverable and request a fresh ruling.

## Live demo (GenLayer Studio / Studionet)

<!-- DEMO-PROOF:BEGIN -->
- Contract: `escrow_court.py`
- Network: GenLayer Studio (Studionet)
- Deployed from (owner): `0x0d1e8B737614bCb563A4BaB329bC257D200B42B4`
- Contract address: `0xbefA8F8F52d506baB4bAE6816aCdBCEbC9be78B0`
  ([explorer](https://explorer-studio.genlayer.com/address/0xbefA8F8F52d506baB4bAE6816aCdBCEbC9be78B0))
- Deploy tx: `0x3b75e3f3e119a45c5b55c8196fe6f27ca5b1d9db102597a8b296f9572e102a79` — FINALIZED, 2026-11-25

Demo case `case-1` — "Demo job: GenLayer price-oracle repository"
(client `0x0d1e…42B4`, freelancer `0x8590e584E02D17D22B58D2f54f6E13f8089934eb`,
x402 terms: 100 USDC, scheme `exact`, network `base-sepolia`):

| Step | Tx | Result |
| --- | --- | --- |
| `open_case` | `0x702a28b932f11e390736fa8fce5a48db84479c72b2cd22e8d0a7c3c93b47c92c` | FINALIZED, case id `case-1` |
| `submit_deliverable` (evidence: https://github.com/Yanasari/hargapangan) | `0x6fae30440ed2a1a475edb6adf885846326ad4847ea7da24944c0c0265411ae2a` | FINALIZED |
| `request_ruling` | `0x2d3895a304a28a2931ebdc7a59885cfd6d1b36b266de4e79fd1db32e17aceb1a` | FINALIZED — ruling **PAY_FREELANCER** |
| `record_settlement` (`demo-settlement-001`) | `0xb7407b3a56d30708ed31bd70e8002bbf05bc9c83e1a72c1db533d0dc95a9a945` | FINALIZED — case **CLOSED** |

Ruling evidence recorded on-chain: *"The evidence page shows a public
GitHub repository containing 'harga_pangan.py' (source file) and
'README.md' which explains the contract's purpose, how it works, the
API, and deployment details."*
<!-- DEMO-PROOF:END -->

Demo flow shown in the walkthrough video: open a case for a real
deliverable (a public GitHub repository containing an Intelligent
Contract and its README), submit the repository as evidence from
the freelancer account, request a ruling, watch the validators
agree on `PAY_FREELANCER`, and record the settlement — end to end
on Studionet.

## Honest limitations

- **v1 is the dispute and record layer.** It stores cases,
  evidence, rulings and settlement records. Executing the actual
  x402 settlement through a facilitator on an EVM chain is driven
  by the on-chain ruling; wiring a live facilitator is the next
  step, the same way a court judgment precedes enforcement.
- **Evidence must be a publicly readable page.** Pages that refuse
  to load from validator infrastructure cannot be judged; if
  validators cannot agree, the ruling transaction reverts and no
  ruling is written — the court prefers no verdict over a disputed
  one.
- **The ruling is only as good as the requirement text.** Vague
  requirements make vague cases; parties should write requirements
  that a third party can verify from the evidence page alone.

## Relationship to the author's other GenLayer work

EscrowCourt is an original, separate project by the same author as
HargaPangan (an AI web oracle for Indonesian staple food prices)
and ArisanChain (transparent rotating savings groups). Where
HargaPangan applies validator consensus to a number extracted from
a web page, EscrowCourt applies the same equivalence pattern to a
legal-style judgement over a deliverable.
