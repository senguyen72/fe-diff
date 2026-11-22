# fe-diff — Evidence Pack

Evidence for the TheDAO Security Fund proposal **"Fe Compiler Differential Fuzzing Harness (fe-diff)"** ($40,000, 6 months).
This pack proves the audit work the proposal is built on. Nothing here is a claim
about the future — every number below is a measurement from a run whose log is
in `evidence/`.

**Author:** Se Nguyen — independent security researcher (Vietnam), AI-assisted audit workflow (Muse).
**Date:** 22 November 2026. **Status:** audit complete, 0 exploitable bugs found.

## What was audited

**Fe Bountiful** ([fe-lang/bountiful](https://github.com/fe-lang/bountiful)) — 7 fifteen-puzzle
game contracts (`Game`, `Game2D`, `GameEnum`, `GameBitboard`, `GameMonadic`, `GameNested`,
`GameTrait`) + `BountyRegistry`, deployed on Ethereum mainnet 01/05/2026. Prize: 1 ETH
in the registry, 7 challenges × 0.25 ETH nominal (the pot pays only the first 4 claims).

- **Compiler:** official `fe 26.1.0` (prebuilt `fe_linux_amd64`).
- **Source audited:** `fe-lang/bountiful` master at commit `5137aa4`. `contracts/` was last
  changed 22/04/2026 (commit `5d658b8`), **before** the 01/05/2026 deployment — the
  audited source is the deployed source.
- **Rules:** read-only audit. No lock, no claim, no on-chain interaction, no spending.

## Results

| # | Check | Result | Evidence |
|---|---|---|---|
| 1 | Hand-written data re-audit (recomputed programmatically, not read by eye) | `ADJACENCY` 16/16 cells, GameBitboard moves 16/16 rows, pack constants 3/3 — **all correct** | `fe-bountiful-solve.md` §2 (see below) |
| 2 | Rebuild + byte-for-byte comparison vs on-chain | **8/8 runtime bytecodes MATCH** — including `GameBitboard` and `GameTrait`, previously unverified on Sourcify | `evidence/bytecode_compare.json`, `evidence/runtime-bytecode-hex/*.runtime.bin`, `evidence/deployments/mainnet-1_25001932.json` |
| 3 | `fe test` (full repo unit tests) | **90 passed, 0 failed** | `evidence/fe-test.log` |
| 4 | Differential fuzz (`DifferentialFuzz.t.sol`) | **PASS at 256 runs (default) and at 1,024 runs — 1,280 runs/test, 0 divergences.** Each run deploys 7 contracts + ≤16 random moves, comparing move results, board and `isSolved` against a pure-Solidity reference after every move | `evidence/fuzz-1024-runs.log`, `evidence/forge-test-full.log` |
| 5 | Deep walks (`ExploitSearch.t.sol`) | **3 seeds × 2,500 steps, PASS, 0 divergences** | `evidence/forge-test-full.log` |
| 6 | Every first move (`ExploitSearch2.t.sol`) | **All 256 first-move indices × 6 games match the reference, 0 solved.** (The original single `test_everyFirstMove` OOM'd at the harness's 2^30 gas limit — a harness limit, not a contract bug — so it was rewritten as 4 chunked tests, 4/4 PASS) | `evidence/forge-test-full.log`, `evidence/tests/ExploitSearch2.t.sol` |
| 7 | Full Foundry suite | **109 tests PASS, 0 fail (11 suites)** — the fresh re-run on 22/11/2026 passes 109, one more than the 108 reported in the audit: the original single `test_everyFirstMove` also passes when run with `--gas-limit 10000000000` (its earlier OOM was at the harness's default 2^30 gas limit) | `evidence/forge-test-full.log` |
| 8 | On-chain state, re-checked live 22/11/2026 | Registry holds **exactly 1.0 ETH**; all 7 challenges open, unlocked, unsolved; boards still the unsolvable starting state (`0x0EFDCBA987654321`) | `evidence/registry-balance.json` |
| 9 | Post-26.1.0 miscompilation bugs (Fe changelog v26.2.0–v26.7.0: spill-slot reuse, phi codegen, `&&`/`||` short-circuit, array-copy, `continue` in for, StoragePackedArray out-of-bounds write, optimizer -O1/-O2, packed-storage lost write) | **Each cross-checked against the deployed source — none reachable** (no `continue`, no functions returning arrays, `&&`/`||` operands are all pure comparisons, packed-storage did not exist in 26.1.0). The differential fuzz above is the empirical check for all of them on exactly these 7 contracts | `fe-bountiful-solve.md` / v4 report §Fe lead 2 |

**Conclusion: 0 exploitable bugs found.** The source is correct; the only remaining
path to the 1 ETH prize is a compiler miscompilation — which is exactly what fe-diff
(the proposed harness) is built to find.

## How to reproduce

```bash
# 1. Bytecode: compare the hex files with on-chain code
#    addresses in evidence/deployments/mainnet-1_25001932.json
#    eth_getCode(<address>)  ==  evidence/runtime-bytecode-hex/<Name>.runtime.bin

# 2. Fe unit tests (fe 26.1.0)
git clone https://github.com/fe-lang/bountiful.git
cd bountiful && git checkout 5137aa4
cd contracts && fe test            # expect: 90 passed

# 3. Foundry suite + fuzz (Foundry 1.10.0; deep walks need the high gas limit)
forge test --gas-limit 10000000000                      # expect: 109 passed, 0 failed
forge test --match-contract DifferentialFuzz \
  --fuzz-runs 1024 --gas-limit 10000000000              # expect: 2 fuzz tests PASS
```

The test files used are in `evidence/tests/` (`DifferentialFuzz.t.sol` and
`ExploitSearch.t.sol` are the repo's own; `ExploitSearch2.t.sol` is our chunked
rewrite of the original `test_everyFirstMove`).

## Files in this repo

```
README.md                  — this file
evidence/
  bytecode_compare.json    — 8/8 MATCH table (contract, local bytes, on-chain bytes)
  runtime-bytecode-hex/    — locally built runtime bytecode (hex) for the 8 deployed contracts
  deployments/             — mainnet deployment manifest (contract addresses)
  fe-test.log              — fresh `fe test` run, 22/11/2026
  forge-test-full.log      — fresh full `forge test` run, 22/11/2026
  fuzz-1024-runs.log       — fresh DifferentialFuzz campaign, 1,024 runs, 22/11/2026
  registry-balance.json    — fresh eth_getBalance of the registry, 22/11/2026
  tests/                   — the Foundry test files + foundry.toml used for the runs
```

The full narrative audit (per-contract notes, edge cases examined, changelog
cross-check) is in the author's working notes; the numbers above are the
verifiable core.

## Links

- Bountiful repo: https://github.com/fe-lang/bountiful
- Registry (Etherscan): https://etherscan.io/address/0x71fB6afcD98c5BD611712ad4b8D314EF729aa69D
- Fe compiler: https://fe-lang.org (v26.1.0)
- TheDAO proposal draft "Fe Compiler Differential Fuzzing Harness (fe-diff)": held as
  a bid file for the RFP "Community Fuzzing Tooling for Ethereum Ecosystem Projects"
  (vote result 11/12/2026); not submitted as a standalone grant.

## License & disclosures

- This pack's own files (README, `ExploitSearch2.t.sol`) are MIT.
- Upstream files (test files from `fe-lang/bountiful`, bytecode built from its source)
  retain their original license.
- No relationship with Argot Collective (Fe/Sonatina team), the `fe-lang/bountiful`
  authors, Perimeter, or any TheDAO curator/badge holder. No other funding for this
  scope. The Fe Bountiful 1 ETH bounty is a separate prize and is untouched by the
  fe-diff proposal — this funds tooling, not an audit of our own project.
- KYC at payout is acceptable.
