# TTL Radar

Plan and run Soroban restores that are too large for one transaction.

> **Status: in development.** The first release (`plan` and `restore`) is being built now. The commands below describe the planned interface and do not work yet. Watch or star the repo to hear when the first version is out.

## Why this exists

Soroban rents storage. Every contract entry has a TTL (time to live), and no extension can go past about 180 days (`max_entry_ttl`). When a persistent or instance entry runs out, the network archives it, and the contract cannot use it until someone pays to restore it.

Restore has a hard size limit. One mainnet transaction can touch at most 200 entries and write at most 132,096 bytes. A contract with more archived state than that cannot be restored in one transaction. Automatic restore does not remove the problem, because the transaction that triggers it has the same limits.

Today the operator works out by hand which keys go into which transaction and what each one costs. TTL Radar does that work: it splits the restore into the fewest transactions that fit, shows the fee of each, and runs them safely.

## Planned usage

### Install

```
npm install -g ttl-radar
```

Requires Node 20 or later. Not published yet.

### Plan

```
ttl-radar plan <CONTRACT_ID> --network mainnet --keys keys.json
```

`plan` reads the contract's entries, finds the archived ones, and prints a restore plan:

- how many transactions the restore needs
- which ledger keys go into each transaction
- the simulated fee of each transaction and the total

Entries that are already live are skipped. Nothing is signed or sent. Add `--json` to save the plan as a file.

The network has no way to list a contract's storage keys. `plan` finds the instance and code entries from the contract ID. Data entries come from a key file, or from the StellarExpert API with `--from-explorer`:

```json
{
  "persistent": ["COUNTER", { "xdr": "AAAAB..." }]
}
```

### Restore

```
ttl-radar restore plan.json --max-total-fee 5
```

- Each transaction is simulated again right before it is sent.
- `--max-total-fee` (in XLM) stops the run before the total goes over the limit.
- If a transaction fails or the run is interrupted, run the same command again. It continues from the first entry that is still archived and never pays twice for an entry that is already live.
- The signing key is read from the `TTL_RADAR_SECRET` environment variable or `--secret-file`. It stays on your machine and is never logged.
- `--unsigned` writes each transaction as XDR for an external signer instead of sending it.

### Networks

Mainnet and testnet, with public RPC endpoints by default. Point at your own node with `--rpc-url`.

## Network limits

TTL Radar reads these from the network every time it runs. They are network settings and can change in a protocol upgrade. Mainnet values at the time of writing:

| Setting | Value |
|---|---|
| Entries one transaction can read from disk | 200 |
| Entries one transaction can write | 200 |
| Bytes one transaction can write | 132,096 |
| Entries in one transaction's footprint | 400 |
| Maximum TTL for any entry (`max_entry_ttl`) | 3,110,400 ledgers, about 180 days |

## Roadmap

| Stage | Scope | Status |
|---|---|---|
| 1 | `plan` and `restore` on mainnet and testnet, with a tested limit engine | in progress |
| 2 | `scan` to list entries by urgency, `extend` planned with the same limit engine, `check` with thresholds, alerts by webhook and Telegram, exit code for CI | planned |
| 3 | `watch` for many contracts from one configuration file, v1.0 on npm, core library as its own package, stellar-cli plugin (`stellar ttl-radar`) | planned |

Not planned: a hosted service, a web dashboard, or anything that holds your keys.

## Project layout

```
packages/core/   network config reader, limit engine, ledger keys, transaction building
packages/cli/    the ttl-radar command
examples/        sample keys.json and a test contract
```

The core library is kept separate from the CLI so other tools can reuse the limit engine.

## Feedback and contributing

Issues and pull requests are welcome. If a plan looks wrong, open an issue with the contract ID, the network and the key file you used.

## License

Apache-2.0. See [LICENSE](LICENSE).
