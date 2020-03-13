# Solidity I

## Index

- [x] Overview

- [x] Solidity basics (Bingo)

- [x] Testing

- [x] Deployment Process

* [x] Tasks

## Overview

In this tutorial we are going to build simple DAPP for Tron network.

### What is Tron?

Tron is blockchain with **DPoS**. The system currency is **TRX** (6 decimals). The block is produced in 3 seconds.

Users have addresses that may have 2 forms: hex(it always starts with prefix 41) and base58.

The main resources for network are **Bandwidth** and **Energy**. Bandwidth is used for transactions. Users receive some amount of it everyday which allows them to send few transactions(that don't consume energy) per day for free. Transaction sender doesn't pay any fee. But if it creates or triggers smart contract it may consume a large amount of Energy. All resources can be bought or received by freezing TRX.

### Development

Generally, development process consists of the following stages:

1. smart contract implementation;
2. deployment (migration) to network;
3. frontend logic implementation.

Let's gaze into tools used for it.

**Solidity** is the object-oriented language for building smart contracts. It is used for other platforms like **Ethereum** as well but there are some petty differences in their versions.

**Tronbox** is powerful tool that allows to compile, debug and migrate(broadcast) smart contracts to blockchain.

**Tronweb** is a library that wraps node api functions. It includes, but is not limited to, triggering smart contract and signing transactions on Frontend.

## Solidity basics (Bingo)

### Goal explained

Due to this lesson we will implement simplified version of **Lines Bingo**. General rules are following:

- one game - one card;
- each card is 5\*5 field with numbers on it and free center number (so only 24 numbers are present on the card);
- all numbers are fom 1 to 75;
- there are two stages: buying and revealing;
- during buying stage user make his bet and receive card;
- during revealing, the 30 balls are chosen randomly;
- if some of the numbers match those on the card they are marked;
- number of lines, culumns and diagonals fullly marked are calculated;
- payout equal numberOfMarked \* payoutConst.

### Get started

The easiest way to work on the Solidity code is to use online IDE, especially [Remix](https://remix.ethereum.org/).

Open Remix and chose Solidity Environment. On the left panel, change compiler version to 0.4.24. **Tron** uses it is own compiler compatible with the Tron TVM which is more like this version.

Move to **File explorers** (icon in the top left corner) and create new file **SafeMath.sol** and **Bingo.sol**. Insert:

_SafeMath.sol_

```
pragma solidity >=0.4.22 <0.6.0;

library SafeMath {
    function add(uint a, uint b) internal pure returns (uint c) {
        c = a + b;
        require(c >= a, "Overflow.");
    }

    function sub(uint a, uint b) internal pure returns (uint c) {
        require(b <= a, "Overflow.");
        c = a - b;
    }

    function mul(uint a, uint b) internal pure returns (uint c) {
        c = a * b;
        require(a == 0 || c / a == b, "Overflow.");
    }

    function div(uint a, uint b) internal pure returns (uint c) {
        require(b != 0, "Division by zero.");
        c = a / b;
    }

    function mod(uint a, uint b) internal pure returns (uint c) {
        require(b != 0, "Division by zero.");
        c = a % b;
    }
}
```

_Bingo.sol_

```
pragma solidity >=0.4.22 <0.6.0; // compile version, required line

import "./SafeMath.sol"; // import all from the file

contract Bingo {
    using SafeMath for uint; // way to make library functions visible and callable for uint
}
```

That is a draft of future game contract.

### Main entities

**contract** is something like a class. It can inherit another contracts (for instance `contract A is B, C {...}`), have some functions or values with different visibilities.

Solidity math isn't really smart. It feels pretty fine if you divide by zero or get overflow after subtraction. Thus developers commonly use this piece of code as wrapper for arithmetic operators as it declared in **SafeMath.sol** and we will follow this practice too.

**library** is something similar to contract, but their functions are executed in the context of the contracts. We will discuss it latter. For now, it is important to understand how functions are declared.

### Functions

Look at the last function in SafeMath library (`mod`). It is clear, that `a` and `b` are argument and `c` is return value. It is obligatory to specify argument names but return value names can be omitted. Lets rewrite the code:

```
    function mod(uint a, uint b) internal pure returns (uint) {
        require(b != 0, "Division by zero.");
        c = a % b;
        return c;
    }
```

Function can return few values. For instance:

```
    function mod2(uint a, uint b, uint c) internal pure returns (uint, uint) {
        require(b != 0, "Division by zero.");
        return(a % b, a % c);
    }
```

`internal` and `public` are visibilities of function. There are 4 types of visibility:

- `external`:

  External functions are part of the contract interface and can be called ONLY from other contracts and via transactions.

- `internal`:

  Those functions and state variables can only be accessed within the current contract or contracts deriving from it. Similar to `protected` in some languages.

- `public`:

  Public functions are part of the contract interface and can be either called internally or via messages. It includes visibility of `external` and `internal`. Similar to `public` in some languages.

- `private`:

  Private functions and state variables are only visible for the contract they are defined in and not in derived contracts. Similar to `private` in some languages.

`pure` is a function state:

- `view`:

  Promises not to modify the state. It can't change variables, transfer money, call any function which is isn't `view` or `pure`.

- `pure`:

  Promises not to modify or read the state. It is more strict than `view`. It isn't allowed to read any state or global variables.

- payable:

  Allows to send TRX during calling the function.

By default, if function state isn't marked all operations are permitted but payment can't be received.

### State layout

The difference between contracts and libraries is that contracts have state. Similar to many languages there are local variables which values are reset after calling some functions and state variables that are kept between contract calls.

State variables can be `private` ar `public` just like functtions.

Let's add some state to `Bingo` contract.

```
pragma solidity >=0.4.22 <0.6.0;

import "./SafeMath.sol";

contract Bingo {
    using SafeMath for uint;

    enum Stage { // are used in order prevent double betting or revealing without betting
        BETTING,
        REVEALING
    }

    struct Player {  // structs are defined just like in C
        Stage stage;
        uint block;
        uint bet;
        uint[] card;
        uint[] generatedNumders;
        mapping(uint => uint) cardsCheker;
    }

    uint public minBet = 100000; // min betting amount in wei/sun
    uint public maxBet = 1000000000; // max betting amount in wei/sun
    uint public payoutPerCombination = 2; // payout multiplier
    mapping (address => Bet) public players; // storage for betting details
}

```

All our state consist of 4 fileds (`minBet`, `maxBet`, `payoutPerCombination`, `players`).

`uint` declararations are quite understandable meanwhile `mapping` may need some explanation. `mapping`(key-value pair) players. Anybody can get Player structure by address. If some required address isn't in the map `Player` structure filds are equal to zeros.

### Structs & Enums

We declare enum `Stage`. As soon as we decide to declare variable of this type without definition it will receive first value (`BETTING`) by default. Any enum type is accesible by dot notation. For instance, `Stage.REVEALING`. Or with type cast: Stage(1).

We also declare struct `Player`. In order to define some variable of this type in the future, the constructor is used. It epects all structure fields as parameters except mapping. For example, `Player(Stage.REVEALING, 14347263743, 100000, new uint[](25), new uint[](30))`. Structure fields are accessible by dot. For example, `player.bet`.

### Random generation

Blockchain opperations should be fully deterministic so there are to litle space for randomness. The single source for "randomness" can be blockhash but only hash for previous blocks are accessible in transaction and that's why it's known offchain before calling the contract. To solve this problem we will use two transactions. First one(`bet`, `payable` function) will receive bet and remind the block number of current block(with unknown hash) and the second(`reveal`) will reveal results.

Let's declare appropriate functions:

```
contract Bingo {
	...

	  function bet() public payable {

    }

    function reveal() public {

    }
}

```

### Global Variables

We will use `msg.value` and `msg.sender` that are not defined in the scope. There are some global variables and functions that provide information about blockchain, current block or transaction and give tools for math computation. Read more in the [article](https://solidity.readthedocs.io/en/v0.5.3/units-and-global-variables.html?highlight=global).

### Check requirements

There is three ways to check requirements in Solidity:

- `require(bool condition[, string errorMessage])`
- `assert(bool condition[, string errorMessage])`
- `if (bool condition) { revert([string errorMessage]); }`

It allows to check if some requirements are met and throws error otherwise. Let's add it to our functions:

```
contract Bingo {
	...
	  function bet() public payable {
        require(msg.value >= minBet && msg.value <= maxBet); // if sent amount is in range
        require(players[msg.sender].stage == Stage.BETTING); // if stage is BETTING
    }
}
```

Due to this feature we can check conditions and either allow user to bet/reveal results or not.

Tasks:

- Finish up `bet` function storing new struct in `players[msg.sender]`. Requirements:
  - stage should be `Stage.REVEALING`;
  - block and bet should be caught from global variables;
  - `card` and `generatedNumders` should be empty arrays on 25 and 30 elements accordingly (use `new uint[](SIZE)`)
  - don't initialize mapping `cardsCheker`, it's initialized by default.
- Modify `reveal` checking if the player stage is `Stage.REVEALING`.

### Randomness

We will use `blockhash` to get the hash of the block when the bet was done. This number is seed of the random. And random is: `uint rand = uint256(keccak256(abi.encodePacked(seed, msg.sender)));`

`keccak256` is expencive function so we split `rand` to 32 `uint8` numbers, and only if the count isn't enough it will be recalculated with salt:

```
card = 1 + (rand & 255) % 75;
rand = (rand >> 8 > 0) ?  rand >> 8 : uint256(keccak256(abi.encodePacked(seed, msg.sender, i)));
```

Tasks:

- declare and define `seed` in `reveal` function.

### Loops

There are few loop types in Solidity:

- `for`:

```
for(<<start_counter>>; <<continue_condition>>; <<counter_updater>> ) {
    // body
}
```

- `while`:

```
while(<<condition>>) {
    // body
}
```

For our dapp, we need to generate 25 number on the bought user card and 30 numbers that were randomly chosen and will be compared with that on card. Let's do it with while loop.

Tasks:

- generate 25 cards and store them in `players[msg.sender].card[i]` and set `player.cardsCheker[card]` to its `idx` if the card was chosen.
- store player card in `numbers` variable.
- generate 30 numbers, if the number exists in `player.cardsCheker[card]` set it and `players[msg.sender].card[i]` to zero.
- declare `result` variable;
- check if any column or row in `players[msg.sender].card` is set to zeros, increment `result` in case of true; use for-loop.
- check if diagonals are set to zeros, increment `result` in case of true.

### Sending tokens

Smart-contracts can send amount of tokens to the address. There are few ways: `send`, `call`, `transfer`.
But the most secure way is `transfer`.
It is used in the following way:

```
<<addr>>.transfer(<<amount>>);
```

Tasks:

- send player reward that is equal to `bet * result * payoutPerCombination`; use `safeMath` for safe multiplication.

### Testing

Remix IDE provides easy way to test your contracts. Navigate to `Deploy & Run transaction` tab on the left panel. Test your bet function:

- chose and deploy `Bingo` contract;
- uncover deployed contract in order to see available methods;
- trigger `players` with your address (it is shown on the top of left panel);
- try bet (without sending money); transaction should be reverted that can be checked with logs on the bottom panel;
- try bet sending some value (chose amount using left panel); it should be applied;
- trigger `players` to check if struct was updated correctly.

You may notice that methods have different colours. It provides some information about it modifiers:

- `red` are payable;
- `yellow` are non-payable;
- `blue` are view or pure.

## Deployment Process

Let's deploy smart contracts to Tron Shasta testnet.
Install tronbox using this [instruction](https://developers.tron.network/docs/tron-box-user-guide)

Create file `tronbox.js` in the root of project directory

```
require("dotenv").config();
const port = process.env.HOST_PORT || 9090;

module.exports = {
  networks: {
    mainnet: {
      // Don't put your private key here:
      privateKey: process.env.PRIVATE_KEY_MAINNET,
      userFeePercentage: 100,
      feeLimit: 1e8,
      fullHost: "https://api.trongrid.io",
      network_id: "1"
    },
    shasta: {
      privateKey: process.env.PRIVATE_KEY_SHASTA,
      userFeePercentage: 100,
      feeLimit: 1e9,
      fullHost: "https://api.shasta.trongrid.io",
      network_id: "2"
    },
    development: {
      privateKey: process.env.PRIVATE_KEY_DEVELOPMENT,
      userFeePercentage: 100,
      fullHost: "http://127.0.0.1:" + port,
      network_id: "9"
    }
  },
  compilers: {
    solc: {
      version: "0.4.24"
    }
  }
};

```

This file helps contains configurations for migrations.

Move contracts to the `contracts` directory.
Create directory `migrations` and create file `00_bingo_migration.js` inside it.
Paste the following code:

```
var Bingo = artifacts.require("./Bingo"); // artifacts from build directory, that will appear after compilations

module.exports = function(deployer) { // function that will be called during migration
  deployer.deploy(Bingo);
};

```

Create `.env` file and put you private key for shasta:

```
PRIVATE_KEY_SHASTA="e1b1a881..."
```

Install `dotenv` node module:

```
npm i dotenv
```

Compile contracts:

```
tronbox compile
```

Deploy contracts:

```
tronbox migrate --network shasta
```

## Tasks

1. Improve `Bingo`:
   1. Add contract owner to storage;
   2. Add modifier `onlyOwner`;
   3. Add method to withdraw profit by owner;
   4. Add methods to modify min/max bets and payout by owner.
2. Implement [Keno](https://www.wikihow.com/Play-Keno).
   1. General instructions:
      - one game - one card;
      - there are 80 numbers on card(from 1 to 80);
      - player choses from `minCounter` to `maxCounter` numbers;
      - player pays `counter`\* `pricePerNumber` for betting;
      - during revealing 20 random balls are generated;
      - payout for winners is calculated based on chosen and random numbers match;
      - owner can change game parameters (`pricePerNumber`, `minCounter`, `maxCounter`, etc).
   2. Bonus part:
      - allow player buy few cards for one game;
      - allow player to "buy" few random balls after 20 first were revealed\*.

\*That means game will consist of rounds. After revealing first 20 balls, result isn't calculated but contract waits for player response. Player can either finish game and get results or pay for new random ball. After revealing the ball, contract store it waits for next player action. And repeat it until some ``extraBallLimit` is reached or user send finish action. Note: such multi-round games are not popular among games on smart contracts as it is expensive and long but it would be great to know how to work with it.
