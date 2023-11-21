---
sidebar_position: 1
---

# Setting up your Environment

## Introduction

This guide will walk you through setting up your environment to deploy LayerZero contracts on Astar Testnet and Ethereum Sepolia Testnet using Hardhat.

## Prerequisites

Before you begin, ensure you have the following installed:

- Node.js (v12.0 or higher)
- npm (Node Package Manager)
- A code editor (like Visual Studio Code)
- Knowledge of Hardhat and Solidity

### Step 1: Setting Up Hardhat

Hardhat is an Ethereum development environment. Follow these steps to set it up:

- Create a New Node.js Project:
    - Open your terminal.
    - Create a new directory for your project and navigate into it.
    - Run `npm init -y` to initiate a new Node.js project.
- Install Hardhat:
    - In your project directory, run `npm install --save-dev hardhat`.
- Create a Hardhat Project:
    - Run `npx hardhat` in your project directory.
    - Select “Create an empty hardhat.config.js” to create a Hardhat configuration file.

### Step 2: Installing Dependencies

Install the following npm packages:

- @nomiclabs/hardhat-ethers: Plugin for integration with ethers.js
- ethers: Ethereum wallet implementation and utilities
- dotenv: Loads environment variables from a .env file
- @layerzerolabs/solidity-examples: Layerzero NPM package for boilerplate app-level contracts

Run this command to install them:

```bash
npm install --save-dev @nomiclabs/hardhat-ethers ethers dotenv @layerzerolabs/solidity-examples
```

### Step 3: Configuring Hardhat

Edit the hardhat.config.js File:

- Import @nomiclabs/hardhat-ethers.
- Set up network configurations for Astar Testnet and Ethereum Sepolia Testnet.

Example configuration:

```javascript
require("@nomiclabs/hardhat-ethers");
require('dotenv').config();

module.exports = {
  solidity: "0.8.4",
  networks: {
    sepolia: {
      url: "process.env.SEPOLIA_RPC_URL",
      accounts: ["process.env.PRIVATE_KEY"]
    },
    astarTestnet: {
      url: "process.env.ASTAR_TESTNET_RPC_URL",
      accounts: ["process.env.PRIVATE_KEY"]
    }
  }
};
```

### Step 4: Setting Up Environment Variables

- Create a .env file in your project root.
- Add your private key and RPC URLs for both networks.
- Make sure to add .env to your .gitignore file to keep your private keys safe.

Example `.env`:

```bash
PRIVATE_KEY=your_private_key_here
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/your_project_id
ASTAR_TESTNET_RPC_URL=https://astar_testnet_rpc_url
```