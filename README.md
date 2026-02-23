# base1231import time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

# keccak256("Transfer(address,address,uint256)")
TRANSFER_TOPIC = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a5f7b3b1a"

SCAN_BLOCK_WINDOW = 20
TOP_N = 5

ERC20_ABI_MIN = [
    {
        "name": "symbol",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "string"}],
    },
    {
        "name": "decimals",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "uint8"}],
    },
]


def safe_call(fn, fallback):
    try:
        return fn()
    except Exception:
        return fallback


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print(f"Scanning last {SCAN_BLOCK_WINDOW} blocks...")

    last_scanned = w3.eth.block_number

    while True:
        current_block = w3.eth.block_number

        if current_block >= last_scanned + SCAN_BLOCK_WINDOW:

            from_block = current_block - SCAN_BLOCK_WINDOW
            to_block = current_block

            print(f"\nAnalyzing blocks {from_block} -> {to_block}")

            inflow_totals = defaultdict(float)
            token_meta_cache = {}

            logs = w3.eth.get_logs(
                {
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "topics": [TRANSFER_TOPIC],
                }
            )

            for log in logs:
                token_addr = Web3.to_checksum_address(log["address"])

                if token_addr not in token_meta_cache:
                    token = w3.eth.contract(
                        address=token_addr, abi=ERC20_ABI_MIN
                    )
                    symbol = safe_call(
                        lambda: token.functions.symbol().call(),
                        "UNKNOWN",
                    )
                    decimals = safe_call(
                        lambda: token.functions.decimals().call(),
                        18,
                    )
                    token_meta_cache[token_addr] = (symbol, decimals)

                symbol, decimals = token_meta_cache[token_addr]

                raw_value = int(log["data"], 16)
                value = raw_value / (10 ** decimals)

                inflow_totals[token_addr] += value

            # Sort tokens by total inflow
            sorted_tokens = sorted(
                inflow_totals.items(),
                key=lambda x: x[1],
                reverse=True,
            )

            print("\nTop token inflows:")
            for token_addr, total in sorted_tokens[:TOP_N]:
                symbol, _ = token_meta_cache[token_addr]
                print(f"{symbol}: {total}")

            last_scanned = current_block

        time.sleep(5)


if __name__ == "__main__":
    main()
