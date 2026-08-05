# jochemnet stateful benchmark runbook

How to build the prestate, fill the repricing benchmark fixtures against it, and replay them under benchmarkoor. Everything here was executed on the VPS at `10.10.101.22` (user `devops`); paths are as they exist there.

The whole pipeline rests on one invariant: **the client datadir head must equal the fixtures' `snapshotBlockHash`**. Every fixture produced here carries block `24,411,765` / `0xd6d937d2705f3f8a1abd3bb73eb8475d987d9d29eec4a280507f103aaec9489e` and **zero** `setupEngineNewPayloads`, because the entire prestate is already committed on chain at that block. A datadir at any other head fails on parent hash. If you rebuild the prestate and the head lands somewhere else, every fixture must be refilled.

---

## Phase 0 — substrate (once)

Download the jochemnet geth snapshot at block 24,402,727 (~1.0 TB) and put it behind an OverlayFS so the download is never mutated and any change can be discarded:

```
https://snapshots.ethpandaops.io/jochemnet/geth/24402727/snapshot.tar.zst

lowerdir = /home/devops/bench_state/jochemnet-dd   # pristine, never written
upperdir = /srv/geth-ovl/upper                     # every prestate block lands here
workdir  = /srv/geth-ovl/work
merged   = /srv/geth-ovl/merged                    # bind-mounted into geth as /data
```

reth and Besu snapshots exist at the same block (`.../jochemnet/{reth,besu}/24402727/snapshot.tar.zst`, ~1.0 TB each) but are not interchangeable with the geth datadir — there is no conversion path.

Genesis (needed only to learn the fork schedule, not passed to geth): jochemnet, chainId 1, `amsterdamTime = 1769856767`, 8,893 alloc entries. Geth flavour lives at `~/bench_state/gistfile1.json`; the Besu-flavoured twin is [this gist](https://gist.githubusercontent.com/skylenet/6c7d89dc5646e074349f0dcee301f709/raw/7fee6c1db02037444b6780301aad077dc5b6189c/genesis-jochemnet-24402727-amsterdam-besu.json).

Boot geth (container `jn-fill`, image `ethpandaops/geth:glamsterdam-devnet-7`) with `/srv/geth-ovl/merged` mounted at `/data`:

```
--datadir /data --networkid 1 --state.scheme=path
--cache 16384 --cache.database 80
--maxpeers 0 --nodiscover
--override.amsterdam=1769856767
--miner.gaslimit=1000000000000
--http --http.addr 0.0.0.0 --http.api eth,web3,debug,net,testing
--authrpc.addr 0.0.0.0 --authrpc.port 8551 --authrpc.vhosts=* --authrpc.jwtsecret /jwt.hex
```

Four of these are load-bearing:

| flag | why |
|---|---|
| `--override.amsterdam=1769856767` | The snapshot's stored chain config has **no** `amsterdamTime`, so without this every built block is judged pre-Amsterdam and `engine_newPayloadV5` rejects it (`nil slotnumber post-amsterdam`, missing `blockAccessList`). The value is snapshot head timestamp + 1. |
| `--miner.gaslimit=1000000000000` | EEST never sends `targetGasLimit`, so geth walks the limit toward its 60,000,000 default at 1/1024 per block. Left unset, a long fill decays 1e12 → 969 G → 51.6 G → 2.95 G. |
| `--http.api …,testing` | `fill-stateful` drives `testing_buildBlockV1`; without the namespace there is nothing to drive. |
| `--http.api …,debug` | The between-test rewind uses `debug_setHead`. |

**Do not guess the Amsterdam timestamp.** Deriving it from a stale snapshot file once produced `1769804040`, which activated Amsterdam 52,727 blocks in the past, invalidated them, and cost a multi-hour PathDB reverse-diff rollback with a 74 GB overlay. The controlled check is simple: same image, only the flag differing, gives 0 versus 52,727 rewinds.

---

## Phase 1 — build the prestate

**Tooling:** execution-specs branch `prestate/jochemnet-deployers`, plus the `--keep-chain-head` patch (branch `feat/fill-stateful-keep-chain-head`, also saved as `~/02-keep-chain-head.patch`).

`--keep-chain-head` is mandatory here. `fill-stateful` normally rewinds to the start block after every test so each fill measures from identical state; that default would discard each prestate stage as soon as it was built.

Every stage runs through `~/fill-stage.sh <outdir> <test-id>`:

```bash
fill-stateful --fork Amsterdam \
  --rpc-endpoint http://127.0.0.1:18545 \
  --engine-endpoint http://127.0.0.1:18551 \
  --engine-jwt-secret-file ~/jwt.hex \
  --rpc-seed-key 0xf045b08fc5e0879abb0e046778c040c46da136848f452ec0a1193deaef12bc76 \
  --keep-chain-head \
  --output ~/prestate-v2/<outdir> --clean \
  "tests/benchmark/stateful/prestate/test_deploy_prestate.py::<test-id>"
```

Stages, strictly in order — each builds on the previous, so a failure must stop the chain rather than continue:

| stage | test id | result |
|---|---|---|
| `01-funding` | `test_fund_sender_pool` | 30,000 deterministic senders + the legacy fill seed, credited by withdrawals in 3 blocks of 15,000 |
| `02-minimal` | `test_deploy_create2_targets[fork_Amsterdam-blockchain_test_stateful_engine-EXISTING_CONTRACT_MINIMAL]` | 150,000 CREATE2 contracts, runtime code `0x00` |
| `03-same-max` | …`-EXISTING_CONTRACT_SAME_MAX]` | 150,000 × 24,576 B, identical code across salts |
| `04-diff-max` | …`-EXISTING_CONTRACT_DIFF_MAX]` | 150,000 × 24,576 B, each embedding its own address at bytes 12–31 |
| `05-delegations` | `test_delegate_7702_authorities` | 150,000 EIP-7702 designators `0xef0100 ++ DIFF_MAX[i]` |

`~/fill-all-v2.sh` drives 02–04 and aborts on the first failure. Targets are CREATE2 deployments from `DETERMINISTIC_FACTORY_ADDRESS` with salt = index, so their addresses are computable without any registry.

Result: head advances 24,402,727 → **24,411,765** across 9,038 blocks. Payloads are written to `~/prestate-v2/<stage>/` and are the durable artifact — the chain itself only exists in `/srv/geth-ovl/upper`.

### Sizing constraints that are not obvious

- **50 deploys per block.** EIP-7928 puts deployed code in the block access list, so 5,000 × 24,576 B would be ~123 MB of BAL (~246 MB as hex on the wire) and blow the 5 MB engine request limit. At 50 per block the BAL is ~1.23 MB and the whole request ~2.49 MB.
- **15,000 withdrawals per block.** A withdrawal is ~150 B of payload JSON *and* a BAL balance-change entry; all 30,001 in one block approaches the same 5 MB limit. `WITHDRAWALS_PER_BLOCK = 15_000` is the proven size.
- **30,000 funded senders.** `test_ether_transfers_onchain_receivers[case_id_diff_to_self]` spends one distinct sender per transaction at 12,000 gas each, so a benchmark targeting *G* gas needs *G*/12,000 senders — 25,000 at 300 Mgas. An earlier `FUNDED_SENDER_COUNT = 15_000` covered exactly up to 180 Mgas and made every transfer case fail from 200 Mgas with `insufficient funds for gas * price + value: … have 0`. `yield_distinct_sender()` counts upward without bound, so overrunning the funded set is silent until a block build fails.

### Verify before moving on

After each stage: head advanced as expected, `gasLimit` still exactly `1000000000000`, and code sizes correct at both ends of the salt range (salt 0 and salt 149,999 populated, salt 150,000 empty). Check a couple of delegations resolve to `0xef0100 ++ DIFF_MAX[i]`, and that senders at index 0, 15,000 and 29,999 all hold balance.

---

## Phase 2 — fill the benchmark fixtures

**Tooling:** execution-specs `perf/batch-transaction-receipts` @ `3e4ad1e8b`, plus the block-RLP-size bound patch (`perf/bound-block-rlp-size`). **Not** `--keep-chain-head` — here the between-test rewind is exactly what you want, so all 1463 fixtures anchor at the same prestate.

Scope is everything marked `@pytest.mark.repricing` under `tests/benchmark/stateful/bloatnet`: **133 cases**.

| cases | test |
|---|---|
| 100 | `test_account_access` (`test_account_query.py`) |
| 18 | `test_ether_transfers_onchain_receivers` (`test_transaction_types.py`) |
| 7 | `test_ext_account_query_warm` (`test_account_query.py`) |
| 4 | `test_sstore_bloated` |
| 2 | `test_sload_bloated` |
| 2 | `test_sload_same_key_benchmark` |

Across 11 gas values (100, 120, …, 300 Mgas) that is **1463 fixtures**.

```bash
GAS_VALUES="100 120 140 160 180 200 220 240 260 280 300" bash ~/fill-repricing-v2.sh
```

which runs, once per gas value:

```bash
fill-stateful --fork Amsterdam \
  --rpc-endpoint http://127.0.0.1:18545 \
  --engine-endpoint http://127.0.0.1:18551 \
  --engine-jwt-secret-file ~/jwt.hex \
  --rpc-seed-key 0xf045b08fc5e0879abb0e046778c040c46da136848f452ec0a1193deaef12bc76 \
  --snapshot-block 0xd6d937d2705f3f8a1abd3bb73eb8475d987d9d29eec4a280507f103aaec9489e \
  -m repricing \
  --gas-benchmark-values <G> \
  --address-stubs tests/benchmark/stateful/stubs/stubs_prestate_repricing.json \
  --output ~/repricing-fixtures/gas-<G padded to 4>M --clean \
  tests/benchmark/stateful/bloatnet
```

Notes on the flags:

- `--gas-benchmark-values` is the **benchmark gas target per test**, in millions — not the block gas limit, which stays at 1e12. It also groups output under `for_amsterdam_at_XXXXM/`.
- `--snapshot-block` accepts a block hash, which survives a reorg on the live client; a bare number does not.
- `--address-stubs` supplies `bloated_eoa_10GB` (`0x87a6314da5ac8832f6e7a176c8fb133b19f5be04`, a delegated EOA with ~10 GB of storage). Tests whose stub prefix has no match are skipped with a warning rather than failing, so check the warning list matches what you expect.
- One invocation per gas value is deliberate: fixtures flush at *module* end, so a crash mid-run loses only the value in flight. A `.done-<TAG>` marker makes a rerun skip completed values, and only the directory of the value being refilled is ever removed.

Then merge and archive:

```bash
bash ~/merge-repricing.sh      # 11 trees -> ~/repricing-fixtures/merged, via hardlinks
bash ~/archive-repricing.sh    # tar.zst + provenance README, and re-tars prestate-v2
```

The merge is a plain copy because the gas subfolder is zero-padded to width 4 for any maximum below 10,000 — so `for_amsterdam_at_0200M` is byte-identical whether you passed `100..300` or `200..300`, and the eleven trees have disjoint names.

Progress: `bash ~/fill-progress.sh ~/fill-repricing-v3.log` reports the current test index by reading the last `[N/133]` counter and adding the result characters written since it.

### Fill performance

Fill cost is dominated by EEST's Python-side handling of the block access list, not by the client. Measured over a 24-minute window (284 tests, 546 blocks): geth executed for **27.7 s of 1467 s — 1.9%** of wall time, at 859 Mgas/s. The BAL is ~450× the size of the block's transactions on account-access benchmarks (0.22–1.06 MB per block).

Two fill-side optimizations were landed and are verified not to change fixture bytes:

| branch | change | effect |
|---|---|---|
| `perf/bal-redundant-rlp-work` (`6355360eb`) | On the client path the BAL hash EEST asserted against was its own recomputation from the same bytes — `a == a`. Gated on a new `FillerBackend.attests_block_access_list_hash`, and the client path now hashes the raw bytes. | removes one of two BAL decodes and both re-encodes |
| `perf/bound-block-rlp-size` | The EIP-7934 size check built an entire `FixtureBlock` (a pydantic model per transaction and receipt, then `model_dump`) just to measure a length. Now an arithmetic upper bound is computed first, and the exact RLP only when that bound could exceed the 8,388,608-byte limit. | ~36% of wall time on transaction-heavy blocks |

Combined, measured like-for-like on identical work: **0100M 619 s → 408 s (1.52×)**, **0120M 757 s → 525 s (1.44×)**.

---

## Phase 3 — run the benchmarks

```bash
docker stop jn-fill                                   # free the datadir
sudo benchmarkoor run --config jn-repricing.yaml
```

Stopping geth first is not optional: it writes to the same overlay upper that benchmarkoor reads. Root is required for the overlay mount and for `drop_memory_caches`.

benchmarkoor stacks its own throwaway overlay on top of `/srv/geth-ovl/merged` — overlay-on-overlay, verified working on this kernel — so a run can never mutate the prestate and rollback is just discarding that upper.

Narrow the set while iterating:

```bash
TEST_FILTER=0200M                        sudo benchmarkoor run --config jn-repricing.yaml
TEST_FILTER=case_id_diff_to_contract_diff_max  sudo benchmarkoor run --config jn-repricing.yaml
```

benchmarkoor has no built-in repetition: invoke it N times for N samples, each writes its own results directory and the suite stats aggregate across them. 1463 fixtures × (client restart + cache drop) is many hours, so filter for anything exploratory.

The config is [`jn-repricing.yaml`](../jn-repricing.yaml). A benchmarkoor build older than `e49938c` logs a spurious "missing pre_run" warning for these fixtures — harmless, fixed upstream by `statefulPreRunMissing()`.

---

## Artifacts

| path | what |
|---|---|
`~/prestate-v2/01-funding` … `06-sender-topup` | prestate stage payloads (the reproducible recipe) |
`~/prestate-v2.tar.zst` | archived prestate payloads |
`~/repricing-fixtures/gas-XXXXM/` | per-gas-value fixtures |
`~/repricing-fixtures/merged/` | what benchmarkoor consumes |
`~/repricing-fixtures.tar.zst` + `.README` | archived fixtures with provenance |
`~/repricing-fixtures-anchor-24411762/` | superseded fixtures from the 15k-sender anchor, kept as a timing baseline |
`/srv/geth-ovl/upper` | the only copy of the prestate chain — the overlay upper is not backed up |

`06-sender-topup` exists because the sender pool was raised from 15,000 to 30,000 *after* the rest of the prestate was built; appending three withdrawal blocks was far cheaper than replaying 9,035 blocks. A clean rebuild folds it into `01-funding`, since `FUNDED_SENDER_COUNT` is now 30,000 — the resulting state is equivalent but not block-identical, and the anchor hash will differ.

---

## Pitfalls hit while building this

- **Wrong `--override.amsterdam`** — derived from a stale snapshot file, activated Amsterdam 52,727 blocks in the past, multi-hour PathDB rollback. Always compute it from the actual head timestamp.
- **`413` on `testing_buildBlockV1`** — all 150k transactions in one request. Chunk by transaction count as well as by state gas.
- **`413` on `engine_newPayloadV5`** — the BAL carries deployed code. Cap deploys per block.
- **Gas limit decay** — see `--miner.gaslimit` above.
- **Chain rewinding between prestate stages** — needs `--keep-chain-head`.
- **Unfunded senders** — the sender generator is unbounded; the funded pool is not. Size it from the highest gas value you intend to fill.
- **Filling over an SSH/RPC tunnel** — one `eth_getTransactionReceipt` per transaction at 65 ms RTT dominated everything. Run on the server (RTT 0.41 ms) and use the receipt-batching branch; MINIMAL went 10m36s → 3m38s.
- **Killing a fill** — the driver's child can outlive its parent's SIGTERM, and a killed fill leaves the head above the anchor. Kill the child first, then re-anchor with `debug_setHead` before restarting.
