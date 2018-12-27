# Read Interface

## Overview
In order to participate in the data retrieval market, Indexing Nodes implement a low-level read interface to the indexed data in their store. The read interface not only provides the means of retrieving data from an Indexing Node, but it defines a contract that an Indexing Node is agreeing to uphold or else be slashed. This is enabled by attestations, which assert that a response was produced correctly, which may be verified on-chain.

### Calling Read Operations
Available read operations are defined by the respective interface of the index being read from. (See [Index Abstract Data Structures](#index-abstract-data-structures) and [Index Types](#index-types)).

While the read interfaces are described using a Typescript notation, all the interfaces are language agnostic, and are defined in terms of JSON types.

Calling these read operations is done via JSON RPC 2.0[[1]](#footnotes). (See the full JSON RPC API)

**TODO** Add link to full JSON RPC API when available

The method of interest here is `readIndex` which accepts the following parameters:
1. `Object`
 - `blockHash`: `String` - The hash of the Ethereum block as of which to read the data.
 - `subgraphID`: `String` - The ID of the subgraph to read from.
 - `index`: `Object` - The [IndexRecord](#indexes) of the index being read from.
 - `op`: `String` - The name of the read operation.
 - `params`: `[any]` - The parameters passed into the called read operation.
1. `Object` (optional) - A conditional micropayment (see Payment Channels).

**TODO** Add link to Payment Chanel Architecture when ready.

The `readIndex` method returns the following:
1. `Object`
 - `data`: `any` - The data retrieved by the read operation.
 - `attestation`: Object - An attestation that `data` is a correct response for the given read operation (see [Attestations](#attestations)).

###### Example - Entity exists
```js
// request
{
  "method": "readIndex",
  "params": [
    {
      "blockHash": "xbf133b670857b983fc1b8f08759bc860378179042a0dba30b30e26d6f7f919d1",
      "subgraphID": "QmTeW79w7QQ6Npa3b1d5tANreCDxF2iDaAPsDvW6KtLmfB",
      "index": {
        "indexType": "kv"
      },
      "op": "get"
      "params": ["User:1"]
    }
  ],
  "jsonrpc": "2.0"
}
// response
{
  "data": {
    "firstName": "Vitalik",
    "lastName": "Buterin",
  },
  // TODO: Provide more realistic attestations
  "attestation": 0x0122340
}
```

###### Example - Entity doesn't exist
```js
// request
{
  "method": "readIndex",
  "params": [
    {
      "blockHash": "xbf133b670857b983fc1b8f08759bc860378179042a0dba30b30e26d6f7f919d1",
      "index": {
        "indexType": "kv"
      },
      "op": "get"
      "params": ["User:1"]
    }
  ],
  "jsonrpc": "2.0"
}
// response
{
  "data": null,
  // TODO: Provide more realistic attestations
  "attestation": 0x0122340
}
```

## Attestations
Attestations are ECDSA (Elliptic-Curve Digital Signature Algorithm) signatures which assert that a given response is true for a given read request.

An Attestation message has the following structure:

| Field Name  | Field Type | Description |
| ----------- | ---------- | ----------- |
|
| messageCID | bytes    | The content ID of the mess. |
| gasUsed     | uint256    | The gas used to process the read operation. |
| bytes       | uint256    | The size of the response data in bytes. |
| responseCID | bytes   | The content ID of the response. |
| v | uint256 | The ECDSA recovery ID . |
| r | bytes32 | The ECDSA signature r. |
| s | bytes32 | The ECDSA signature v. |

### Encoding
The attestation message is encoded and signed according to the [EIP-712 specification](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md) for hashing and signing typed structured data. See the [Smart Contract Architecture]() for more info on how the protocol utilizes said specification.

**TODO** Add link to smart contract architecture section, when available.

### Content IDs
`messageCID` and `responseCID` are both produced according to the [IPLD CID V1 specification](https://github.com/ipld/cid#cidv1).

Content IDs must use the canonical [CBOR encoding](https://tools.ietf.org/html/rfc7049#section-3.9), and SHA-256 multi-hash.

In producing the `messageCID`, the optional `id` field from the JSON-RPC 2.0 specification should be omitted, as well as the optional conditional micropayment in the `readIndex` params list.

## Indexes
All read operations require that the caller specify an index. Index data structures efficiently organize the data to support different read access patterns. All indexes are covering indexes, meaning they store values directly rather than pointers.

Indexes may include the entire dataset, or they may cover only a subset of the overall dataset. This is useful for enabling sharding, where different Indexing Nodes may store different subsets of the dataset, to reduce the storage requirements for a single Indexing Node or enable better read performance.

Indexes are defined by an `IndexRecord` which has the following shape:
```typescript
interface IndexRecord {
  // The type of the index (required).
  indexType: String;
  // Parameters specific to the type of index specified (optional).
  indexParams?: {
    // An identifier for the partition of data that the index will cover. Available partitions determined by the database model of the index (optional).
    partition?: String;
    // Parameters which are specific to the type of index specified (optional).
    params?: [String];
  };
}
```
###### Example Index Records

Given a dataset with the following data model...
```graphql
interface EthereumAccount {
  id: ID!
  address: String!
}

type Contract implements EthereumAccount {
  id: ID!
  address: String!
}

type User implements EthereumAccount {
  id: ID!
  address: String!
  name: FullName
}

type FullName {
  first: String!
  last: String!
}
```
... then the following would be valid index names for that dataset:

| Index Name | Description|
| ---------- | ---------- |
| `{ indexType: "kv" }`       | A basic key-value index supporting constant-time lookup of all entities in dataset. |
| `{ indexType: "kv", partition: "User" }`  | A basic key-value index supporting constant-time lookup of `User` entities. |
| `{ indexType: "kv_sortById" }` | A sorted key-value index supporting iteration through all entities, sorted by ID. |
| `{ indexType: "kv_sortByAttribute", indexParams: { partition: "EthereumAccount", params: ["address"] }}` |  A sorted key-value index supporting iteration through all entities implementing the `EthereumAccount` interface, sorted by the `address` field. |
| `kv:User:sortByField:name.first` `{ indexType: "kv_sortByAttribute", indexParams: { partition: "User", params: ["name.first", "name.last"]}}` | A sorted key-value index supporting iteration through all `User` entities, first sorted by the nested field `name.first`, then by the nested field `name.last` (i.e. a compound index). |

### Index Abstract Data Structures

All concrete index types implement an indexing abstract data structure, which specify the interface, semantics, and gas costs for read operations against that index.

The concrete types (i.e. `K` and `V` shown below), as well as the implicit comparator function to determine sort order, are specified by each concrete index type.

#### Dictionary

##### Operations
| Op  | Signature | Description | Gas Cost |
| --- | --------- | ----------- | -------- |
| get | `(key: K) => V` | Retrieves a value by its key. | `opCostDictionaryGet` (set via governance) |

#### Search Tree

##### Operations
| Op  | Signature           | Description  | Gas Cost |
| --- | ------------------- | ------------ | -------- |
| find | `(predicate: FilterPredicate, options?: { gte?: K, lt?: K } ) => V` | Retrieves the first value for which the `FilterPredicate` returns true, searching in ascending order. If specified, only take values whose sort keys are between the range parameters `gte` (inclusive) and `lt` (exclusive). | `(opCostSearchTreeStep opCostFilterPredicate) * N` where `N` is the amount of iterations taken to find the value, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided. |
| findLast | `(predicate: FilterPredicate, options?: { gt?: K, lte?: K } ) => V` | Retrieves the first value for which the `FilterPredicate` returns true, searching in descending order. If specified, only take values whose sort keys are between the range parameters `gt` (exclusive) and `lte` (inclusive). |  `(opCostSearchTreeStep opCostFilterPredicate) * N` where `N` is the amount of iterations taken to find the value, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided. |
| get | `(key: K) => V` | Retrieves a value by its sort key. If multiple values share same sort key, will retrieve first value inserted with sort key. | `opCostSearchTreeGetPerH * H` where `H` is the height of a binary search tree and `opCostSearchTreeGetPerH` is set via governance. |
| take | `(count: Number, options?: { skip?: Number, gte?: K, lt?: K}) => [V]` | Retrieves the first N values, specified by `count`, from the index in ascending order, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gte` (inclusive) and `lt` (exclusive).| `opCostSearchTreeStep * N` where `N` is the amount of iterations taken including skipped values. |
| takeUntil | `(predicate: FilterPredicate, options?:{ skip?: Number, gte?: K, lt?: K}) => [V]` | Retrieves values from the index in ascending order, until `FilterPredicate` returns false, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gte` (inclusive) and `lt` (exclusive). | `(opCostSearchTreeStep + opCostFilterPredicate) * N + opCostSearchTreeStep * S` where `N` is the amount of iterations taken not including skipped values, and `S` is the number of values skipped over, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided.  |
| takeWhile | `(predicate: FilterPredicate, options?:{ skip?: Number, gte?: K, lt?: K}) => [V]` | Retrieves values from the index in ascending order, while `FilterPredicate` returns true, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gte` (inclusive) and `lt` (exclusive). | `(opCostSearchTreeStep + opCostFilterPredicate) * N + opCostSearchTreeStep * S` where `N` is the amount of iterations taken not including skipped values, and `S` is the number of values skipped over, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided. |
| takeLast | `(count: Number, options?: { skip?: Number, gt?: K, lte?: K })` | Retrieves the last N values, specified by `count`, from the index in descending order, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gt` (exclusive) and `lte` (inclusive). | `opCostSearchTreeStep * N` where `N` is the amount of iterations taken including skipped values.  |
| takeLastUntil | `(predicate: FilterPredicate, options?:{ skip?: Number, gt?: K, lte?: K}) => [V]` | Retrieves values from the index in descending order, until `FilterPredicate` returns false, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gt` (exclusive) and `lte` (inclusive). | `(opCostSearchTreeStep + opCostFilterPredicate) * N + opCostSearchTreeStep * S` where `N` is the amount of iterations taken not including skipped values, and `S` is the number of values skipped over, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided. |
| takeLastWhile | `(predicate: FilterPredicate, options?:{ skip?: Number, gt?: K, lte?: K}) => [V]` | Retrieves values from the index in descending order, while `FilterPredicate` returns true, optionally skipping the number of values specified by `skip`. If specified, only take values whose sort keys are between the range parameters `gt` (exclusive) and `lte` (inclusive). | `(opCostSearchTreeStep + opCostFilterPredicate) * N + opCostSearchTreeStep * S` where `N` is the amount of iterations taken not including skipped values, and `S` is the number of values skipped over, `opCostSearchTreeStep` is set via governance, and `opCostFilterPredicate` is calculated for the specific filter predicate provided. |

### Filter Predicates
Filter predicates allow for declaratively asserting whether a value meets certain criteria. Filter predicates are expressed as objects which can be passed into several low-level index read operations such as `takeWhile` and `find`.

#### Structure

Filter predicates are expressed through a simple DSL:
```typescript
type FilterPredicate = FilterPredicateLeaf | FilterPredicateAnd | FilterPredicateOr

interface FilterPredicateOr {
  or: [FilterPredicate];
}

interface FilterPredicateAnd {
  and: [FilterPredicate];
}

type FilterPredicateLeaf = StringFilter | NumberFilter | BooleanFilter

interface BaseFilter {
  // The field the predicate will be applied to. Nested fields may be
  // specified, by concatenating field names with a "."
  // If no field specified, the predicate will be applied to the value. This
  // is only supported if the value is a primitive type.
  field?: String;
}

// If multiple filter clauses supplied will be treated as logical AND
interface StringFilter extends BaseFilter {
  equals?: String;
  notEquals?: String;
  // Contains string
  contains?: String;
  // Does not contain string
  notContains?: String;
  startsWith?: String;
  notStartsWith?: String;
  endsWith?: String;
  notEndsWith?: String;
  // Less than
  lt?: String;
  // Less than or equal to
  lte?: String;
  // Greater than
  gt?: String;
  // Greater than or equal to
  gte?: String;
  // Contained in list
  in?: [String];
  // Not contained in list
  notIn?: [String];
}


// If multiple filter clauses supplied will be treated as logical AND
interface NumberFilter extends BaseFilter {
  equals?: Number;
  notEquals?: Number;
  // Less than
  lt?: Number;
  // Less than or equal to
  lte?: Number;
  // Greater than
  gt?: Number;
  // Greater than or equal to
  gte?: Number;
  // Contained in list
  in?: [Number];
  // Not contained in list
  notIn?: [Number];
}

// If multiple filter clauses supplied will be treated as logical AND
interface BooleanFilter extends BaseFilter {
  equals?: Boolean;
  notEquals?: Boolean;
}
```

###### Example - Simple Value Filter Predicate
```js
{
  equals: 12
}
```

###### Example - Object Filter Predicate
```js
{
  field: "fullName",
  contains: "Vitalik"
}
```

###### Example - Filter Predicate with Boolean Operators and Nested Fields
```js
{
 and: [
   {
     field: "name.first",
     equals: "Vitalik"
   },
   {
     field: "name.last",
     equals: "Buterin"
   }
 ]
}
```

#### Gas Cost
The clauses in the filter predicate DSL can be grouped into several buckets of operation types, which share equivalent gas cost calculations.

| Operation Type | Description | Gas Cost |
| --------- | ----------- | -------- |
| Number Comparison | Includes `lt`, `lte`, `gt`, `gte`, `equals` and `notEquals` clauses on Number types. | `opCostBitCompare * N ` where `N` is the amount of bits compared to complete the operation, and `opCostBitCompare` is set via governance. |
| String Comparison | Includes `lt`, `lte`, `gt`, `gte`, `startsWith`, `notStartsWith`, `endsWith`, `notEndsWith`, `equals` and `notEquals` clauses on String types. | `opCostCharCompare * N ` where `N` is the amount of characters compared to complete the operation, and `opCostCharCompare` is set via governance. |
| Bit Comparison | Includes `equals` and `notEquals` clauses on Boolean types. Also used for combining two filter predicate clauses via boolean operators `or` and `and` (including the implicit `and` described above). | `opCostBitCompare` where `opCostBitCompare` is set via governance. |
| String Match | Used for `contains` and `notContains` clauses on String types | `opCostStringSearch * (M + N)` where `N` is the amount of characters in the pattern being matched, and `M` is the amount of characters in the string being searched. `opCostStringSearch` is set via governance. |


### Database Models
The semantics of reading from an Indexing Node are determined by the database model which the index being read from implements, such as [key-value (KV)](https://en.wikipedia.org/wiki/Key-value_database), [entity-attribute-value (EAV)](https://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model) and the [relational model](https://en.wikipedia.org/wiki/Relational_model). Index types are prefixed with a short label indicating the database model the index implements:
- `kv_` - The index is based on a key-value database model.
- `rdb_` -  The index is based on a relational database model.
- `eav_` -  The index is based on an entity-attribute-value database model.

The database model also defines the available partitions and index types for use in read operations.

In the v1 protocol, we support only a KV database model.

#### Key-Value Entity DB Model
In the protocol's key-value database model, entities are stored as key-value pairs, where the key is a concatenation of the entity type and the entity ID, and the value is an entity object. Indexes which implement this database model start with `kv` in their `indexType`.

###### Example entities stored as key-value pairs
| Key | Value |
| --- | ----- |
| `user:1` | `{ user: "Alice", age: 17 }` |
| `user:2` | `{ user: "Bob", age: 47 }` |

##### Partitions
Partitions define the subset of the data which is covered by the index.

Possible values of `<partition>` in index name:
 - `null` - Includes all entities in the dataset. Default partition, may be ommitted.
 - `"<entityType>"` - Includes entities of the type specified by `entityType` (entity type is case-sensitive).
 - `"<interface>"` - Includes entities which implement the provided interface (interface name is case-sensitive).

### Index Types

#### Entity Dictionary
The entity dictionary supports simple key-value lookups of entities by their entity type and ID, in constant time.
- database model - Key-value
- type - `Dictionary<String, Object>` where `Object` is an entity which conforms to its type as defined in the schema of the dataset.
- `indexType` - `kv`

#### ID Ordered Entity Index
This index supports iterating through entities, ordered by their entity type and ID.
- database model - Key-value
- type - `SearchTree<String, Object>` where `Object` is an entity which conforms to its type as defined in the schema of the dataset.
- `indexType` - `kv_sortById`

#### Attribute Ordered Entity Index
This index supports iterating through entities, ordered by possibly nested attribute values. Supports compound indexes, where an entity is sorted first by one attribute, then by another.
- database model - Key-value
- type - `SearchTree<Object, Object>` where `Object` is an entity which conforms to its type as defined in the schema of the dataset.
- `indexType` - `kv_sortByAttribute`
- `indexParams.params`
   - primaryAttribute - The first attribute to sort by, expressed as a `String`, using `.` to indicate nested attributes (i.e. `"name.first"`)
   - secondaryAttribute -  The second attribute to sort by, expressed as a `String`, using `.` to indicate nested attributes (i.e. `"name.first"`)

## Footnotes
- [1] https://www.jsonrpc.org/specification
- [2] https://github.com/multiformats/multicodec