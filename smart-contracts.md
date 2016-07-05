# Ethereum Contract Security Techniques and Tips

[The DAO](https://github.com/slockit/DAO), a crowdfunded investment contract that had substantial security flaws, highlights the importance of security and proper software engineering of blockchain-based contracts. This document outlines collected security tips and techniques for smart contract development. This material is provided as is - and may not reflect best practice. Pull requests are welcome.

**Currently, this document is an early draft - and likely has substantial omissions or errors. This message will be removed in the future once a number of community members have reviewed this document.**

#### Note for contributors

This document is designed to provide a starting security baseline for intermediate Solidity programmers. It includes security philosophies, code idioms, known attacks, and software engineering techniques for blockchain contract programming - and aims to cover all communities, techniques, and tools that improve smart contract security. At this stage, this document is focused primarily on Solidity, a javascript-like language for Ethereum, but other languages are welcome.

##### Additional Requested Content

We especially welcome content in the following areas:

- Testing Solidity code (structure, frameworks, common test idioms)
- Software engineering practices for smart contracts and/or blockchain-based programming

## General Philosophy

Ethereum and complex blockchain programs are new and you should expect an ongoing number of bugs, security risks, and changing best practice - even if you follow the security practices noted in this document. Further, blockchain programming requires a different engineering mindset as it is much closer to hardware programming or financial services programming with high cost to failure and limited release opportunities, unlike the rapid and forgiving iteration cycles of web or mobile development.

As a result, beyond protecting yourself against currently known hacks, it's critical to follow a different philosophy of development:

- **Practice defensive programming**; any non-trivial contract will have errors in it. This means that you likely need to build contracts where you can:
  - pause the contract ('circuit breaker')
  - manage the amount of money at risk (rate limiting, maximum usage)
  - fix and iterate on the code when errors are discovered
  - provide superuser privileges to a party or many parties for contract administration
  - have an orderly wind down process if the unthinkable happens

- **Conduct a thoughtful and carefully staged rollout**
  - Test contracts thoroughly, adding in all newly discovered failure cases
  - Provide hacking bounties starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

- **Keep contracts simple** - complexity increases the likelihood of errors
  - Ensure the contract logic is simple, especially at first when the code is untested or lightly used
  - Modularize code, minimizing performance optimizations at the cost of readability; legibility increases audibility
  - Put logic that requires decentralization on the blockchain, and put other logic off; this allows you to continue rapid iteration off the blockchain

- **Follow all key security resources** (Gitter, Twitter, blogs, Reddit accounts of key members), as newly discovered issues can be quickly exploited

- **Be careful about calling external contracts** (especially, at this stage)
  - look at the dependencies of external contracts
  - remember that many levels of dependencies are particularly problematic

## Security Notifications

This is a list of resources that will often highlight discovered exploits in Ethereum or Solidity:

- [Ethereum Gitter](https://gitter.im/orgs/ethereum/rooms) chat rooms
  - [Solidity](https://gitter.im/ethereum/solidity)
  - [go-ethereum](https://gitter.im/ethereum/go-ethereum)
  - [cpp-ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [Ethereum Blog](https://blog.ethereum.org/): The official Ethereum blog
- [Hacking Distributed](http://hackingdistributed.com/): Professor Sirer's blog with regular posts on cryptocurrencies and security
- [Reddit](https://www.reddit.com/r/ethereum)
- [Vessenes.com](http://vessenes.com/): Peter Vessenes blog

It's highly recommended that you *regularly* read all these sources, as exploits they note may impact your contracts.

Additionally, this is a list of community members who may write about security:

- **Vitalik Buterin**: [Twitter](https://twitter.com/vitalikbuterin), [Github](https://github.com/vbuterin), [Reddit](https://www.reddit.com/user/vbuterin), [Ethereum Blog](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**: [Twitter](https://twitter.com/ethchris), [Github](https://github.com/chriseth), [Ethereum Blog](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**: [Twitter](https://twitter.com/gavofyork), [Blog](http://gavwood.com/), [Github](https://github.com/gavofyork)
- **Dr. Emin Gun Sirer**: [Twitter](https://twitter.com/el33th4xor)
- **Vlad Zamfir**: [Twitter](https://twitter.com/vladzamfir), [Github](https://github.com/vladzamfir), [Ethereum Blog](https://blog.ethereum.org/author/vlad/)

## Key Security Tools

- [Oyente](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf), an upcoming tool, will analyze Ethereum code to find common vulnerabilities (e.g., Transaction Order Dependence, no checking for exceptions)

## Recommendations for Smart Contract Security in Solidity

#### Avoid external calls, when possible

External calls (including `.send`, which triggers the fallback function) can introduce several unexpected risks or errors. For calls to untrusted contracts, you may be executing malicious code in that contract _or_ any other contract that it depends upon. As such, it is strongly encouraged to minimize external calls. Over time, it is likely that a paradigm will develop that leads to safer external calls - but the risk currently is high.

If you must make an external call, ensure that external calls are the last call in a function - and that you've finalized your contract state before the call is made. You should also remember to check the result of all external calls (`.send()` and other raw calls will provide a boolean value, while other external function calls will throw on failure).

Source:

#### Favor `.send()` over `.call()`, `.callcode()`, `.delegatecall()`

`send` is usually safer, as it only has access to gas stipend of 2,300 gas, which is insufficient for the send recipient to trigger state changes (2,300 gas was chosen to allow the recipient to fire an event, but not much else). Raw calls do not have a cap in how much gas can be used, unless explicitly specified. If you do use a raw external call, the return value should be checked to see if it failed - and you should send an explicit amount of gas.

```

```

Source:

#### Always test if `.send` succeeded

Sends (and raw calls) can fail, so you should always test if it succeeded.

```
// bad
someAddress.send();
someAddress.call.value(100000)();


// good
if(!someAddress.send()) {

}

if(!someAddress.call.value(100000)()) {

}
```

Source: [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)

<a name='favor-pull-over-push'></a>
#### Favor *pull* payments over *push* payments

The call stack limit of 1024 means that calls in your contract can fail, if it will be the 1025th call. To prevent sends failing, create a balance in an action that can be separately withdrawn using another action.

In the future, with lower block times and having to interact across shards, asynchrony might become a more important pattern besides for security.

```
// bad
contract auction {
  address highestBidder;
  uint highestBid;

  function bid() {
    if (msg.value < highestBid) throw;

    if (highestBidder != 0) {
      if (!highestBidder.send(highestBid)) { // if the call stack limit was reached, no one would be able to bid
	throw;
      }
    }

    highestBidder = msg.sender;
    highestBid = msg.value;
  }
}

// good
contract auction {
  address highestBidder;
  uint highestBid;
  mapping(address => uint) refunds;

  function bid() {
    if (msg.value < highestBid) throw;

    if (highestBidder != 0) {
      refunds[highestBidder] += highestBid; // record the refund that this user can claim
    }

    highestBidder = msg.sender;
    highestBid = msg.value;
  }

  function withdrawRefund() {
    uint refund = refunds[msg.sender];
    refunds[msg.sender] = 0;
    if (!msg.sender.send(refund)) {
      refunds[msg.sender] = refund;
    }
  }
}
```

Source: [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)

#### Remove arbitrary-length iterations and handle failures in loop gracefully

Blocks of code have an explicit gas limit, and so you should never allow loops that can have arbitrary lengths based on usage. These loops may never conclude, leading to broken contracts - or may allow an attacker to force you to use an excessive amount of gas for computation. Further, when loops call other contracts (like a `.send`), they can fail leading to the failure of the entire loop - if failure is not handled correctly.

```
address[] public refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() {
  for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
    if(refundAddresses[x].send(refunds[refundAddresses[x])) {
      throw; // doubly bad, now a single failure on send will hold up all funds
    }
  }
}

// good
mapping (address => uint) public refunds;

function getRefund() {
  if(refunds[msg.sender] > 0) {
    uint amountToSend = refunds[msg.sender];
    refunds[msg.sender] = 0;

    if(!msg.sender.send(refunds[msg.sender])) {
      refunds[msg.sender] = amountToSend;
    }
  }
}

```

Source: [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)


#### Beware rounding with integer division

All integer divison rounds down to the nearest integer - use a multiplier to keep track, or use the future fixed point data types.

```
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

// in the near future, Solidity will have fixed point data types like fixed, ufixed
```

Source:

#### Name functions and events differently

Favor capitalization and a *Log* prefix in front of events, to prevent the risk of confusion between functions and events (this was a mistake made in [The DAO](https://github.com/slockit/DAO/) ). For functions, always start with a lowercase letter.

Source: [Deconstructing the DAO Attack: A Brief Code Tour](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)

```
// bad
event transferHappened() {}
function Transfer() {}

// slightly better
event TransferHappened() {}

// good
event LogTransferHappened() {}
function transfer() {}
```

Source:

#### Explicitly mark visibility in functions and state variables

Explicitly label the visibility of functions and state variables. Functions can be specified as being external, public, internal or private. For state variables, external is not possible.

```
// bad
uint x; // the default is public protected, but it should be made explicit
function transfer() {

}

// good
uint public protected y;
function transfer public () {

}
function internalAction private () {

}

// for state variables, 'external' is not possible
```

#### Keep fallback functions simple

Fallback functions are the default functions called when a contract is sent a message with no arguments, and only has access to 2,300 gas when called from a `.send()` call. As such, the most you should do in most fallback functions is call an event. Use a proper function if a computation or more gas is required.

```
// bad
function () { balances[msg.sender] += msg.balance; }

// good
function() { throw; }
function() { LogSomeEvent(); }
function deposit() { balances[msg.sender] += msg.balance; }
```

#### Mark untrusted contracts

Mark which contracts are untrusted.  For example, some abstract contracts are implementable by 3rd parties. Use a prefix, or at minimum a comment, to highlight untrusted contracts.

```
// bad
Bank.withdraw(100);

// good
ExternalBank.withdraw(100);
```

## Known Attacks

1. [Call depth attack](#call-depth-attack)
1. [Iterator Deadlocking](#iterator-deadlocking)
1. [Iteration Maximum](#iteration-maximum)
1. [Reentrant Attacks](#reentrant-attacks)
1. [Timestamp Dependence](#timestamp-dependence)
1. [Transaction-Ordering Dependence](#transaction-ordering-dependence)
1. [Leech Attack](#leech-attack)

<a name="call-depth-attack"></a>
#### Call depth attack

Given the call stack limit of 1024, a malicious party can build up a chain of calls that forces subsequent calls within a contract to fail, even with sufficient gas. These subsequent calls can be functions within a contract and external contract calls (including `.send()`).


Example:

```
function bid() {
  if (msg.value > highestBid) {
    address previousBidder = msg.sender;
    highestBidder = msg.sender;
    highestBid = msg.Value;

    previousBidder.send(); // this send will fail, meaning the money is retained within the contract even though the bidder changed
}
```

All raw external calls should be examined and handled carefully for errors.  In most cases, the return values should be checked and handled carefully.  We recommend explicit comments in the code when such a return value is deliberately not checked.

<a name="iterator-deadlocking"></a>

#### Iterator Deadlocking

Loops that depend on external calls succeeding can fail due to a single malicious party.

```
address[] refunds; // assume this array is populated

for (uint x = 0; x < refunds.length; x++) {
  if (refundAddresses[x].send(100)) { // any single address that overrides send to return a failure can now prevent payments for the entire group
    throw;
  }
}
```

The recommended pattern is that each user should withdraw their refund themselves.

<a name="iteration-maximum"></a>

#### Iteration Maximum

Manipulating the amount of elements in an array can increase gas costs substantially - or prevent the code from running if the block maximum is hit. For example, a malicious party can create many entities to break a iterative payout code.

One way to mitigate this is to transform push models to an individual pull model. See [favor pull over push](#favor-pull-over-push).

An alternative approach is to have a payout loop that can be split across multiple transactions, like so:

```
struct Payee {
  address addr;
  uint256 value;
}
Payee payees[];
uint256 nextPayeeIndex;

function payOut() {
  uint256 i = nextPayeeIndex;
  while (i < payees.length && msg.gas > 200000) {
    payees[i].addr.send(payees[i].value);
    i++;
  }
  nextPayeeIndex = i;
}
```
<a name="reentrant-attacks"></a>

#### Reentrant Attacks

A key benefit of Ethereum is the ability for one contract to call another. Additional functionality can then be easily attached to existing smart contract ecosystems - but this also can introduce risks. One of these is reentrant attacks: allowing a function to be called in a contract when another function in that contract is running.

External calls (Contract OR raw calls) are particularly problematic when one does not know what the other contract might do when triggered. This can cause reentrant attacks, such as recursive & unexpected state manipulation attacks.

(The DAO hack combined both these attacks)

##### Recursive Reentrant Attack

This attack occurs when the state has not been properly set before the external call occurs in a function. The attacker can reenter and call the function again, being to recursively call a piece of code, whilst it was expected that they could only do it once. They can only recursively call until the gas limit has been reached or before the call depth is reached.

This is particularly problematic where it is not known that something like `address.call.value()()` to send ether, can trigger a fallback function. However, this is not just limited to a fallback function. It can happen with any external (Contract or raw).

```
// Code from The DAO: https://github.com/slockit/DAO/blob/develop/ManagedAccount.sol

function payOut(address _recipient, uint _amount) returns (bool) {
  if (msg.sender != owner || msg.value > 0 || (payOwnerOnly && _recipient != owner))
      throw;
  if (_recipient.call.value(_amount)()) {
      PayOut(_recipient, _amount);
      return true;
  } else {
      return false;
  }
}
```

To protect against recursive reentry, the function needs to set the state such that if the function is called again in the same transaction, it wouldn’t continue to execute.

##### Unexpected State Manipulation Reentrant Attack

This attack occurs when a function expects a certain state, but another contract function alters this state, while the original function is still running. A malicious party would often start the first function, and then calls the second function before the first function completes.

In [The DAO](https://github.com/slockit/DAO), when the splitting function was called, the token balances were zeroed out. However, before that was done, the attacker reentered and transfer out his/her balance to another address, letting the split function zero out an account that already had zero funds in it. This allowed the attacker to again call the splitting function in multiple transactions, as long as s/he kept transferring the tokens around before the final split.

```
TODO: Add snippet
```
If there are more than 2 functions that share state, an attacker can cause substantial damage. Functions that do not share any state, are safe from this attack (even if state changes happen after the external call). It is not generally a safe pattern, so it is still recommended to do state changes before doing any external calls even if no functions share states in your contract.

To mitigate this attack, you should:

- Always check the result of a contract call, even when using `send()` (don't ever assume things went well)
- Only make external calls at the end of functions

Otherwise, you MUST ensure that functions do not share state:

```
contract ReentrantSafe {
  uint fState;
  uint gState;

  f()  // only changes fState
  g() // only changes gState
}
```

Alternatively, you can use a mutex (often used in concurrent programming) where you lock certain variables:

```

```

Mutexes have their own disadvantages with the potential for deadlocks and reduced throughput - so choose the approach that works best for your use case and text extensively.

<a name="timestamp-dependence"></a>

#### Timestamp Dependence Bug

The timestamp of the block can be manipulated by the miner - and should not be used for important parts of the code.

A way around this could be to use block numbers and estimate time that has passed based on the average block time. However, this is NOT future proof as block times might change in the future (such as the current planned 4 second block times in Casper). So, consider this when using block numbers as as timekeeping mechanism (how long your code will be around).

<a name="transaction-ordering-dependence"></a>

#### Transaction-Ordering Dependence (TOD)
(also known as frontrunning)

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is recorded (included in a block).

This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another potential way to use a pre-commit scheme (“I’m going to submit the details later”).

<a name="leech-attack"></a>

#### Leech attack

**TODO: Is this an 'attack'?**

If your contract is an oracle, it may want protection from leeches that will use your contract’s data for free. If not encrypted, the data would always be readable, but one can restrict the usage of this information in other smart contracts.T

Part of the solution is to carefully review the visibilities of all function and state variable.

## Software Engineering Techniques

Designing your contract for unknown, often unknowable, failure scenarios is a key aspect of defensive programming, which aims to reduce the risk from newly discovered bugs. We list potential techniques you can use to mitigate this unknown failure scenarios.

#### Testing

This section is under construction.


##### Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
  - exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

```
modifier isActive() {
  if (now > SOME_BLOCK_NUMBER) {
    throw;
  }
  _
}

function deposit()
isActive() {
  // some code
}

function withdraw() {
  // some code
}

```


##### Use fake Ether or restrict amount of Ether per user/contract

In the early stages, you can use tokens to represent large amounts of Ether, or restrict the amount of Ether for any user (or for the entire contract) - reducing the risk.

#### Permissioned Guard (changing code once deployed)

Code will need to be changed if errors are ever discovered - and there are various techniques to do this. The simplest is to have a registry contract that holds the address of the latest contract. An easier approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components (data, logic) - so that code changes do not break functionality or require substantial costs to port. Additionally, it's critical to have a **secure** way for parties to upgrade the code - and this may be a single trusted party, a group of members, or require a vote/quorom from the full set of stakeholders.

**Example 1: Use a registry contract to store latest version of a contract**

```
contract SomeRegister {
  address backendContract;
  address[] previousBackends;

  function changeBackend(address newBackend) {
      if(newBackend != backendContract) {
	previousBackends.push(backendContract);
	backendContract = newBackend;
	return true;
      }

      return false;
  }
}

```


**Example 2: Use a `DELEGATECALL` to forward data and calls**

```
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
      if (msg.sender != owner) {
	throw;
      }
      _
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(newVersion)
    onlyOwner()
    {
	currentVersion = newVersion;
    }

    function() {
        if(!currentVersion.delegatecall(msg.data)) throw;
    }
}
```
Source: [Stack Overflow](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)

#### Circuit Breakers (Pause contract functionality)

Circuit breakers stop contract code from being executed if certain conditions are met, and can be useful when new errors are discovered. They sometimes come at the cost of injecting some level of trust, though smart design can minimize the trust required. Pausing can protect ether and many other items (e.g., votes).

Example:

```
bool private paused = false;

function public toggleContractActive()
isAdmin() {
    // You can add an additional modifier that restricts pausing a contract to be based on another action, such as a vote of users
    paused = !paused;
}

modifier isAdmin() {
  if(msg.sender != owner) {
    throw;
  }
  _
}

modifier isActive() {
  if(paused) {
    throw;
  }
  _
}

function transfer()
isActive() {
  // some code
}
```

#### Speed Bumps (Delaying contract results)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 28 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, allowing a greater likelihood of recovery (other fundamental flaws made this functionality useless without a fork in Ethereum). Speed bumps can be combined with other techniques (like contract pausing).

Example:

```
mapping (address => uint) balances;
mapping (address => uint) requestedWithdrawals;

struct RequestedWithdrawal {
  uint amount;
  uint time;
}

function requestWithdrawal() {
  if (balances[msg.sender] > 0) {
    uint amountToWithdraw = balances[msg.sender];
    balances[msg.sender] = 0;

    requestedWithdrawals[msg.sender] = RequestedWithdrawal({
	amount: amountToWithdraw;
	time: now
      };
  }
}

function withdraw() {
  if(requestedWithdrawals[msg.sender] && requestedWithdrawals[msg.sender].amount> 0 &&
    ) {

    }
}

```

#### Rate Limiting

Malicious activity is often correlated with extreme movements (e.g., size of fund transfer, number of transfer requests). This risk can be reduced by limiting amount of activity allowed within a time period or by a certain address.

Example:

```

```

#### Whitelist Users

Locking certain or all actions to a list of whitelisted users or contracts can prevent a malicious user from accessing your contract.

#### Assert Guards

Assert failures allow developers to upgrade the code only if an assert failure is triggered. For example, if balances exceed the contract amount.

#### Manage what is on Blockchain

Blockchain programming is difficult because it has a level of permanence and data cost different than other forms of programming. In many cases, you may be able to put core functionality on the blockchain, while managing ancillary functionality off the blockchain. This allows you the benefits of trustless, decentralized systems - with the advantages of iteration off the blockchain.

#### Automatic Wind Down

If the contract is attacked and paused - you may want to provide a facility to wind down the contract, rather than fixing it. It's critical to have an orderly wind down process from the start and think through various cases, such as:

- How should money be returned, if there is not enough to return all users' funds
- Who will be able to start the wind down process

## Security-related Documentation and Procedures

When launching a contract that will have substantial funds or is required to be mission critical, it is important to include proper documentation. Some documentation related to security includes:

**Status**

- Where current code is deployed
- Current status of deployed code (including outstanding issues, performance stats, etc.)

**Known Issues**

- Key risks with contract
  - e.g., You can lose all your money, hacker can vote for certain outcomes
- All known bugs/limitations
- Potential attacks and mitigants
- Potential conflicts of interest (e.g., will be using yourself, like Slock.it did with the DAO)

**History**

- Testing (including usage stats, discovered bugs, length of testing)
- People who have reviewed code (and their key feedback)

**Procedures**

- Notification process if bug is discovered
- Wind down process if something goes wrong (e.g., funders will get percentage of your balance before attack, from remaining funds)
- If hacker bounty*provided for discovered bugs, responsible disclosure policy, where to report, etc
- Recourse in case of failure (e.g., insurance, penalty fund, no recourse)

**Contact Information**

- Who to contact with issues?
- Names of programmers and/or other important parties
- Chat room where questions can be asked

This information ideally exists on a wiki editable by the community, served over HTTPS/TLS.

## Future improvements

- **Editor Security Warnings**: Editors will soon alert for common security errors, not just compilation errors. Browser Solidity is getting these features soon.

- **New functional languages that compile to EVM bytecode**: Functional languages gives certain guarantees over procedural languages like Solidity, namely immutability within a function and strong compile time checking. This can reduce the risk of errors by providing deterministic behavior. (for more see [this](https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k), Curry-Howard correspondence, and linear logic)

## Noted Security Blog Posts

##### Writing Safer Contracts

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity): Blog post from Devcon 1 from the creator or Solidity
- [Making Smart Contracts Smarter](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf) (Loi Luu, Duc-Hiep Chu, Prateek Saxena, Hrishi Olickel, Aquinas Hobor)
- [We need fault-tolerant smart contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc) (Peter Borah)
- [We need Escape Hatches](http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/) (Hacking Distributed)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [Assert Guards: Towards Automated Code Bounties & Safe Smart Contract Coding on Ethereum](https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c) (Simon de la Rouviere)
- [Safer Smart Contracts through type-driven development](http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf) (Jack Pettersson and Robert Edström)
- [Simple Contracts are Better Contracts](https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a)
- [In Bits We Trust?](https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b) (David Xiao)

##### Common Contract Errors

- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [More Ethereum Attacks: Race-to-empty is the Real Deal](http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal/)
- [Devcon1 and Ethereum Contract Security](http://martin.swende.se/blog/Devcon1-and-contract-security.html)
- [Potential Attacks against Rouleth contract](https://github.com/Bunjin/Rouleth/blob/master/Security.md)
- [Ethereum Griefing Wallets: Send w/Throw Is Dangerous](http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful/)

##### DAO-related Security Posts

- [Deja Vu DAO Smart Contracts Audit Results](https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e#.x9frbu72d) (Stephen Tual)
- [DAO Call for Moratorium](http://hackingdistributed.com/2016/05/27/dao-call-for-moratorium/) (Dino Mark, Vlad Zamfir, and Emin Gün Sirer)
- [Analysis of the DAO Exploit](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/) (Phil Daian/Hacking Distributed)
- [Deconstructing the DAO Attack](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)
- [Chasing the DAO Attacker's Wake](https://pdaian.com/blog/chasing-the-dao-attackers-wake/) (Phil Daian)

##### Other

[Least Authority Security Audit](https://github.com/LeastAuthority/ethereum-analyses)

## Reviewers

The following people have reviewed this document (date and commit they reviewed in parentheses):

-


#### License

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)
