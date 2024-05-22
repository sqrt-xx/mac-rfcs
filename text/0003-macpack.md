- Title: macpack
- Authors: [Marek Narozniak](mailto:marek.narozniak@protonmail.com)
- Start date: April 17th, 2024

---

## Summary
[summary]: #summary

In this RFC we propose a way to serialize circuit values for MAC! smart contract generator state into regular strings which are convenient to share between contract involved parties.

## Motivation
[motivation]: #motivation

Smart contract involving more than one party requires all the parties to require necessary data. This is especially the case for zkApps, or zero-knowledge applications as at least part of the information remains private and retained only by the involved parties.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

This RFC describes how to serialize and deserialize the `Preimage`, `Outcome` and `Participant` objects defined in RFC0002. This process is necessary for storage of those objects as well as for communication of those objects between participants of the contract.

## Technical-level explanation

There are multiple ways of representing the `Preimage`, `Outcome` and `Participant` objects defined in RFC0002. The proper way of representing depends on the purpose of serialization. For hashing we would use representation as an array of `Field`-typed values, this makes it convenient as an input for `Poseidon` hashing function and that is how a commitment is computed from a `Preimage`. The disadvantage of an array of fields representation is it is a one-way representaiton as it does not specify how many Fields the `CircuitStrings` occupy. For that reason we also introduce representation as bytes, this allows both serialization and deserialization making it two-way representation convenient for storage and communication. Lastly, we will introduce a JSON representation which has an advantage of being human-readable and machine-readable, making it a= convenient for debugging and development.

### General interface

Each of the `Preimage`, `Outcome` and `Participant` objects should implement following methods:
1. serialize
2. deserialize
3. toBytes
4. fromBytes
5. toJSON
6. fromJSON

The `serialize` and `deserialize` result in array of `Field`-typed values.

### Serialization / Deserialization to Fields

For the purpose of producing a Poseidon hash of the entire `Preimage` one must be able to turn the Preimage into an array of `Fiel`-typed values. This RFC will not go indepth into that process as Preimage as well as other data structures are `CircuitValue`-types. The `CircuitValue` type provided by `o1js` library is equipped with `toFields` method. As long as data structures are implemented according to the RFC0002 specification.

### Serialization / Deserialization to bytes

In order to be able to communicate the contract between involved parties it is necessary to serialize it into a bytestring. This makes it possible to save it as raw files as well as applying other forms of encoding such as `base58` or `base64`. For that purpose in this section we will define bytestring represenation of each of the data structures. Description consists of a table specifying the order of serialization of object attributes, first one on top of the table and last one on the bottom. Each raw also consists of bytes lenghth of the attribute.

#### Participant

The `Participant` data structure only holds `PublicKey` of the participant. It is represented by 40 bytes string.

| Value                 | Bytes |
|-----------------------|-------|
| `participant_address` | 40    |

#### Outcome

The `Outcome` consists of multiple attributes, one of them being of a variable lenght - `text`. For that reason in addition to text, the serialized bytestring also has the text length as a single byte. This limits the text length to occupy between `0` and `255` bytes.

| Value                | Bytes         |
|----------------------|---------------|
| `payment_employer`   | 8             |
| `payment_contractor` | 8             |
| `payment_arbiter`    | 8             |
| `start_after`        | 4             |
| `finish_before`      | 4             |
| `text_length`        | 1             |
| `text`               | `text_length` |

The resulting bytestring will be between `33` and `288` bytes. This variation depends on how long outcomes descriptions are.

#### Contract preimage

The contract preimage serialization is specification

| Value                  | Bytes                  |
|------------------------|------------------------|
| `contract_version`     | 1                      |
| `format_version`       | 1                      |
| `nonce`                | 8                      |
| `contract_text_length` | 1                      |
| `contract_text`        | `contract_text_length` |
| `address`              | 40                     |
| `employer`             | 40                     |
| `contractor`           | 40                     |
| `arbiter`              | 40                     |
| `deposited`            | 33 to 288              |
| `success`              | 33 to 288              |
| `failure`              | 33 to 288              |
| `cancel`               | 33 to 288              |

The resulting bytestring will be between `303` and `1579` bytes. This variation depends on how long contract and outcomes descriptions are.

### Packed form

Example of a packed macpack where `word_len=13` (length of a word) and `col_len=1` (number of columns).

```
BEGINMACPACK. 8XmkqY3jU6ghp bAXsoZjwCJx9G RYQnu3Dt6nBD5 w3NBQSJvQ5vjJ zdjmFgdbtF3ZD f9HHp8h7jFGce RPR8Z22K6YLvZ ZKiMAr7tBhPPG NmbftKrLkNgLT db4RSQhYwmDEk Ht7WBKbhUcYk1 rCjSoDa2mGUay FEqAgY4n9WhSG PyGbx3Jp6FHpk StEsGxYZWC475 5FW3gW8mSxwPR JuxWV2sXiSqNP SZTcs3yfeRqF1 r8nngKQgFyzgy WPC3B4pyAZ72g jb534aWWaydCi Z9wA8tWMA7xei NACKZWYYaNTqJ USbA5EJEV93kr ETDQdUNjo9kfD bYWBU9DanGjDe ZbJxwhQ2JntVB 4YmEyKvhUH5eu dfmnA4PSA3cwM eT6PCHGCjZJLA ZN1o7mFk3sif8 Ua77qkCMga7kV ScDc4zk256QHS Le7j2LVsc73kZ oe1GvJTyeRhRF rBA. ENDMACPACK.
```

#### Packing

A general algorithm of packing a `Preimage`

1. Serialize the preimage as concatenated bytes.
2. Apply base58 encoding of those bytes.
3. Split the resulting string into chunks of `word_len` in a way that all words but last one are of `word_len` and last word is at most `word_len` length.
4. Split words into `col_len` columns. Append a newline character at the end of every column.
5. Prepend the string with `BEGIN MACPACK ` header.
6. Append the string with `END MACPACK` footer.

Spaces and newlines in between header and footer are optional and subject to aesthetic preferences.

#### Unpacking

1. Extract the value contained between `BEGINMACPACK\n` header and `END MACPACK` footer.
2. Remove newline characters and spaces.
3. Apply the base58 decoding to obtain concatenated bytes.
4. Apply the `fromBytes` method of the `Preimage` class.

### JSON form

The contract is more visible as a JSON object.

#### Participant

A standard MINA address as stringified `PublicKey` object. For example: `B62qpDL2EcMDebv1GUEHVhx44HCgDZ3awy5zhMb34QcwXNRW27Qg9UN`.

#### Outcome

The entire `Preimage` as JSON object consists of keys named in exactly same way as `Outcome` object attributes.

```json
{
    "payment": {
        "employer": 0.01,
        "contractor": 0.01,
        "arbiter": 0.01
    },
    "start_after": 0,
    "finish_before": 1,
    "text": "justification of the payout policy and deadlines"
}
```

The JSON representation of the `Outcome` must be validated as follows:
1. Assert all values are non-null,
2. Assert `employer`, `contractor` and `arbiter` are deserializable as `PublicKey` object,
3. Assert that `start_after` and `finish_before` are integers,
4. Assert that `finish_before` is strictly greater than `start_after`,
5. Assert that `text` is a string.
6. Assert that `text` is of length between `0` and `255` characters.
7. Assert all balances changes are `number` typed.
8. Assert all balances changes are positive.

We require the balances changes to be positive because whether they represent deposists of withdrawal depends on the `Outcome`. More details on that in the RFC0002.

#### Preimage

The entire `Preimage` as JSON object consists of keys named in exactly same way as `Preimage` object attributes.

```json
{
    "contract_version": 1,
    "format_version": 1,
    "nonce": "24113853500102154836973742279689596782610842521222361814408365110578515602081",
    "contract_text": "Contractor should Tweet a feather emoji.",
    "address": "B62qpDL2EcMDebv1GUEHVhx44HCgDZ3awy5zhMb34QcwXNRW27Qg9UN",
    "employer": "B62qkThtJWtKXN56efqC4ZXqv5NMD3mvjoVg5GpLLHRv9b3z2uMrkN4",
    "contractor": "B62qoADwbR2iVHMz9SS8uY3WXdsxYW7aWRTfv9ET7BG5tpp82mTSNDV",
    "arbiter": "B62qjXdpDfDYM3tXJcUTcpzZeqWBZpM6NcEHsJhNjVcZdzWqdCEYb1J",
    "deposited": {
        "payment": {
            "employer": 0.0115,
            "contractor": 0.0015,
            "arbiter": 0.0001
        },
        "start_after": 6644,
        "finish_before": 7124,
        "text": ""
    },
    "success": {
        "payment": {
            "employer": 0.001,
            "contractor": 0.011,
            "arbiter": 0.0011
        },
        "start_after": 7124,
        "finish_before": 7604,
        "text": ""
    },
    "failure": {
        "payment": {
            "employer": 0.01125,
            "contractor": 0.00075,
            "arbiter": 0.0011
        },
        "start_after": 7604,
        "finish_before": 8084,
        "text": ""
    },
    "cancel": {
        "payment": {
            "employer": 0.0111,
            "contractor": 0.0009,
            "arbiter": 0.0011
        },
        "start_after": 7124,
        "finish_before": 7604,
        "text": ""
    }
}
```

The validation algorithm of the `Preimage` data structure consists of two steps. First we validate the JSON object before instantiating `Preimage`-typed object from it:
1. Assert all the values are non-null,
2. Assert `deposited`, `success`, `failure` and `cancel` values are valid `Outcome` JSON objects,
3. Assert `contract_text` is a string,
4. Assert `contract_text` is of length between `0` and `255` characters,
5. Assert `contract_version` and `format_version` are integers,
6. Assert `nonce` is a string,
7. Assert `nonce` is deserializable as `Field` object.
8. Assert `address`, `employer`, `contractor` and `arbiter` are deserializable as `PublicKey` object.
The second step is instantiating the `Preimage` object from that and applying the `Preimage` validation as described in the RFC0002. This ensures the net balances change add-up to same value.

## References
[references]: #references

- [CircuitValue specification](https://docs.minaprotocol.com/zkapps/o1js-reference/classes/Sign#extends)
