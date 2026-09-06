<img src="assets/wireframe-mesh.svg" alt="Abstract dark wireframe mesh: glowing connected nodes over a perspective grid" width="100%" />

# Susu Protocol — Contracts

[![CI](https://github.com/susu-labs/susu-contracts/actions/workflows/ci.yml/badge.svg)](https://github.com/susu-labs/susu-contracts/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status: Testnet · reviewed](https://img.shields.io/badge/status-testnet%20%C2%B7%20reviewed-orange.svg)](#project-status)
[![Mainnet: readiness implemented · deployment gated](https://img.shields.io/badge/mainnet-readiness%20implemented%20%C2%B7%20deployment%20gated-yellow.svg)](#project-status)

Soroban (Rust/Wasm) smart contracts for **Susu Protocol** — a non-custodial rotating savings
protocol on Stellar. These contracts are the sole financial authority of the system: they hold
every group's funds and release them only to the recipient the schedule names.

> **Reviewed by the maintainer, and not independently audited.** It is deployed to Stellar Testnet,
> where the balances are worthless. Mainnet readiness is implemented and mechanically verified, but
> deployment is deliberately gated until the attestations are satisfied. Read
> [Project status](#project-status) before you read anything else.

---

## The system

Susu is four repositories. This one holds the money.

| Repository | Responsibility | Runs on |
| --- | --- | --- |
| **`susu-contracts`** *(you are here)* | Soroban contracts. The financial authority. | **Testnet** |
| [`susu-indexer`](https://github.com/susu-labs/susu-indexer) | Reads chain events, records them in Postgres on a schedule. | **Testnet** (Supabase Cron) |
| [`susu-api`](https://github.com/susu-labs/susu-api) | Read model, accounts, invites, notifications, transaction preparation. | Local |
| [`susu-web`](https://github.com/susu-labs/susu-web) | The client. | Local |

The chain is the source of truth. Everything else is a convenience over it that can be deleted
without affecting a single balance.

## Project status

**Testnet. Reviewed by the maintainer. Mainnet readiness implemented — deployment gated on attestations.**

All twelve planned build phases are implemented, and the canonical lifecycle has been executed
end-to-end against the deployed Testnet contracts with balances asserted from chain state.

Two gates stand between this and Mainnet. The machinery for both is written; neither is satisfied:

| Gate | State |
| --- | --- |
| **Independent security review** | **Not commissioned. The maintainer has reviewed this code, which is a different claim.** What an independent reviewer needs is written and waiting in [`docs/AUDIT_SCOPE.md`](docs/AUDIT_SCOPE.md), and the code to review is frozen at the annotated `audit-freeze-1` tag. |
| **Mainnet readiness** | **Implemented and mechanically verified; `NO-GO` until the attestations are satisfied.** [`scripts/check-mainnet-readiness.sh`](scripts/check-mainnet-readiness.sh) verifies the mechanical gates and refuses to pass while the audit and the approval attestation are absent. |

Nothing is deployed to Mainnet, and the code refuses to write there.

## Contents

- [What Susu is](#what-susu-is)
- [Architecture](#architecture)
- [Financial invariants](#financial-invariants)
- [Layout](#layout)
- [Development](#development)
  - [Testnet](#testnet)
  - [Mainnet readiness](#mainnet-readiness)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

## What Susu is

Members of a group contribute a fixed amount at a fixed interval. Once every member has
contributed for the current round, the pool is paid to the scheduled recipient, minus a
transparent **0.50% (50 bps)** protocol fee sent to a dedicated treasury. Rounds continue
until every member has received exactly one payout.

It is an old arrangement — a *susu*, a *tanda*, a *chit fund* — and it works. What usually
breaks it is having to trust whoever is holding the money. Here, nobody holds it.

## Architecture

**Factory + one Group contract per group.** The Factory creates and registers groups and
never holds funds. Each Group contract is the sole financial authority for its own pool.

```text
Factory ──creates──> Group N   ──> 99.50% scheduled recipient
                              └──>  0.50% treasury
```

## Financial invariants

```text
fee              = amount * fee_bps / 10_000      (integer arithmetic only)
recipient_amount = amount - fee
fee_bps          = 50
fee + recipient_amount == pool
```

- One contribution per member per round.
- One payout per round; no early payout.
- Configured token only, exact configured amount.
- Recipient is derived from the immutable payout order.
- No arbitrary withdrawal — not by creator, admin, backend, or treasury.
- Final round completes exactly once.

**Floating point is never used for money.**

Each of these is traced to the test that proves it by
[`scripts/check-mainnet-readiness.sh`](scripts/check-mainnet-readiness.sh), which runs in CI.

## Layout

```text
contracts/
  factory/   # creates + registers groups; never custodies funds
  group/     # per-group pool; the financial authority
docs/        # contract spec, threat model, deployment + runbook
```

## Development

Prerequisites: Rust ≥ 1.84 (pinned in `rust-toolchain.toml`) with the `wasm32v1-none`
target, and the Stellar CLI.

```bash
rustup target add wasm32v1-none

cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
./scripts/build-contracts.sh   # or: stellar contract build
cargo test -p susu-factory --features wasm-integration
cargo deny check
cargo audit
```

### Testnet

Deployment scripts for Stellar Testnet. Both are idempotent; see
[`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) and [`docs/TESTNET.md`](docs/TESTNET.md).

```bash
./scripts/deploy-testnet.sh   # build, upload, deploy the Factory, verify on-chain
./scripts/e2e-testnet.sh      # full 3x10 lifecycle, plus the refusals, with balance assertions
```

The E2E script retries transport failures (Testnet's RPC is intermittently flaky),
and distinguishes them from contract refusals so a network blip is never reported as
a contract bug. Both scripts hash themselves at startup and re-check on exit, and
refuse to trust a run if they were modified while it was in flight — bash reads
scripts by byte offset, so a mid-run edit would otherwise execute misaligned content.

The deployed Factory, its admin, the treasury and the verified `fee_bps` are recorded in
[`docs/TESTNET.md`](docs/TESTNET.md).

### Mainnet readiness

Not a deployment script — a check. Mainnet is blocked, and the gate that blocks it is verified
rather than remembered; see [`docs/MAINNET_READINESS.md`](docs/MAINNET_READINESS.md).

```bash
./scripts/check-mainnet-readiness.sh     # evidence report; exits non-zero while anything is unmet
```

It traces each of the eleven financial invariants to the test that proves it, proves the
no-arbitrary-withdrawal invariant structurally by enumerating every place the contracts can move
funds, diffs the public interface against a recorded freeze, and requires an independent audit and an
explicit approval to exist as a written attestation. It **cannot pass the gate on its own** — the
audit and the approval are attestations, and while they are absent the verdict is `NO-GO`. That is the
point of it.

CI runs it in `--mechanical-only` mode, so the invariant mapping, the interface freeze and the
Mainnet safety switches cannot regress without failing a build.

There is deliberately no `deploy-mainnet.sh`: Mainnet has no Friendbot, so the identity funding
`deploy-testnet.sh` depends on does not exist there, and a deployment procedure should be reviewed
alongside the contracts rather than written before the audit.

## Documentation

| Document | What it covers |
| --- | --- |
| [`docs/CONTRACT_SPEC.md`](docs/CONTRACT_SPEC.md) | The implemented interface, method by method. |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Storage layout, authorization, and how the two contracts relate. |
| [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) | Trust boundaries, adversaries, and off-chain surfaces. |
| [`docs/AUDIT_SCOPE.md`](docs/AUDIT_SCOPE.md) | The reviewer's brief: scope, assets, what is already known to be weak. |
| [`docs/TESTNET.md`](docs/TESTNET.md) | The live deployment, and the reference 3 × 10 USD scenario. |
| [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | How to deploy, and why Mainnet has no script. |
| [`docs/MAINNET_READINESS.md`](docs/MAINNET_READINESS.md) | Every gate between here and Mainnet, machine-checked or attested. |
| [`docs/RUNBOOK.md`](docs/RUNBOOK.md) | What to do when something is wrong. |

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) and the [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

Changes affecting financial invariants, custody, authorization, payout behavior, or fees
require human review before merge — a green CI run is not sufficient for those.

## Security

Contracts have been **reviewed by the maintainer, not by an independent auditor**. See [`SECURITY.md`](SECURITY.md) for the disclosure process.

## License

[MIT](LICENSE)
