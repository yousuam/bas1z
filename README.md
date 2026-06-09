# bas1z
0x35Ab7Ad4969E3eeE69B697cB26237E24EB2275Ab
import time
from collections import defaultdict, Counter
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 30
FUNDER_THRESHOLD = 5
ACTIVITY_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting pre-activity funding patterns...\n")

    last_block = w3.eth.block_number

    wallet_funders = defaultdict(Counter)
    wallet_activity = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )


                    if (
                        sender != ZERO
                        and receiver != ZERO
                    ):

                        wallet_funders[receiver][sender] += 1

                        wallet_activity[receiver] += 1


                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, funders in wallet_funders.items():

                    if wallet_activity[wallet] < ACTIVITY_THRESHOLD:
                        continue

                    top_funders = [
                        f for f, count
                        in funders.items()
                        if count >= FUNDER_THRESHOLD
                    ]

                    if top_funders:

                        print("💰 Pre-Activity Funding Pattern")
                        print("Wallet:", wallet)
                        print(
                            "Funding wallets:",
                            len(top_funders)
                        )

                        for f in top_funders[:3]:
                            print(" -", f)

                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
