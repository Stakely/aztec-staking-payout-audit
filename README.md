# Stakely Aztec Staking Payout

Public audit record for **Stakely's** Aztec sequencer reward distributions to its delegators.

- Provider page: <https://stake.aztec.network/providers/47>
- Tool source: <https://github.com/AztecProtocol/aztec-staking-payout>

## What this repo is

Stakely operates Aztec sequencers and shares the rewards they collect with delegators,
at a chosen commission rate. The per-delegator amounts are computed off-chain with the
[`aztec-staking-payout`](https://github.com/AztecProtocol/aztec-staking-payout) tool and paid
out **once a week** as a single Multicall3 batch.

This repository is the **public audit trail** for those payouts. After each weekly settlement,
the tool's output is committed under [`runs/`](./runs) so delegators — and any third party — can
verify every distribution without trusting Stakely's word for it.

> [!NOTE]
> The tool **holds no funds, deploys no contracts, and sends nothing by itself**. It produces
> calldata that Stakely executes through its distribution wallet. The numbers here are
> derived entirely from on-chain data and are reproducible by anyone.

## Why a payout tool exists

Provider commission rates in the Aztec staking protocol are baked into each delegation's
coinbase split contract and **cannot be changed** after the fact. To keep commission adjustable
without redeploying contracts, sequencers are configured to send their L2 `coinbase` rewards to
a distribution wallet Stakely controls. Each week the tool figures out which attesters earned
what over a given epoch range, computes per-delegator amounts at the current commission, and
emits a ready-to-sign batch that pays everyone in one transaction.

## What's in `runs/`

Each weekly settlement adds two files for the settled epoch window:

| File | What it is |
|---|---|
| `epoch-<from>-<to>-<runId>.json` | **The audit record.** Epoch + checkpoint + L1 block range, the finalized block, the rollup's `getRewardConfig` snapshot, per-attester checkpoint counts, the full list of attributed checkpoints with their L1 tx hashes, the commission, the per-delegator transfer breakdown, **and the encoded calldata**. Self-contained. |
| `epoch-<from>-<to>-<runId>.safe.json` | The same transactions in Safe Transaction Builder import format. |

## How to verify a distribution (without trusting us)

Everything needed to re-derive a payout is in the audit JSON:

1. Open `epoch-<from>-<to>-<runId>.json`. The `rewardConfig` snapshot and `checkpointsProposed`
   give the total reward (`checkpointsProposed × sequencerRewardPerCheckpoint`); `transfers[]` is
   the per-delegator breakdown.
2. Pick any row in `attributedCheckpoints[]` and look up its `txHash` on a block explorer. The
   `propose()` calldata's signature recovers to that row's `attester`, confirming Stakely
   actually proposed that checkpoint.
3. Sum the proposals per delegator, apply the published commission, and compare to `transfers[]`.
   The numbers must match exactly.
4. Re-run the tool with the published config and the same pinned `--from-epoch`/`--to-epoch`. For
   a fixed epoch window the result is deterministic — you should get byte-identical numbers.

The accuracy guarantees (epoch-aligned windows, L1-finalization gate, protocol-derived reward,
all-or-nothing proposer recovery, full determinism) are documented in the
[tool repository](https://github.com/AztecProtocol/aztec-staking-payout).

## Operator details

| | |
|---|---|
| Operator | Stakely |
| Provider id | 47 |
| Distribution wallet | `PENDING` |