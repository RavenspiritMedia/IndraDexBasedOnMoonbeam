

# Uniswap V2 on IndraChain

This repository provides the tools necessary to deploy Uniswap V2 on IndraChain
**This repository is for demostration purposes only.**

## Getting started

There are three different repositories.

### Uniswap Contracts IndraChain

A Hardhat setup to deploy all the necessary on  IndraChain chain.

### Uniswap Interface IndraChain

The simple v2 Uniswap interface modified to work with IndraChain node.

### Uniswap SDK IndraChain

Modified version of the Uniswap SDK to add compatibility with IndraChain node.

### Porting Uniswap V2 to IndraChain

Deploying Uniswap V2 Contracts to IndraChain.
A concrete example is better than 1000 words, so this section illustrates the process of deploying all the necessary Uniswap V2 contracts to IndraChain.
To do so, a Github repo was prepared. The “uniswap-contracts-IndraChain” folder includes Hardhat setup, which contains all the necessary contract files and a deployment script for the following contracts (essentials to run Uniswap):-


=> WETH: ether wrapped in an ERC20 token interface (in the interface it will be called wrapped DEV or WDEV)
=> UniswapV2Factory: only input required is an address that can activate a protocol-level fee (input needed but not used in this example)
=> UniswapV2Router02: requires the address of both the WETH and UniswapV2Factory contracts
=> Multicall: aggregates results from multiple contract constant function calls, reducing the number of separate JSON RPC requests that need to be sent. This is required by the Uniswap interface
=> Two ERC20 tokens: these are not required but were included to have some tokens to start to play around with.

Getting started is just as easy as cloning the repo, installing the dependencies, and deploying the contracts with Hardhat. So first, let’s start cloning the repo and installing the dependencies:

In this folder, you will find a “hardhat-config.js” file that contains two pre-configured networks: a Moonbeam standalone node and the Moonbase Alpha TestNet and change network configuration here . Here, make sure you modify `privateKey2` for the private key you want to use to deploy the contracts in the TestNet.

For this example, I’ll be deploying the contracts in a Moonbeam standalone node. You can spin up your own by following this tutorial. With a fresh instance of a standalone node running, we can run our deployment script:

npx hardhat run --network standalone scripts/deploy-factory.js


Note: To ensure that the Uniswap interface included in this repo works with a Moonbeam standalone node, you need to deploy the contracts against a fresh instance using the provided private key. The contracts will be deployed to specific addresses that are already configured in the interface and SDK.


Using only a simple Hardhat + Ethers.js script we’ve deployed all the necessary Uniswap V2 contracts to a Moonbeam standalone node. Note that, at first glance, no modifications were required at a contract level.

However, Uniswap uses an init_code hash that was changed in the UniswapV2Library contract. Remember the create2 opcode? Well the change is related to it. This init_code hash is used to calculate the address of a pool (created by the opcode) by providing only the addresses of the two ERC20 tokens. The address can be calculated with the following formula:

address = keccak256(0xff+factoryAddress+salt+keccak256(UniswapV2Pair.byteCode))


Where the salt is the same as the one shown in the UniswapV2Factory code (related to both token addresses). The init_code hash is the last bit of the equation (keccak256(UniswapV2Pair.bytecode)), that is the keccak256 hash of the bytecode of the UniswapV2Pair contract. If the wrong init_code is provided, the smart contracts (and interface) won’t be able to calculate the correct address of the pool. The ini_code can be obtained in different ways. For example, fetching the bytecode manually in Remix and calculating the keccak 256 hash.

The code that was changed lies inside the pairFor function of the UniswapV2Library contract:

// calculates the CREATE2 address for a pair without making any external calls
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
(address token0, address token1) = sortTokens(tokenA, tokenB);
pair = address(uint(keccak256(abi.encodePacked(
hex'ff',
factory,
keccak256(abi.encodePacked(token0, token1)),
hex'01429e880a7972ebfbba904a5bbe32a816e78273e4b
38ffa6bdeaebce8adba7c' // init code hash
))));
}

Nevertheless, this modification is not related to deploying the contract on Moonbeam rather than Ethereum, but it is a consequence of the create2 opcode.

###  Adapting Uniswap V2 Interface to Support IndraChain


The Uniswap V2 Interface is used as a middleware between the smart contracts and the user. The interface relies on external providers (such as MetaMask) and the ethers.js library to connect and interact with the blockchain, respectively.

Deploying the smart contracts to IndraChain (in this case, a standalone node) was a pretty straightforward task, as it provides a full Ethereum-like environment. Adapting the interface so that it works with our deployment was somewhat more complicated. This is because the interface has two main factors which limit its implementation into a new blockchain: the network chain ID, and the addresses of the deployed contracts. Moreover, the Uniswap SDK (a package the interface depends on) needs to be modified as well for the same reasons (next section).

The modified interface is included in this Github repo, inside the “uniswap-interface-IndraChain” folder. Getting started is quite simple: clone the repo, install the dependencies and run the interface instance.


Note that the interface (and SDK) have some contract addresses hardcoded. If you are using a fresh  node, and deploy the contracts with the Hardhat configuration provided, it should work. However, if you deploy the contracts to the TestNet, you would need to modify the addresses in both the interface and SDK.

The following sections dive into a more detailed breakdown of the actual changes that took place.

### Including the Chain ID

The chain ID was introduced in EIP-155. It originated to prevent replay-attacks between ETH and ETC chains that shared the same network ID. The chain ID argument is included in the transaction signature so that two identical transactions will have different v-r-s signature values. Some chain ID values include:

Network	                                                           Chain ID
Ethereum MainNet	                                                    1
Rinkeby TestNet	                                                        4
Kovan TestNet	                                                        42
Moonbeam Standalone node	                                           1281
Moonbase Alpha	                                                       1287


The Uniswap V2 Interface has baked-in some predefined chain IDs so that, once the interface is connected to a blockchain via a provider, it checks that the protocol supports the network you are connecting to.

Both Moonbeam related Chain IDs were added to the following files (inside the “uniswap-interface-IndraChain folder) :

  ==> ./src/connectors/index.ts: add the corresponding chain IDs inside the “supportedChainIds” array:

  export const injected = new InjectedConnector({
  supportedChainIds: [1281, 1287]
  //old supportedChainIds: [1, 3, 4, 5, 42]
  })

  ==> ./src/components/Header/index.tsx: add the chain IDs and network labels following the corresponding format inside “NETWORK_LABELS”:

  const NETWORK_LABELS: { [chainId in ChainId]: string | null } = {
  [ChainId.MAINNET]: null,
  [ChainId.STANDALONE]: 'Moonbase Standalone',
  [ChainId.MOONBASE]: 'Moonbase Alpha'
  }

 ==> ./src/constants/index.ts: add the chain IDs following the corresponding format inside “WDEV_ONLY: ChainTokenList”:
   
   const WDEV_ONLY: ChainTokenList = {
   [ChainId.MAINNET]: [WDEV[ChainId.MAINNET]],
   [ChainId.STANDALONE]: [WDEV[ChainId.STANDALONE]],
   [ChainId.MOONBASE]: [WDEV[ChainId.MOONBASE]]
   }

==> ./src/state/lists/hooks.ts: add the chain IDs following the corresponding format inside “EMPTY_LIST: TokenAddressMap”:

    const EMPTY_LIST: TokenAddressMap = {
    [ChainId.MAINNET]: {},
    [ChainId.STANDALONE]: {},
    [ChainId.MOONBASE]: {}
  }

  In the previous code snippets, the ChainId.* object is imported from the SDK.

### Adding the New Contract Addresses

As you would expect, the Uniswap interface is configured to work with the addresses of the contracts deployed to Ethereum. Therefore, the new addresses need to be specified.

The goal was for the interface to work with both a fresh Moonbeam standalone node and the Moonbase Alpha TestNet. Therefore, some modifications were made so that the address was dependent on the network’s chain ID to which the provider is connected. The files that were modified (inside the “uniswap-interface-moonbeam” folder) are listed below. Note that each file imports a `moonbase_address.json` file that is located in the `./src` folder, which contains the addresses of the deployment in Moonbase Alpha.


==> ./src/constants/index.ts: add the corresponding router address. The routerv2 variable is imported

    export const ROUTER_ADDRESS: { [key: string]: string } = {
    [ChainId.STANDALONE]: '0x42e2EE7Ba8975c473157634Ac2AF4098190fc741',
    [ChainId.MOONBASE]: routerv2
    }

==> ./src/constants/multicall/index.tsx: add the corresponding multicall address

    const MULTICALL_NETWORKS: { [chainId in ChainId]: string } = {
    [ChainId.MAINNET]: '0xeefBa1e63905eF1D7ACbA5a8513c70307C1cE441',
    [ChainId.STANDALONE]: '0xF8cef78E923919054037a1D03662bBD884fF4edf',
    [ChainId.MOONBASE]: multicall
    }


==> ./src/state/swap/hooks.ts: add the corresponding factory and router addresses inside “BAD_RECIPIENT_ADDRESS”. For testing purposes, this parameter is not required


  const BAD_RECIPIENT_ADDRESSES: string[] = [
  factory, // v2 factory
  routerv2 // v2 router 02
]


Other Changes
Everything related to Uniswap V1 was removed from the files, as no V1 deployment was used in this example.

Other changes also include:
Logo: file in ./src/assets/images/mainlogo.png
Links: file in ./src/components/Menu/index.tsx


<!-- <MenuItem id="link" href="https://moonbeam.network/">
<Home size={14} />
{t('Website')}
</MenuItem>
<MenuItem id="link" href="https://discord.gg/PfpUATX">
<MessageCircle size={14} />
{t('discord')}
</MenuItem>
<MenuItem id="link" href="https://github.com/PureStake/moonbeam">
<Code size={14} />
{t('code')}
</MenuItem> -->

Adapting Uniswap SDK to Support Moonbeam
The SDK contains additional information that is used by the interface as an NPM package. The modified SDK is included in this Github repo, inside the “uniswap-sdk-IndraChain” folder.

The SDK folder provided works out of the box with the Moonbeam standalone contract deployment done via the Hardhat configuration previously described. It also contains the addresses of the contracts deployed to the Moonbase Alpha TestNet. This is all packed and published as an NPM package with the name “moonbeamswap.”

Before building the NPM package, the files listed below need to be modified (inside the “uniswap-sdk-moonbeam” folder). Note that each file imports a `moonbase_address.json` file that is located in the `./src` folder and contains the addresses of the deployment in Moonbase Alpha:

==> ./src/constants.ts: add the corresponding chain IDs inside the “ChainId” enum, change the factory address, and modify the init_code hash

export enum ChainId {
MAINNET = 1,
STANDALONE = 1281,
MOONBASE = 1287
}
...
export const FACTORY_ADDRESS: { [key: string]: string } = {
[ChainId.STANDALONE]: '0x5c4242beB94dE30b922f57241f1D02f36e906915',
[ChainId.MOONBASE]: factory
}
...
export const INIT_CODE_HASH = '0x01429e880a7972ebfbba904a5bbe32a816e78273e4b38ffa6bdeaebce8adba7c'


==> ./src/entities/token.ts: add the WETH (WDEV in this case) token contract address:

export const WDEV = {
[ChainId.MAINNET]: new Token(
ChainId.MAINNET,
'0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2',
18,
'WETH',
'Wrapped Ether'
),
[ChainId.STANDALONE]: new Token(
ChainId.STANDALONE,
'0xC2Bf5F29a4384b1aB0C063e1c666f02121B6084a',
18,
'WDEV',
'Wrapped Dev'
),
[ChainId.MOONBASE]: new Token(ChainId.MOONBASE, WETH, 18, 'WDEV', 'Wrapped Dev')
}


==> ./src/entities/token.ts: add the WETH (WDEV in this case) token contract address:


export const WDEV = {
[ChainId.MAINNET]: new Token(
ChainId.MAINNET,
'0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2',
18,
'WETH',
'Wrapped Ether'
),
[ChainId.STANDALONE]: new Token(
ChainId.STANDALONE,
'0xC2Bf5F29a4384b1aB0C063e1c666f02121B6084a',
18,
'WDEV',
'Wrapped Dev'
),
[ChainId.MOONBASE]: new Token(ChainId.MOONBASE, WETH, 18, 'WDEV', 'Wrapped Dev')
}


Once all the files have been modified, change package name, version and description inside the `package.json` file. When you are ready, log in into your npm account run the publish command:

npm login #enter credentials
npm publish


If you are going to run your custom package in the interface, make sure to add it as a dependency in the `package.json` file of the interface folder.
--

Have fun :)
