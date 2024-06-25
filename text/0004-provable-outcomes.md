- Title: Provable Outcomes
- Authors: [Marek Narozniak](mailto:marek.narozniak@protonmail.com)
- Start date: June 25th, 2024

---

## Summary
[summary]: #summary

In the first RFC we have introduced a zero-knowledge escrow protocol smart contract for MINA blockchain. Then we defined a protocol of communicating the contract state using serialized strings, we named this standard a `macpack`. The protocol described to this date was lacking certain features and suffered from limitations. It also was not taking advantage of all the zero-knowledge advantages. In this RFC we aim to define those limitations and extend the previous standard to fill those gaps.

## Motivation
[motivation]: #motivation

Here we may list some of the limitations of the previous standards to motivate the changes we are about to introduce.

### Fixed number of participants of the protocol

The original standard allowed only three participants - employer, contractor and the arbiter. This makes it significantly harder to create more advanced deals and in some cases could enforce usage of trusted setup.

### Fixed number of outcomes of the protocol

The original standard allowed only 5 outcomes - deposited, success, failure, cancel and unresolved. In practice, real-life deals may require more nuance, there could be many possible outcomes for an arbitrary deal involving various payout scenarios.

### Manual execution of the protocol

One of the biggest limitations of the original standard was manual execution mode. Contract was just a string and resolution depended whether the arbiter approved particular outcome. This is an advantage in a way as contracts become manually programmable - people can read contracts. Obvious limitation would be machine resolution that suddenly becomes significantly more difficult as it would require machine capable of understanding natural language or creation of contracts which would hold machine parseable data as contract strings, which would not be aesthetically pleasing.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

### zkPrograms

One of the most powerful features of o1js is something none of the MAC! standards were taking any advantage of until this RFC. The zkPrograms allow to prove an arbitrary computation. By compiling a zkProgram we get verification key and using this key we may check that computation. Using zkPrograms we will introduce new features to MAC, in particular unlimited number of outcomes, participants, arbitrary logic, automated execution mode, more privacy to the participants and more!

### Unlimited number of participants

Using merkle root we can store arbitrary number of participants. In this RFC we introduce effects, which are data structures holding participant and deposit or withdrawal data. Arbitrary number of effects can be stored in every outcome and arbitrary number of outcomes can be stored in the contract.

### Unlimited number of outcomes

Again using merkle root we can store arbitrary number of outcomes in the contract. Moreover, it is possible to partially share the contract and providing more privacy to the participants. They no longer need to have access to all the contract data.

### Types of execution

Unlike in the previous standard which allowed only manual execution mode, current standard allows contract resolution to be automated while remaining fully compatible with previous one.

#### Manual

The initial standard only allowed manual execution. Contract was specified as a string and approval of particular outcome depended on signature during contract method execution. This is still possible under current standard. The outcomes hold a zkProgram, which could be a proof of signature of particular string effectively implementing same logic. This asserts the current standard is fully compatible with the previous standard when it comes to manual execution mode.

#### Automated

The zkProgram embedded in the outcomes can be a proof of an arbitrary computation. This enhances the standard, it allows automatic resolution of the contracts.

## Technical-level explanation

### Provable outcomes

In this RFC we introduce an enhanced outcome data structure which incorporates zkProgram verification keys. This allows to embed custom logic of that particular outcome. In addition we introduce effects, which are data structures allowing unlimited number of participants being part of the deal.

#### Outcome effect

An effect specifies particular outcome effect from the perspective of a single participant. It specifies the participant address, whether it is a deposit or withdrawal and net MINA account balance change.

| Type        | Name          | Description                                                                                              |
|-------------|---------------|----------------------------------------------------------------------------------------------------------|
| `PublicKey` | `participant` | The MINA account of the participant.                                                                     |
| `Bool`      | `is_deposit`  | Indicates if balance change is from participant to the contract or from the contract to the participant. |
| `UInt64`    | `payment`     | The MINA amount balance change for this outcome.                                                         |

#### Provable outcome

A provable outcome as mentioned earlier, embeds a zkProgram to specify custom logic, all the effects using a merkle list and distribution proof. The details regarding each of those proofs are specified at the end of this document.

| Type             | Name                 | Description                                                                                                 |
|------------------|----------------------|-------------------------------------------------------------------------------------------------------------|
| `MerkleList`     | `effects`            | The merkle list root of all the effects associated to this outcome                                          |
| `VerficationKey` | `outcome_proof`      | Proof that needs to be provided for this outcome to become effective                                        |
| `VerficationKey` | `distribution_proof` | Proof of distribution, outcome is effective only if all the participants have received the `required_proof` |

### Contract preimage

#### Preimage

The preimage still holds random nonce to prevent preimage attacks, contract address and protocol version to associate it with contract implementation. Unlike in previous standards, unlimited number of outcomes is stored using merkle list and also a validity proof as zkProgram is provided. The details regarding each of those proofs are specified at the end of this document.

| Type             | Name                 | Description                                                                                                 |
|------------------|----------------------|-------------------------------------------------------------------------------------------------------------|
| `Field`          | `protocol_version`   | Fixed for `1` for this RFC, allows to modify the standard in the future                                     |
| `Field`          | `nonce`              | Random value preventing preimage attack                                                                     |
| `PublicKey`      | `address`            | Public key of the smart contract, deployer needs to have access to the corresponding private key            |
| `MerkleList`     | `outcomes`           | The merkle list root of all the effects associated to this outcome                                          |
| `VerficationKey` | `validity_proof`     | Proof that needs to be provided for this outcome to become effective                                        |

### Proofs

#### Outcome Proof

This is a zkProgram that proves particular outcome constraint is satisfied. Upon providing it depends if the smart contract will allow withdrawal or deposits to the contract according to associated outcome. It allows a lot of flexibility and embedded logic into the outcome. For compatibility with previous standard, this zkProgram would verify if the arbiter digital signature of that outcome does exist.

#### Distribution Proof

Outcome proof described right above is an evidence the contract logic associated to that outcome is satisfied. However, there is a possibility that parties involved in the contract do not have access to that proof. The distribution proof asserts that everyone who has a non-zero balance change for that outcome also has access to the outcome proof, ergo is able to deposit or withdraw.

#### Validity Proof

While designing a contract, various additional constraints might be required. Given that this standard allows unlimited outcomes and unlimited participants, it is not guaranteed all participants will gain access to all data. They might however demand some information, for instance a proof that other participants are not sanctioned list addresses. The validity proof is meant to prove certain properties of the contract. It is not relevant for the resolution itself but rather of the involved parties to satisfy their requirements.

## References
[references]: #references

- [zkProgram specification](https://docs.minaprotocol.com/zkapps/o1js-reference/type-aliases/ZkProgram)
