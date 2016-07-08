
## General Philosophy

Ethereum and complex blockchain programs are new and therefore you should expect an ongoing number of bugs, security risks, and changing best practice - even if you follow the security practices noted in this document. Further, blockchain programming requires a different engineering mindset as it is much closer to hardware programming or financial services programming with high cost to failure and limited release opportunities, unlike the rapid and forgiving iteration cycles of web or mobile development.

As a result, beyond protecting yourself against currently known hacks, it's critical to follow a different philosophy of development:

- **Practice defense in depth**. You shoudl design contracts to gracefully manage (potentially inevitable) failure, using multiple layers of security controls. Many of these failure conditions are unknown and often unknowable in a new ecosystem like Ethereum - and so require more than simply managing for all currently known bugs, antipatterns, or malicious inputs. As such, you may need to:
  - pause the contract ('circuit breaker')
  - restrict the amount of money at risk (rate limiting, maximum usage)
  - fix and iterate on the code when errors are discovered
  - provide superuser power to a party or many parties for contract administration
  - *(these are some illustrative examples, and multiple techniques should be layered)*

- **Conduct a thoughtful and carefully staged rollout**
  - Test contracts thoroughly, adding in all newly discovered failure cases
  - Provide hacking bounties starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

- **Keep contracts simple** - complexity increases the likelihood of errors
  - Ensure the contract logic is simple, especially at first when the code is untested - or lightly used
  - Modularize code, minimizing performance optimizations at the cost of readability; legibility increases audibility
  - Put logic that requires decentralization on the blockchain, and put other logic off; this allows you to continue rapid iteration off the blockchain

- **Follow all key security resources** (Gitter, Twitter, blogs, Reddit accounts of key members), as newly discovered issues can be quickly exploited

- **Be careful about calling external contracts** (especially, at this stage)
  - look at the dependencies of external contracts
  - remember that many levels of dependencies are particularly problematic


<sup>\*</sup>[Defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) uses *multiple layers* of controls to provide redundancy in the event that security controls fail. It can be extended beyond code to the personnel, procedural, and physical aspects of security (e.g., the human-based voting mechanism used to replace a piece of code).
