---
sidebar_position: 3
---

# Omnichain Fungible Token (OFT) Implementation

Omnichain Fungible Tokens (OFTs) are a type of token built on top of Layerzero that can exist and be transferred across multiple blockchain networks while maintaining a single, unified total supply. This is a groundbreaking feature because, traditionally, tokens are native to a single blockchain. For example, an ERC-20 token exists solely on the Ethereum network. However, OFTs break this limitation by allowing the same token to move seamlessly between different blockchains, such as Ethereum, Binance Smart Chain, and Astar.

## Overview

Transferring OFT tokens across chains involves similar steps to the simple cross-chain interaction process. However, there are specific nuances related to handling fungible tokens and their associated transactions. Here's a high-level example of how a user might send OFT tokens from Astar to Ethereum:

1. The user calls a function in the OFT contract on Astar, specifying the token amount to send and the Ethereum destination address.
2. The contract debits the tokens from the user's balance on Astar and sends a cross-chain message to the Ethereum contract.
3. Upon receiving the message, the Ethereum contract credits the tokens to the specified address.

## Deployment

The OFT contract needs to be deployed on both the source (e.g., Astar Network) and the destination (e.g., Ethereum) chains. This process is similar to the deployment process for our simple cross-chain transaction sample. However, instead of building a new contract, we will deploy the boilerplate OFT contract provided by Layerzero's solidity examples, which provides all of the base functionality necessary to deploy an OFT on multiple chains.

To initiate, set up a deployment script in your `/scripts` folder, and add constructor arguments based on the OFTV2 constructor function below:



***Solidity***

    ```sol

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _sharedDecimals,
        address _lzEndpoint
    ) ERC20(_name, _symbol) BaseOFTV2(_sharedDecimals, _lzEndpoint) {
        uint8 decimals = decimals();
        require(_sharedDecimals <= decimals, "OFT: sharedDecimals must be <= decimals");
        ld2sdRate = 10**(decimals - _sharedDecimals);
    }
    
    ```

***Javascript***

    ```javascript
    // deploy.js
    // Using Ethers v6

    const hre = require("hardhat");

    async function main() {

        const YourOFT = await hre.ethers.getContractFactory("OFTV2");

        const endpointAddress = "0x00000000000000000000000000000"; // Replace with the given chain's endpoint address
        
        // Deploy the contract with the specified constructor arguments
        
        const oft = await YourOFT.deploy(
            
            "MyFirstOFT", // OFT name
            "MYOFT", // OFT symbol
            8, // shared decimals for your OFT **See INFO for full explanation of this**
            "0x000000000000000000000000000" // chain endpoint address

        );

        // Wait for the deployment to finish
        await oft.waitForDeployment(); 
        console.log("Your Omnichain Application deployed to:", await oft.getAddress());
   
    }

    // We recommend this pattern to be able to use async/await everywhere
    // and properly handle errors.
    main().catch((error) => {
        console.error(error);
        process.exitCode = 1;
    });
    ```

***Hardhat CLI***

    ```bash
    npx hardhat compile
    ```

    ```bash
    npx hardhat run scripts/deploy.js --network astarTestnet
    ```

    ```bash
    npx hardhat run scripts/deploy.js --network sepolia
    ```

:::info
<b>What should I set as shared decimals?</b>

Setting Decimals on Non-EVM Chains: If your token is deployed on both non-EVM and EVM chains, you should set the decimal count to the lowest value used on any chain. For example, if it's 6 on Aptos (non-EVM) and 18 on Ethereum (EVM), you should use 6 as the standard across all chains. This ensures consistency in token divisibility across different platforms.

Setting Decimals on EVM Chains Only: If your token is only on EVM chains and all deployments use more than 8 decimals, you should standardize it to 8 decimals across these chains.
:::

## Setting Trusted Remotes

In the OFT framework, setting up trusted remotes is crucial for secure cross-chain transactions, mirroring the workflow in our simple cross-chain transaction model. This is achieved through the `setTrustedRemoteAddress` function, where you specify the remote chain ID and the address of the destination contract as a bytes array on both chains. This configuration ensures that your contract only accepts messages from verified sources.

 ```sol
    function setTrustedRemoteAddress(uint16 _remoteChainId, bytes calldata _remoteAddress) external onlyOwner {
        trustedRemoteLookup[_remoteChainId] = abi.encodePacked(_remoteAddress, address(this));
        emit SetTrustedRemoteAddress(_remoteChainId, _remoteAddress);
    }
```

## Conducting an OFT Token Transfer

To initiate a transfer:

**From the Source Chain**: 

- Define your adapterParams according to your specifications. For more info on adapterParams click [here](https://layerzero.gitbook.io/docs/evm-guides/advanced/relayer-adapter-parameters).
- Call the `estimateSendFee` function, which will return a `nativeFee` gas quote for your token transfer.
- Call the `sendFrom` function in your smart contract that triggers `_debitFrom`, which handles the deduction of tokens from the user's balance and initiates the cross-chain request using LayerZero's messaging system.
    - **Set the native fee as the msg.value for your `sendFrom` function.



**On the Destination Chain**: When the message is received, `_creditTo` is invoked on the destination chain, crediting the specified amount of tokens to the target address.

Let's compile this token transfer process into a script that sends 10 MYOFT tokens from the Astar Network Testnet to Sepolia with the necessary input parameters for `estimateSendFee` and `sendFrom` properly labeled:

***Javascript***

```javascript

// crossChainOFTTransfer.js
// Using Ethers v6

const hre = require("hardhat");
async function performOFTTransfer() {

    const ethers = hre.ethers;

    // Connect to your OFT contract

    const oftContractFactory = await ethers.getContractFactory("YourOFTContract");
    const oftContract = oftContractFactory.attach("0xYourOFTContractAddress"); // Replace with your OFT contract's address

    // Define the destination chain ID and recipient address

    const dstChainId = '101'; // Destination chain ID (example: 101)
    const toAddress = ethers.formatBytes32String("0xRecipientAddress"); // Convert recipient's address to bytes32 format

    // Specify the amount of tokens to be transferred

    const transferAmount = ethers.parseUnits("10", 8); // Transferring 10 tokens with 8 decimals (shared decimals amount)

    // Prepare adapter parameters for LayerZero

    const adapterParams = ethers.solidityPacked(['uint16', 'uint256'], [1, 200000]); // Customizing LayerZero's adapter parameters

    // Estimate the fees for the cross-chain transfer

    const [nativeFee, zroFee] = await oftContract.estimateSendFee(dstChainId, toAddress, transferAmount, false, adapterParams); // false indicates no ZRO payment will be made
    console.log(`Estimated Fees: Native - ${nativeFee}, ZRO - ${zroFee}`);

    // Execute the cross-chain transfer

    const tx = await oftContract.sendFrom(
        "0xYourAddress", // Your address as the sender
        dstChainId, 
        toAddress, 
        transferAmount, 
        { 
            refundAddress: "0xYourRefundAddress", // Address for refunded gas to be sent
            zroPaymentAddress: "0xZroPaymentAddress", // Address for ZRO token payment, if used
            adapterParams: adapterParams
        }, 
        { value: nativeFee }
    );
    await tx.wait();

    console.log('Cross-chain OFT transfer completed', tx);
}

performOFTTransfer()
    .then(() => console.log('OFT Transfer completed successfully'))
    .catch(error => console.error('Error in OFT transfer:', error));

```

***Hardhat CLI***

    ```bash
    npx hardhat run scripts/sendTx.js --network testnet-astar
    ```

### Examine the transaction on LayerZero Scan

Finally, let's see what's happening in our transaction. 
Take your transaction hash (printed in your console logs from the step before) and paste it into: https://testnet.layerzeroscan.com/.

You should see Status: Delivered, confirming your message has been delivered to its destination using LayerZero.

Congrats, you just executed your first Omnichain fungible token transfer! 🥳