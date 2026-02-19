# How to Get a Failed Transaction with Revert Trace in Otterscan

## Prerequisites
- Docker & Docker Compose
- `curl`
- The playground `.env` configured (copy `.env.example` → `.env` and set your `ETH_RPC_URL`)

## What Changed
- Added `--steps-tracing` flag to Anvil in `docker-compose.fork.yml`
  - This enables `debug_traceTransaction` support, which Otterscan needs to render traces (both successful and failed txs)
  - Without this flag, the Trace tab shows "Loading..." indefinitely

## Steps

### 1. Start the playground

```bash
cd playground
docker compose -f docker-compose.fork.yml up --build -d
```

First build takes a while (Rust compilation). Monitor with:
```bash
docker compose -f docker-compose.fork.yml logs -f
```

Wait until all services are healthy (`docker compose ps`).

### 2. Force a failed transaction

Send a WETH `transfer()` with an amount larger than the account's balance:

```bash
curl http://localhost:8545 -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_sendTransaction","params":[{"from":"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","to":"0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2","data":"0xa9059cbb0000000000000000000000000000000000000000000000000000000000000001ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"}],"id":1}'
```

This calls `WETH.transfer(0x0...01, type(uint256).max)` from the Hardhat test account, which will revert due to insufficient balance.

You'll get back a tx hash:
```json
{"jsonrpc":"2.0","id":1,"result":"0x<TX_HASH>"}
```

### 3. Verify it failed

```bash
curl -s http://localhost:8545 -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt","params":["<TX_HASH>"],"id":1}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin)['result']; print(f'status: {r[\"status\"]}')"
```

Should print `status: 0x0` (= failed).

### 4. View in Otterscan

Open `http://localhost:8003/tx/<TX_HASH>` in your browser.

- **Overview tab**: Shows the red "Fail" badge + "Show Revert Trace" button
- **Trace tab**: Shows the call trace for the failed transaction (e.g., `call Wrapped Ether (WETH) [C] . transfer ({ ... })`)

### 5. Take screenshots

You need:
1. **Overview** of the failed tx (showing "Fail" badge)
2. **Trace tab** of the failed tx (showing the revert trace loaded)

These go into `playground/docs/images/` as:
- `otterscan-failed-tx.png`
- `otterscan-revert-trace.png`

## Notes
- The `--steps-tracing` flag adds some overhead to Anvil. It's acceptable for a dev playground.
- The test account `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` is Hardhat account #0 (private key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`).
- For a more realistic failed tx (e.g., a CoW settlement failure), you'd need to manipulate state mid-solver-execution, which is non-trivial since solvers simulate before submitting.
