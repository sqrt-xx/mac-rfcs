- Title: escrow-protocol
- Authors: [Marek Narozniak](mailto:marek.narozniak@protonmail.com)
- Start date: March 27th, 2024

---

## Summary
[summary]: #summary

MAC! is a zkApp which enables anyone to create zkApps (smart contracts) on MINA blockchain. This RFC aims to clarify of the necessary concepts and explain the escrow protocol behind MAC!.

## Motivation
[motivation]: #motivation

At the moment of writing this RFC, MAC! has been funded already three times and the most basic version is publicly available. Regardless of that, the mechanisms and motivation behind it still remains confusing to most members of the community.

As the development continues and amount of new feature ideas growth, there also is a growing documentation-debt. The amount of questions from the community grows and a need of proper documentation of what MAC! is and how it works become more and more significant.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

### Blockchain technology, smart contracts and zero-knowledge proofs

Centralized systems rely on a single authority or entity to maintain control and manage operations, offering efficiency but often at the expense of transparency and autonomy. Conversely, decentralized systems distribute authority across a network of participants, fostering resilience, transparency, and democratic decision-making.

At the heart of many decentralized systems lie consensus algorithms, making it possible for participants of the network to agree on information state without the need of central authority nor trusted intermediaries. This is the foundation of various blockchain technologies, such as cryptocurrencies.

Embedded within the ethos of decentralization is the Cypherpunk philosophy, a movement advocating for privacy, cryptography, and individual sovereignty in the digital age. Cypherpunks envision a future where individuals retain control over their personal data and interactions, free from the prying eyes of governments and corporations.

One of the most compelling applications of blockchain technologies within the realm of decentralized systems is the concept of smart contracts. These self-executing contracts, encoded with predefined rules and conditions, automate and enforce agreements without the need for intermediaries, reducing costs and increasing efficiency across various industries. One particular example of a smart contract would be an Escrow smart contract, which is a blockchain account capable of handling transactions, accepting to and sending from blockchain users or even other smart contracts. Escrow smart contracts can have an embedded logic, conditions which need to be satisfied for transaction to be valid. This can be used to enforce terms of the contract. This RFC describes an escrow smart contract with an arbitration protocol which is using zero-knowledge proofs technology for privacy.

In traditional blockchain implementations such as Bitcoin (BTC) the overal state of every account remains public, exposing entire transaction history of that account. In case of Ethereum (ETH) which is another cryptocurrency which in addition, permits the use of smart contracts, the interactions between users of the blockchain and the smart contract by default remain public as well. This makes it significantly more difficut when it comes to adoption of smart contract technology into our daily lives. Most of us probably would not feel comfortable having all the interactions and deals remained public to everyone.

The zero-knowledge proof (ZK) technology is a set of cryptographic techniques making it possible to verify the validity of the information without exposing some or all the data. For example, using a range-proof, which is a kind of zero-knowledge proof one can expose a hash of a number along with a zero-knowledge proof that the hashed values is contained within certain range without revealing the number itself.

Zero-knowledge proofs technology can be useful in many areas of our lives, it also finds its place in securing blockchain technology which is particularly vulnerable to private information leaks. The Mina blockchain is based on the zero-knowledge technology and provides tools of constructing smart contracts which are also based on the zero-knowledge proofs making it possible for everyone to verify the contract computation without accessing sensitive data which only the involved parties of the contract should be familiar with.

### MAC! is a zero-knowledge escrow smart contract generator

An arbitration protocol describes an interaction between multiple parties in which some of the parties are in charge of certifying or asserting certain requirements have been satisfied. An escrow smart contract with embedded arbitration protocol would involve following parties:

1. Employers (E), who pay for having something done,
2. Contracts (C), who are paid to execute tasks,
3. Arbiters (A), who are in charge of asserting if contractors fulfiled the terms of a deal.

Such an escrow smart contract would provide deposit and withdrawal methods as well as methods for (A) allowing to assert if deal is successful and failed. In this RFC we consider a most basic setup involving exactly three parties, a single (E), single (C) and single (A). We intend to document this protocol with a greated detail so that a pre-coded smart contract could handle as many cases possible.

The security scheme of this application is based on security deposits. Without them, (E) would be the only party needed to deposit, (C) would be the only party who needs withdrawal and (A) would basically assert for free. There would be no mechanism preventing parties of the contract from cheating as (C) could simply accept the deal just to hope (A) would make a mistake or maybe enter a deal with (A) to collectively cheat on (E). Security deposits cannot completely prevent that, but they are meant to mitigate such risks. For instance, if every party has something to lose in case of cheating it becomes less profitable to cheat. The smart contracts described by this RFC allow the contract creator to pre-define each outcome payout scenario and adjust the security deposit to prevent malicious behaviours of the involved parties.

### MAC! intended impact

One might argue that smart contracts are not meant to be executed by humans. There is a point in that statement. Human participants, such as (A) are unreliable, even oracles are not considered completely secure. In case of the protocol described in this RFC it is possible for (A) to be another smart contract, such as oracle or some completely trustless algorithm. This RFC does not go deeper into that aspect and it remains to be investigates and specified in one of the future RFCs.

The human-executable smart contracts are justified by their ridiculously slow development costs and universality. Such smart contracts could be created by anyone using a graphical user interface and potentially secure the real-life interactions bringing value to people who were not interested in blockchain technologies until this moment. Given the fact that (A) by default is a blockchain user and defined contract can predict compensation for (A), it brings a potential for creating arbitration markets where public trust could be monetized and provide a source of income. Since it requires no programming skills to develop and deploy those smart contracts, the (E) and (C) parties could be arbitrary non-technical individuals or businesses who wish to secure their deal using blockchain technology with security deposits.

### Future of MAC!

In the future, it would be beneficial to extend this RFC to allow
1. Unlimited outcomes instead of just `success` and `failure`,
2. Multiple (A) with voting and sortition,
3. Machine readable metadata / automating arbitration,
4. Eco-friendly mode (contract re-using to avoid paying 1 Mina account creation fee for each interaction).

## Technical-level explanation

### Architecture overview

In order to fully grasp this protocol one must understand the relevant data (such as parties of the contract and its description) and the data structures that represent this information. Moreover, understanding of allowed smart contract methods, their preconditions and postconditions is also necessary.

#### Relevant values

Some values will be often references in this RFC,

| Name | Description               |
|------|---------------------------|
| `bl` | current blockchain length |

#### Contract parties

From the technical perspective, every party of the contract is a MINA on-chain account. There are four specific pre-defined roles.

1. Employer (E) - who pays for what is the contract subject,
2. Contractor (C) - who received the payment for the contract subject,
3. Arbiter (A) - who judges the contract outcome as `success` or `failure`,
4. Deployer (D) - who deploys the smart contract, (D) can be anyone willing to cover the 1 Mina account creation fee and has access to private key of the contract.

#### Data structures

##### Participant

The `Participant` data-structure holds the contract participant public key and provides serialization / deserialization capabilities.

| Type      | Name                | Description                                                                                  |
|-----------|---------------------|----------------------------------------------------------------------------------------------|
| PublicKey | participant_address | MINA public key address existing on chain held by the contract participant.                  |

##### Outcome

The `Outcome` data-structure holds the necessary data for contract balance changes and deadlines for a particular outcome of the contract. It also holds human-readable description of that outcome in case if contract creator wishes to justify the payout or deadline policy.

| Type            | Name                 | Description                                                                                  |
|-----------------|----------------------|----------------------------------------------------------------------------------------------|
| `CircuitString` | `description`        | Human-readable description of this outcome.                                                  |
| `UInt64`        | `payment_employer`   | Escrow contract balance difference allowed from employer.                                    |
| `UInt64`        | `payment_contractor` | Escrow contract balance difference allowed from contractor.                                  |
| `UInt64`        | `payment_arbiter`    | Escrow contract balance difference allowed from arbiter.                                     |
| `UInt32`        | `start_after`        | MINA blockchain length value after which this outcome becomes possible to occur. (inclusive) |
| `UInt32`        | `finish_before`      | MINA blockchain length value until which this outcome is possible to occur. (exclusive)      |

Values are always positive, whether they are sent to the contract or withdrawn from the contract depends on the type of the outcome. There are four pre-defined types of outcomes.

##### Contract preimage

Entire contract can be defined, created and deployed by anyone, even someone who does not take part in the contract.

The smart contract is designed to work with any escrow contract that is compatible with this RFC. Necessary data required to interact created as `CircuitValue`-typed data structure is listed in further subsections of this section. We call it a `Contract preimage`. The contract preimage can be serialized / deserialized into a binary string. This serialized data can be shared between parties of the deployed contract allowing them to interact.

The poseidon hash of the serialized contract preimage is called a `contract commitment`.

| Type          | Name             | Description                                                                                      |
|---------------|------------------|--------------------------------------------------------------------------------------------------|
| Field         | protocol_version | Fixed for `0` for this RFC, allows to modify the standard in the future                          |
| Field         | nonce            | Random value preventing preimage attack                                                          |
| CircuitString | contract         | Written, human-readable description of the contract                                              |
| PublicKey     | address          | Public key of the smart contract, deployer needs to have access to the corresponding private key |
| Participant   | employer         | Employer participant data                                                                        |
| Participant   | contractor       | Contractor participant data                                                                      |
| Participant   | arbiter          | Arbiter participant data                                                                         |
| Outcome       | deposited        | Deposited outcome data                                                                           |
| Outcome       | success          | Success outcome data                                                                             |
| Outcome       | failure          | Failure outcome data                                                                             |
| Outcome       | cancel           | Cancel outcome data                                                                              |
| Outcome       | unresolved       | Unresolved outcome data                                                                          |

###### Outcome values table and validation

For the purpose of further protocol design, let us label and summarize all the outcome values under the form of a single table.

| Outcome    | payment_employer | payment_contractor | payment_arbiter | start_after | finish_before |
|------------|------------------|--------------------|-----------------|-------------|---------------|
| deposited  | `odpe`           | `odpc`             | `odpa`          | `odsa`      | `odfb`        |
| success    | `ospe`           | `ospc`             | `ospa`          | `ossa`      | `osfb`        |
| failure    | `ofpe`           | `ofpc`             | `ofpa`          | `ofsa`      | `offb`        |
| cancel     | `ocpe`           | `ocpc`             | `ocpa`          | `ocsa`      | `ocfb`        |
| unresolved | `oupe`           | `oupc`             | `oupa`          | `ousa`      | `oufb`        |

Following outcomes are labeled but never occur in the contract logic: `ocsa`, `ocfb`, `ousa`, `oufb`.

The sums of possible withdrawals and deposits must cancel out.
```
odpe + odpc + odpa = ospe + ospc + ospa
= ofpe + ofpc + ofpa
= ocpe + ocpc + ocpa
= oupe + oupc + oupa
```

Every action needs non-zero amount of blocks to occur
```
odsa < odfb
ossa < osfb
ofsa < offb
```

It is not required, but recommended to have failure declaration included withing possible success declaration
```
ossa < ofsa < offb <= osfb
```
which avoids situation under which the Arbiter forgot to judge success and judges failure just to avoid an unresolved state.

#### On-chain data

Each smart contract generated and deployed holds on-chain three `Field`-typed values.

##### commitment

Poseidon hash of the serialized contract preimage. Storing this value on-chain helps ensure participants interact with the contract they agreed upon during contract creation.

##### automaton_state

An integer value indicating the state of the contract automaton. It is contained between `0` and `4`, each integer value holds a different meaning of current state and depending on that different actions are allowed by the contract.

| automaton_state | Description          | Withdrawal Outcome | Explanation                                                                                                           |
|-----------------|----------------------|--------------------|-----------------------------------------------------------------------------------------------------------------------|
| 0               | Initial              | N/A                | Contract is deployed and awaiting deposits to be made.                                                                |
| 1               | Deposited            | Unresolved         | Everyone has made deposit of the funds. Withdrawal only possible according to `unresolved` outcome after `offb <= bl` |
| 2               | Canceled for free    | Deposit            | Contract has been canceled before all deposits were made. It is possible to withdraw deposited amounts.               |
| 3               | Canceled for penalty | Cancel             | Contract has been canceled by the contractor during execution stage.                                                  |
| 4               | Succeeded            | Success            | According to the Arbiter, the contractor succeeded to fulfil the contract within required time frame.                 |
| 5               | Failed               | Failure            | Indicates that according to the Arbiter, the contractor failed to fulfil the contract within required time frame.     |

##### memory

It is an integer value contained between `0` and `7`. Its binary encoding indicates which of the involved parties made action. The leftmost bit is `Employer`, middle bit is `Contractor` and the rightmost bit is `Arbiter`. Under this encoding, `000` means noone took an action, `110` means the `Employer` and `Contractor` made and action and the `Arbiter` did not. The `111` would indicate everyone took an action.

Which action exactly it represents depends on the automaton state. For instance, if automaton state indicates `0` meaning `initial` stage the action would indicate deposit and `110` memory would indicate the `Employer` and `Contractor` made their deposits and `Arbited` did not made the deposit yet.

| memory | ECA | who acted                     |
|--------|-----|-------------------------------|
| 0      | 000 | Nobody                        |
| 1      | 001 | Arbiter                       |
| 2      | 010 | Contractor                    |
| 3      | 011 | Contractor, Arbiter           |
| 4      | 100 | Employer                      |
| 5      | 101 | Employer, Arbiter             |
| 6      | 110 | Employer, Contractor          |
| 7      | 111 | Employer, Contractor, Arbiter |

#### Actions

The contract actions are methods which can be executed.

1. Deploy
2. Initialize
3. Deposit
4. Withdraw
5. Declare success
6. Declare failure
7. Cancel

The deploy, initialize and withdraw methods are complex examples and we will detail those later and we will start by focusing on simpler methods.

##### General preconditions

For most methods (unless said otherwise) it is required to

1. Check if provided preimage results in commitment stored onchain. This ensures executing party has access to the smart contract conditions.
2. Check if executing party is one of the involved parties: Employer, Contractor or Arbiter.

The remaining preconditions are contract specific and will be specified accordingly.

##### Deposit, declare success, declare failure and cancel

The simple methods can be summarized in a single table. Some actions occur more than once as depending on contract state they might create different postconditions requiring multiple rows to explain same action. General preconditions do apply.

| action          | automaton state (current) | automaton state (next) | start after | finish before | acted    | actor      | last | zero memory |
|-----------------|---------------------------|------------------------|-------------|---------------|----------|------------|------|-------------|
| deposit         | Initial                   | Initial                | `odsa`      | `odfb`        | must not | N/A        | no   | no          |
| deposit         | Initial                   | Deposited              | `odsa`      | `odfb`        | must not | N/A        | yes  | yes         |
| declare success | Deposited                 | Success                | `ossa`      | `osfb`        | N/A      | Arbiter    | N/A  | yes         |
| declare failure | Deposited                 | Failed                 | `ofsa`      | `offb`        | N/A      | Arbiter    | N/A  | yes         |
| cancel          | Initial                   | Canceled for free      | `odsa`      | N/A           | N/A      | N/A        | N/A  | no          |
| cancel          | Deposited                 | Canceled for penalty   | `ossa`      | `osfb`        | N/A      | Contractor | N/A  | yes         |

The column `acted` value `must not` indicated that the actor who executes the action must not have acted based on memory state. On the other hand, `must` indicates that according to the memory state, the actor executing the action must be someone who have acted.

The column `last` indicates if it is required for the actor the be the last one who acts according to the contract memory state.

The column `zero memory` indicates the contract memory should be reset after execution.

For all the columns, if `N/A` value occurs, it means this column value is not relevant.

##### Withdraw method

The withdrawal method has a dedicated section as its execution is conditional and depends on multiple criteria which are summarized in a table below. General preconditions do apply.

| automaton state (current) | withdrawal allowed | start after | acted    | E      | C      | A      |
|---------------------------|--------------------|-------------|----------|--------|--------|--------|
| Initial                   | no                 | N/A         | N/A      | N/A    | N/A    | N/A    |
| Deposited                 | yes                | `offb`      | must not | `oupe` | `oupc` | `oupa` |
| Canceled for free         | yes                | N/A         | must     | `odpe` | `odpc` | `odpa` |
| Canceled for penalty      | yes                | N/A         | must not | `ocpe` | `ocpc` | `ocpa` |
| Success                   | yes                | N/A         | must not | `ospe` | `ospc` | `ospa` |
| Failure                   | yes                | N/A         | must not | `ofpe` | `ofpc` | `ofpa` |

The `withdrawal allowed` column indicates if the withdrawal method is allowed purely based on the automaton state.

The `E`, `C` and `A` columns indicate the withdrawable amounts for (respectively) `Employer`, `Contractor` and `Arbiter`.

##### Deployment and Initialization

The general preconditions do not apply for deployment nor initialization. Deployment can be done by anyone who has contract private key and is willing to pay MINA account creation fee. The initialization is run in the same transaction as deployment and its purpose is to set:

1. contract commitment to preimage hash
2. automaton state to `Initial`
3. memory to `0`

It is required to prevent execution of the initialization method in any other case than right after deployment. Reason for that is avoiding an attack scenario that aims to disrupt of execution of an already ongoing contract.

#### Preimage generation algorithm example

The previous section have defined the form of a preimage and how various methods can be executed depending on the preimage and onchain state of the smart contract. We did not explain how such preimage can be defined. There is no one way to do it and specific details of that are sensitive and depend on what the involved parties intend to achieve. For that reason this section is only an example of preimage definition.

Let us define following values which are introduced by the contract creator.

| Value      | Unit              | Description                                                                    |
|------------|-------------------|--------------------------------------------------------------------------------|
| cp         | MINA              | Contractor payment                                                             |
| cd         | MINA              | Contractor security deposit                                                    |
| cfp        | %                 | Contractor failure to deliver penalty                                          |
| ccp        | %                 | Contractor cancel penalty                                                      |
| ed         | MINA              | Employer security deposit                                                      |
| eas        | %                 | Employer arbitration fee share                                                 |
| ap         | MINA              | Arbiter payment                                                                |
| ad         | MINA              | Arbiter security deposit                                                       |
| anp        | %                 | Arbiter non-acting penalty                                                     |
| wd         | Blockchain length | Warm-up phase duration, in number of blocks                                    |
| dd         | Blockchain length | Deposit phase duration, in number of blocks                                    |
| ed         | Blockchain length | Execution phase duration, in number of blocks                                  |
| fd         | Blockchain length | Failure declaration phase duration, in number of blocks                        |
| subject    | String            | Written description of the contract                                            |
| deposit    | String            | Written description and justification of deposit outcome deadlines and amounts |
| success    | String            | Written description and justification of success outcome deadlines and amounts |
| failure    | String            | Written description and justification of failure outcome deadlines and amounts |
| cancel     | String            | Written description and justification of success outcome amounts               |
| unresolved | String            | Written description and justification of success outcome amounts               |

The string-types values can directly be converted to `CircuitString` type variables and put in the preimage as contract description (subject) and descriptions of each of the outcomes justifying the policy. To define the preimage we still need to compute all the deadlines and payout values.

The deposit amounts
```
odpe = ed + cp + eas*ap
odpc = cd + (1-eas)*ap
odpa = ad
```

The success outcome withdrawable amounts
```
ospe = ed
ospc = cd + cp
ospa = ad + ap
```

The failure outcome withdrawable amounts
```
ofpe = ed + cp + cfp*cd
ofpc = (1-cfp)*cd
ofpa = ad + ap
```

The cancel outcome withdrawable amounts
```
ocpe = ed + cp + ccp*cd
ocpc = (1-ccp)*cd
ocpa = ad
```

The unresolved outcome withdrawable amounts
```
oupe = ed + cp + eas*ap
oupc = cd + (1-eas)*ap
oupa = (1-anp)*ad
```

Deposit deadlines
```
odsa = bl + wd
odfb = odsa + dd
```
we remind that `bl` is the blockchain length at the moment of contract creation.

Success declaration deadlines
```
ossa = odfb
osdb = ossa + ed
```

Failure declaration deadlines
```
ofsa = osfb
offb = osfb + fd
```

This is all the values to define a contract preimage.

This example aimed for simplicity, it does not satisfy the constraint
```
ossa < ofsa < offb <= osfb
```
thus it might create an incentive for the Arbiter to declare failure even if Contractor has successfully delivered the desired outcome of the contract.


## References
[references]: #references

- [Mina protocol](https://minaprotocol.com/about)
- [Mina protocol - start building](https://docs.minaprotocol.com/)
- [Theoretical foundations of Mina - Mina book](https://o1-labs.github.io/proof-systems/)
- [Cypherpunk Manifesto](https://www.activism.net/cypherpunk/manifesto.html)
