# Backrunning bot

## Graph
Using graphtools (https://graph-tool.skewed.de/), I create a graph of tokens as nodes and LPs as edges by querying two subgraphs (via The Graph - https://thegraph.com/en/). These queries return the current reserves and token addresses needed to build the graph. Once the graph is built, I use a graphtools path finding algorithm to find all cycles of length 2 or 3. I then create a map of pair token address -> pair info (reserves and token address). I then save this map as a JSON file that will be used by my bot. Getting the latest graph should be run on startup of the bot. It only needs to be run on startup (not on-going).


### Note
the subgraph now no longer provides services to assist us in querying the pair list, for those that have a locally run archive node, you can use the following script to query the uniswap v2 factory address and then put that into a pairs.json file.

```javascript
// Import ethers from Hardhat package
const { ethers } = require("hardhat")
// We'll use fs to write JSON to a file
const fs = require("fs")

async function main() {
    // Connect to the network (Change the network in hardhat.config.js if necessary)
    const [deployer] = await ethers.getSigners()

    // Log the deployer address
    console.log("Using account:", deployer.address)

    // Uniswap V2 Factory Contract Address and ABI
    const factoryAddress = "0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f"
    const FactoryABI = [
        "function allPairsLength() view returns (uint)",
        "function allPairs(uint256) view returns (address)",
    ]

    // Create a contract instance
    const factoryContract = new ethers.Contract(
        factoryAddress,
        FactoryABI,
        deployer,
    )

    // Fetch the total number of pairs
    const totalPairs = await factoryContract.allPairsLength()
    console.log(`Total pairs: ${totalPairs}`)

    // We'll store all pair addresses in an array
    let pairsData = []

    // Loop through each index and fetch the pair address
    for (let i = 0; i < totalPairs; i++) {
        const pairAddress = await factoryContract.allPairs(i)
        console.log(`Pair index ${i}: Address - ${pairAddress}`)

        // Push pair address to the array
        pairsData.push(pairAddress)
    }

    // Write the array of pair addresses to a JSON file
    fs.writeFileSync("pairs.json", JSON.stringify(pairsData, null, 2))
    console.log(`Successfully saved ${pairsData.length} pairs to pairs.json`)
}

// Call the main function and catch any errors
main().catch((error) => {
    console.error(error)
    process.exitCode = 1
})
```


### and following that, to make the json file more complete, you can run the subsequent code to write another json file with enriched date of token0 and token1


```javascript
const { ethers } = require("hardhat");
const fs = require("fs");

async function main() {
  // 1. Read the list of pair addresses from pairs.json
  const pairsData = JSON.parse(fs.readFileSync("pairs.json", "utf8"));
  console.log(`Loaded ${pairsData.length} pair addresses from pairs.json`);

  // Create a provider from Hardhat
  const provider = ethers.provider;

  // Minimal Pair ABI (only token0 & token1 needed)
  const pairABI = [
    "function token0() external view returns (address)",
    "function token1() external view returns (address)"
  ];

  // We'll accumulate the new data here
  let pairsWithTokens = [];

  // 2. Loop over each pair address and fetch token0 & token1
  for (const pairAddress of pairsData) {
    const pairContract = new ethers.Contract(pairAddress, pairABI, provider);

    try {
      const token0 = await pairContract.token0();
      const token1 = await pairContract.token1();

      console.log(`Pair ${pairAddress} => token0: ${token0}, token1: ${token1}`);

      // Store the enriched data
      pairsWithTokens.push({
        pairAddress,
        token0,
        token1
      });
    } catch (err) {
      // If there's an error reading token0/token1 (e.g. if it's not actually a Uniswap pair)
      console.error(`Error fetching token0/token1 for pair ${pairAddress}: ${err.message}`);
    }
  }

  // 3. Write the enriched data to a new JSON file
  fs.writeFileSync("pairsWithTokens.json", JSON.stringify(pairsWithTokens, null, 2));
  console.log(`Successfully wrote ${pairsWithTokens.length} entries to pairsWithTokens.json`);
}

main().catch((error) => {
  console.error("Error in main:", error);
  process.exit(1);
});
```


## The bot
You need to provide a Blocknative API key, an RPC URL and the private key to your EOA
1. Loads the pairsToTokens.json file that we created above.
2. Uses the blocknative Golang SDK (https://github.com/marshabl/blocknative-go-sdk) to create a websocket feed to watch a number of different DEX and Aggregator addresses
3. Uses your RPC to watch for sync events on all token pairs in your graph, so that you get updated reserve data to be used in your profit calculations
4. Subscribes to block events to get the latest baseFee (although this could be done if Blocknative provides the baseFee in its payloads)
5. In mempool.go, I am leveraging Blocknative's sim platform and the net balance changes. For each transaction, I check net balance change addresses and if one of them is a LP in my graph, then I check for profit.
6. For each transaction I check for profit in utils.go. I am using the formula in Daniel's great post here (https://www.ddmckinnon.com/2022/11/27/all-is-fair-in-arb-and-mev-on-avalanche-c-chain/). So given a transaction that impacts a LP, I get all of the cycles from pairsToToken.json, and for each cycle I get the optimal amount in from the formula in the blog post. With this optimal amount in I check the profit of each cycle and I choose the highest one. If that is greater than baseFee * approx 150K gas, then I know this is a potential opportunity worth pursuing further
7. If there is a potential opportunity, I create the bundle and simulate it. If the simulation succeeds, I grab the actual gas used and calculate the actual gas price I want to spend based on a chosen margin. And then I fire it off to the builders through sendBundle
   
