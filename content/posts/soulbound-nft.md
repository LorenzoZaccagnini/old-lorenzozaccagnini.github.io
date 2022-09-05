---
title: "Develop a Soulbound NFT using Foundry and Slither"
date: 2022-08-16T00:03:47+02:00
cover: "img/soulbound-nft-cover.jpg"
---

Today we will develop a Soulbound NFT, an NFT that can be only minted and not traded or transferred, it is bounded to the first owner. We'll do it using foundry with hardhat integrated. The Github workflow will test **(foundry solidity and hardhat typescript)** the contracts and [uses Slither to statically analyze the code](https://github.com/crytic/slither), trying to find the most common vulnerabilities.

You can find all the code used in my repository [here](https://github.com/LorenzoZaccagnini/Soulbound-NFT-foundry-slither)

## 1. Install Foundry and create the project

### 1.1 Install [Foundry](https://github.com/gakonst/foundry).

```bash
curl -L https://foundry.paradigm.xyz | sh
```

### 1.2 Create the a new foundry project

```bash
forge init soulbound-nft
```

### 1.3 Install the openzeppelin library

```bash
forge install openzeppelin/openzeppelin-contracts
```

## 2. Write the smart contract

Rename the smart contract in the `src` folder to `NFTToken.sol` or the name you want.

### 2.1 Use a pragma solidity directive to specify the compiler version.

```solidity
pragma solidity ^0.8.13;
```

Use a version above 0.6.0 to avoid errors and previous vulns, like overflow and underflow. I prefer the latest stable version.

### 2.2 Import the openzeppelin libraries.

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
```

### 2.3 Inherit from the `ERC721` and `Ownable` contract.

```solidity
contract NFTToken is ERC721, Ownable {
    ...
}
```

Ownable is a library that allows the contract to be owned by a single address. Later we will implement the `onlyOwner` modifier to allow only the owner to call the functions.

### 2.4. Setup a counter for the token incremental id.

```solidity
    using Counters for Counters.Counter;

    Counters.Counter private _tokenIdCounter;
```

Here we are making a new counter and naming it `_tokenIdCounter`, using the `safemath` library.

### 2.5 Setup the token name and symbol.

```solidity
constructor() ERC721("Soulbound", "SBNFT") {}
```

If you feel confident you can even make a factory to create the token.

### 2.6. Write the `safeMint` function.

```solidity
    function safeMint(address to) public onlyOwner {
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        _safeMint(to, tokenId);
    }
```

This function is similar to the `mint` function in the `ERC721` contract, but it is protected by the `onlyOwner` modifier and uses the `_safeMint` function. An Internal function to safely mint a new token. Reverts if the given token ID already exists. If the target address is a contract, it must implement `onERC721Received`, which is called upon a safe transfer, and return the magic value `bytes4(keccak256...` otherwise, the transfer is reverted. [Source Openzeppelin documentation](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#ERC721-_safeMint-address-uint256-bytes-).

### 2.7 Let's implement the soulbound features

```solidity
modifier oneTransfer(address from) {
    require(
        from == 0x0000000000000000000000000000000000000000,
        "Soulbound nft can't be transferred"
    );
    _;
}

function _beforeTokenTransfer(
    address from,
    address to,
    uint256 tokenId
) internal override oneTransfer(from) {
    super._beforeTokenTransfer(from, to, tokenId);
}

```

The `oneTransfer` modifier is used to prevent the transfer of the token to any other address, the 0x0000000000000000000000000000000000000000 address it's an impossible from address. We implement `oneTransfer` to the `_beforeTokenTransfer` function. The `_beforeTokenTransfer` overrides the `_beforeTokenTransfer` function in the `ERC721` contract, and it is called before the transfer, now we have a soulbound nft.

### 2.8 Recap the entire contract.

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.13;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract NFTToken is ERC721, Ownable {
using Counters for Counters.Counter;

    Counters.Counter private _tokenIdCounter;

    constructor() ERC721("Soulbound", "SBNFT") {}

    modifier oneTransfer(address from) {
        require(
            from == 0x0000000000000000000000000000000000000000,
            "Soulbound nft can't be transferred"
        );
        _;
    }

    function safeMint(address to) public onlyOwner {
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        _safeMint(to, tokenId);
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 tokenId
    ) internal override oneTransfer(from) {
        super._beforeTokenTransfer(from, to, tokenId);
    }

}

```

## 3. Write Foundry tests in solidity

Foundry tests are a way to test the contracts in solidity, they are really fast compared to truffle or hardhat. Foundry is made in Rust, so it's blazing fast. Rename the test contract file in the `test` folder to `NFTToken.t.sol`, or the name you want that respects the naming convention `NAMECONTRACT.t.sol`.

### 3.1 Setup the test environment.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/NFTToken.sol";

contract NFTTokenTest is Test {
    using stdStorage for StdStorage;
    NFTToken private nft;

    function setUp() public {
        nft = new NFTToken();
    }
}
```

I import the `Test` contract, and the `NFTToken` contract. NFTTokenTest is the name of the test contract. We are using the `stdStorage`, [a library that makes manipulating storage easy](https://book.getfoundry.sh/reference/forge-std/std-storage). `setUp()` is a function that is called before each test, and it initializes the contract.

### 3.2 Test Smart contract functions.

Here I'm testing the smart contract functions. The pattern is easy to understand, we test the function correct output and revert if it's not correct with the `vm.expectRevert()` function. The `vm.startPrank()` and `vm.stopPrank()` functions are used to simulate a user that is not the owner. By using solidity to write tests we can test the smart contract functions the closest possible to a real user or external contract.

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/NFTToken.sol";

contract NFTTokenTest is Test {
using stdStorage for StdStorage;
NFTToken private nft;

    function setUp() public {
        nft = new NFTToken();
    }

    function testDeployment() public {
        assertEq(nft.name(), "Soulbound");
        assertEq(nft.symbol(), "SBNFT");
    }

    function testOwner() public {
        assertEq(nft.owner(), address(this));
    }

    function testMintFailByNotOwnerUser() public {
        vm.expectRevert("Ownable: caller is not the owner");
        vm.startPrank(address(2));
        nft.safeMint(address(2));
        vm.stopPrank();
    }

    function testMint() public {
        nft.safeMint(address(1));
        assertEq(nft.balanceOf(address(1)), 1);
        assertEq(nft.ownerOf(0), address(1));
    }

    function testTransferFail() public {
        nft.safeMint(address(2));
        vm.expectRevert("Soulbound nft can't be transferred");
        vm.startPrank(address(2));
        nft.safeTransferFrom(address(2), address(3), 0);
        vm.stopPrank();
    }

    function testOwnerTransfer() public {
        assertEq(nft.owner(), address(this));
        nft.transferOwnership(0x1111111111111111111111111111111111111111);
        assertEq(nft.owner(), 0x1111111111111111111111111111111111111111);
    }

    function testOwnerTransferFail() public {
        assertEq(nft.owner(), address(this));
        vm.expectRevert("Ownable: caller is not the owner");
        vm.startPrank(address(2));
        nft.transferOwnership(0x1111111111111111111111111111111111111111);
        vm.stopPrank();
    }

}

```

### 3.3 Test contract using Forge

```bash
forge test
```

All test cases should pass.

## 4. Integrating Foundry with Hardhat

Hardhat by default expects libraries to be installed in `node_modules`, the default folder for all NodeJS dependencies. Foundry expects them to be in `lib`. Of course we can configure Foundry but not easily to the directory structure of `node_modules`. [Documentation](https://book.getfoundry.sh/config/hardhat) has more information.

### 4.1 Install hardhat

```bash
yarn init
yarn add hardhat hardhat-preprocessor
npx hardhat
forge remappings > remappings.txt
```

You will need to re-run forge remappings everytime you modify libraries in Foundry. Now your `remappings.txt` should look like this:

```bash
ds-test/=lib/solmate/lib/ds-test/src/
forge-std/=lib/forge-std/src/
openzeppelin-contracts/contracts/=lib/openzeppelin-contracts/contracts/
solmate/=lib/solmate/src/
```

### 4.2 Configure hardhat

Edit `hardhat.config.ts` to look like this:

```typescript
import fs from "fs";
import "@nomicfoundation/hardhat-chai-matchers";
import "@typechain/hardhat";
import "hardhat-preprocessor";
import { HardhatUserConfig, task } from "hardhat/config";
function getRemappings() {
  return fs
    .readFileSync("remappings.txt", "utf8")
    .split("\n")
    .filter(Boolean)
    .map((line) => line.trim().split("="));
}

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.13",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
  paths: {
    sources: "./src", // Use ./src rather than ./contracts as Hardhat expects
    cache: "./cache_hardhat", // Use a different cache for Hardhat than Foundry
  },
  // This fully resolves paths for imports in the ./lib directory for Hardhat
  preprocess: {
    eachLine: (hre) => ({
      transform: (line: string) => {
        if (line.match(/^\s*import /i)) {
          getRemappings().forEach(([find, replace]) => {
            if (line.match(find)) {
              line = line.replace(find, replace);
            }
          });
        }
        return line;
      },
    }),
  },
};

export default config;
```

## 5. Write Hardhat tests in typescript

### 5.1 Create test contract

Create the contract in the `test` folder using this name convention `nfttoken.test.ts`.

### 5.2 Write test hardhat file

Here we do the same as the Foundry tests, but we use the hardhat `expect` and the `to.be.revertedWith` function. If the revertedWith is not recognized install the `"@nomicfoundation/hardhat-chai-matchers` package. More info [here](https://hardhat.org/hardhat-chai-matchers/docs/overview) about this.

```typescript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import { NFTToken } from "../typechain-types";

describe("NFTToken", function () {
  let owner: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let nft: NFTToken;

  beforeEach(async function () {
    const Nft = await ethers.getContractFactory("NFTToken");
    nft = await Nft.deploy();
    await nft.deployed();

    [owner, addr1, addr2] = await ethers.getSigners();
  });

  it("Should return name and symbol", async function () {
    expect(await nft.name()).to.equal("Soulbound");
    expect(await nft.symbol()).to.equal("SBNFT");
  });

  it("Should set the first account as the owner", async () => {
    expect(await nft.owner()).to.equal(owner.address);
  });

  it("Should NOT mint a token as NOT the Owner", async () => {
    await expect(nft.connect(addr1).safeMint(addr1.address)).to.be.revertedWith(
      "Ownable: caller is not the owner"
    );
  });

  it("Should mint a token as the Owner", async () => {
    await nft.safeMint(addr1.address);
    expect(await nft.ownerOf(0)).to.equal(addr1.address);
    expect(await nft.balanceOf(addr1.address)).to.equal(1);
  });

  it("Should not be able to transfer soulbound token", async () => {
    await nft.safeMint(owner.address);

    //OVERLOADED TRANSFER function
    await expect(
      nft["safeTransferFrom(address,address,uint256)"](
        owner.address,
        addr2.address,
        0
      )
    ).to.be.revertedWith("Soulbound nft can't be transferred");
  });

  it("Should NOT transfer the contract NON owner to the new owner", async () => {
    await expect(
      nft.connect(addr1).transferOwnership(addr2.address)
    ).to.be.revertedWith("Ownable: caller is not the owner");
  });

  it("Should transfer the contract owner to the new owner", async () => {
    await nft.transferOwnership(addr2.address);
    expect(await nft.owner()).to.equal(addr2.address);
  });
});
```

Different addresses are simulated by using the `connect` function. The `beforeEach` function is used to set up the contract and the `it` function is used to test the contract.

### 5.3 Hardhat VS Forge VS Truffle

I prefer Forge, is faster and closer to the real environment. Hardhat right now is optional, but I would recommend using it as an alternative. Truffle is now obsolete and will be less and less used in the future.

## 6. Create a Github workflow with tests and slither audit

### 6.1 Create a Github workflow

Create a file named `.github/workflows/audit.yml` and put this content:

```yaml
name: Audit

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly

      - name: Run Forge build
        run: |
          forge --version
          forge build --sizes
        id: build

      - name: Run Forge tests
        run: |
          forge test -vvv
        id: forge-test
```

This will install Foundry and run the Forge build and tests.
{{< image src="/img/foundry_tests.jpg" alt="foundry tests" position="center" style="border-radius: 8px;" >}}

### 6.2 Optional Hardat tests

If you want to use hardhat add a step that uses `yarn` to instal the packages and run the command `npx hardhat test`, this will run the tests with Hardhat.

```yaml
- name: Setup NodeJS 14
  uses: actions/setup-node@v2
  with:
    node-version: "14"
- name: Show NodeJS version
  run: npm --version

- name: Install Dependencies
  run: npm install

- name: Run hardhat Test
  run: npx hardhat compile; npx hardhat test
```

Should look like this:
{{< image src="/img/hardhat_tests.jpg" alt="Passed tests" position="center" style="border-radius: 8px;" >}}

### 6.3 Add slither audit

Slither is a Solidity static analysis framework written in Python 3. It runs a suite of vulnerability detectors, prints visual information about contract details, and provides an API to easily write custom analyses. Slither enables developers to find vulnerabilities, enhance their code comprehension, and quickly prototype custom analyses. [Source](https://github.com/crytic/slither)

We can even add slither manually, but I prefer to use `slither-action` command. It's a great tool because will push slither's alerts to the Security tab of the Github project, easing the triaging of findings and improving the continious integration flow.

```yaml
- name: slither-action
  uses: crytic/slither-action@v0.1.1
  continue-on-error: true
  id: slither
  with:
    sarif: results.sarif

- name: Upload SARIF file
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: ${{ steps.slither.outputs.sarif }}
```

If everything is correct you will this result in the Security tab of the Github project:
{{< image src="/img/slither_results.jpg" alt="slither results" position="center" style="border-radius: 8px;" >}}

### 6.4 Recap

They yaml file is used to create a Github workflow that will run the tests and slither audit. Should look like this:

```yaml
name: Hardhat Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly

      - name: Run Forge build
        run: |
          forge --version
          forge build --sizes
        id: build

      - name: Run Forge tests
        run: |
          forge test -vvv
        id: forge-test

      - name: Setup NodeJS 14
        uses: actions/setup-node@v2
        with:
          node-version: "14"
      - name: Show NodeJS version
        run: npm --version

      - name: Install Dependencies
        run: yarn

      - name: Run hardhat Test
        run: npx hardhat compile; npx hardhat test

      - name: slither-action
        uses: crytic/slither-action@v0.1.1
        continue-on-error: true
        id: slither
        with:
          sarif: results.sarif

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: ${{ steps.slither.outputs.sarif }}
```
