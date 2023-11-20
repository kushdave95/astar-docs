---
sidebar_position: 2
---

# Simple Cross-Chain Interaction

LayerZero (LZ) provides Astar Network with cross-chain communication capabilities, enabling asset and data transfer across different blockchains. By leveraging LayerZero's capabilities, developers on Astar can design versatile and efficient omnichain solutions.

## Overview

For a developer to send a customized cross-chain transaction from Astar (source) to another network (destination: ethereum will be used for this tutorial) a few things need to happen: 

1. **Write**: A LayerZero Omnichain smart contract must be written to estimate cross-chain fees for transactions, as well as compose send and receive logic.
2. **Deploy**: This smart contract must be deployed on both the source (Astar) and destination (Ethereum) chains, specifying the LZ endpoint address associated with each chain in the contract's constructor argument.
    - LZ endpoints contain the functionality needed for your smart contracts to communicate with an UltraLightNode (ULN), a low-cost mechanism designed to validate and relay cross-chain messages.
3. **Set Trusted Remotes**: A trusted remote (contract address) must be configured for both the source and destination chain contracts after deployment, enabling them to authenticate that incoming messages are from the appropriate source contract.
4. **Estimate Fees and Send Transactions**: Estimate how much gas to send using `estimateFees` and send a simple message to your destination chain

Let's go through these steps one-by-one:

### 1. Write

- Three functions are crucial for an omnichain user application to successfully send and receive cross-chain messages.
    1. `estimateFees`
    
    ```sol
     // @return nativeFee The estimated fee required denominated in the native chain's gas token.
    function estimateFees(uint16 dstChainId, bytes calldata adapterParams, string memory _message) public view returns (uint nativeFee, uint zroFee) {
        
        //Input the message you plan to send.
        bytes memory payload = abi.encode(_message);
        
        // Call the estimateFees function on the lzEndpoint contract.
        // This function estimates the fees required on the source chain, the destination chain, and by the LayerZero protocol.
        return lzEndpoint.estimateFees(dstChainId, address(this), payload, false, adapterParams);
    }
    ```

    2. `send`

    ```sol
     // This function is called to send the data string to the destination.
    // It's payable, so that we can use our native gas token to pay for gas fees.
    function send(uint16 dstChainId, bytes calldata adapterParams, string memory _message) public payable {
        
        // The message is encoded as bytes and stored in the "payload" variable.
        bytes memory payload = abi.encode(_message);
        
        // The data is sent using the parent contract's _lzSend function.
        _lzSend(dstChainId, payload, payable(msg.sender), address(0x0), adapterParams, msg.value);
    }
    ```

    3. `_lzReceive`

    ```sol
     // This function is called when data is received. It overrides the equivalent function in the parent contract.
    function _lzReceive(uint16, bytes memory, uint64, bytes memory _payload) internal override {
       
       // The LayerZero _payload (message) is decoded as a string and stored in the "data" variable.
       data = abi.decode(_payload, (string));
    }
    ```

### 2. Deploy

- The `estimateFees` and `send` functions leverage inherited methods from LzApp to engage with the LZ endpoint. This endpoint must be defined as a parameter in the contract's constructor, as shown below:
    ```sol
    import "@layerzerolabs/solidity-examples/blob/main/contracts/lzApp/LzApp.sol";

    contract OmnichainApp is LzApp {

        constructor(address _lzEndpoint) LzApp(_lzEndpoint) {
        
        }

    }
    ```

Once your constructor is properly defined, you can deploy your contracts by: 
1. Running the following script on both chains:

    ```javascript
    // deploy.js
    // Using Ethers v6

    const hre = require("hardhat");

    async function main() {

        const YourOApp = await hre.ethers.getContractFactory("OmnichainApp");

        const endpointAddress = "0x00000000000000000000000000000"; // Replace with the given chain's endpoint address
        
        // Deploy the contract with the specified constructor argument
        
        const omniApplication = await YourOApp.deploy(endpointAddress);

        // Wait for the deployment to finish
        await omniApplication.waitForDeployment(); 
        console.log("Your Omnichain Application deployed to:", await omniApplication.getAddress());
   
    }

    // We recommend this pattern to be able to use async/await everywhere
    // and properly handle errors.
    main().catch((error) => {
        console.error(error);
        process.exitCode = 1;
    });
    ```

2. Using Hardhat CLI:

    ```bash
    npx hardhat run scripts/deploy.js --network astarTestnet
    ```

    ```bash
    npx hardhat run scripts/deploy.js --network sepolia
    ```

### 3. Set Trusted Remotes

In the LayerZero framework, when a message is sent from the source contract on one blockchain, the destination contract on another blockchain must be pre-configured to recognize and accept messages only from a specific, trusted source contract address. This setup ensures that when the destination contract receives a message, it can authenticate that it's indeed from the correct source. Trusted remotes can be set by calling the following function:

    ```sol
    function setTrustedRemoteAddress(uint16 _remoteChainId, bytes calldata _remoteAddress) external onlyOwner {
        trustedRemoteLookup[_remoteChainId] = abi.encodePacked(_remoteAddress, address(this));
        emit SetTrustedRemoteAddress(_remoteChainId, _remoteAddress);
    }
    ```

Once your trusted remotes are set your contracts are ready to send and receive cross-chain transactions!

### 4. Estimate Fees and Send Transactions

Every cross-chain transaction has a different fee quote associated with it based on 3 inputs:

1. `dstChainId`: The destination chain
2. `adapterParams`: A bytes array that contains custom instructions for how a LZ Relayer should transmit the transaction. Custom instructions include: 
    1. The upper limit on how much destination gas to spend
    2. Instructing the relayer to airdrop native currency to a specified wallet address
3. `_message`: This is the message you intend to send to your destination chain and contract

These inputs are passed into the `estimateFees` function which returns a quote. The quote is then passed as the `msg.value` of your `send` transaction. This cross-chain transaction flow can be packaged into a script and ran via Hardhat CLI like so:

***Javascript***

    ```javascript
    // sendTx.js
    // Using Ethers v6

    const hre = require("hardhat");
    async function testMessaging() {

        const ethers=hre.ethers

        // Define your contract

        const yourOAppFactory = await ethers.getContractFactory("OmnichainApp");
        const omnichainApp = yourOAppFactory.attach("0x00000000000"); // Replace with your source chain OApp's address

        // Define your input parameters

        const abicoder = new ethers.AbiCoder()
        const dstChainId = '101';
        const adapterParams = ethers.utils.solidityPack(['uint16','uint256'], [1, 200000]) // Passing 200000 as a default for gas

        // Set the content of your cross-chain message, that will be decompiled and stored on your destination contract via lzReceive

        const message = abicoder.encode(["string"], ["Transaction Passed!"]);

        // Run estimateFees to get your quote

        const [nativeFee, zroFee] = await omnichainApp.estimateFees(dstChainId, false, adapterParams);
        console.log(`Estimated Fees: Native - ${nativeFee}, ZRO - ${zroFee}`);

        // Send your message passing your nativeFee as the msg.value

        const tx = await omnichainApp.send(dstChainId, adapterParams, message, { value: nativeFee });
        await tx.wait();

        console.log('Message sent successfully', tx);
    }
    
    testMessaging()
    .then(() => console.log('Messaging test completed'))
    .catch(error => console.error('Error in messaging test:', error));
    ```

***Hardhat CLI***

    ```bash
    npx hardhat run scripts/sendTx.js --network testnet-astar
    ```

### Examine the transaction on LayerZero Scan

Finally, let's see what's happening in our transaction. 
Take your transaction hash (printed in your console logs from the step before) and paste it into: https://testnet.layerzeroscan.com/.

You should see Status: Delivered, confirming your message has been delivered to its destination using LayerZero.

Congrats, you just sent your first Omnichain message! 🥳