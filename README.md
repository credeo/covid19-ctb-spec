# COVID-19 Contact Tracing Backend Specification

> This specification is featured in [COVID-19 Tracker Apps and Apple Google Contact Tracing API - The need for interoperability and standardisation](https://medium.com/@kristijan.cvetkovic/covid-19-tracker-apps-and-apple-google-contact-tracing-api-ed9983ea049a) and published as early draft version and will be subject to Subject to modification and extension. Please consider it as invitiation to conversation

## Design Goals

The main goal of this specification is to assist COVID-19 Tracking apps which are using the recently published Apple/Google Contact Tracing API in answering key architectual such as

- How does an app pull published diagnosis keys?
- Who is allowed to pull the published diagnosis keys?
- When and how often shall apps pull diagnosis keys?
- How does an app push its diagnosis keys in case of of a positive diagnosis?
- How is the push of diagnosis keys secured to prevent pranksters and fraud attempts?
- Where and how are the diagnosis keys stored?
- What data needs to be stored in the cloud beside the diagnosis keys?
- How can apps of different developers share diagnosis keys?

## Actors

The main actors in the context of this specification are:

| Actor | Description |
| --- | --- |
| App Developer | Developer of a COVID-19 Tracking app |
| User | End user who install their preferred COVID-19 Tracking app on their device |
| Health Authority | official institution, which conducts COVID-19 tests and informs users about a positive diagnosis |
| Contact Tracing Backend Provider | Developer or hoster of a Contact Tracing Backend Instance |
| Contact Tracing Backend Authority | Owner of the single Contact Tracing Backend Registry |
| Data analyst | Data analyst, who use published diagnosis keys for research |



## Components

The main components in the context of this specification are:

| Actor | Description |
| --- | --- |
| App | COVID-19 Tracker app developed and published by app developers |
| OS | Android and iOS devices with enabled Contact Tracing API |
| Contact Tracing Backend | instance of a running Contact Tracing Backend according to this specification |
| Contact Tracing Backend Registry | single instance of Contact Tracing Backend Registry |

Subject of standardisation are the components ```Contact Tracing Backend``` and ```Contact Tracing Backend Registry```

## Interfaces

The main interfaces are

| Interface | Description | provided by | used by |
| --- | --- | --- | --- |
| Push | Push diagnosis key in case of positive diagnosis | ```Contact Tracing Backend``` | ```App``` | 
| Pull | Pull diagnosis to match in Contact Tracing API | ```Contact Tracing Backend``` | ```App``` | 
| Sync | Sync batch data | ```Contact Tracing Backend``` | ```Contact Tracing Backend``` | 
| On-Boarding | authorise Contact Tracing providers to accept pushed diagnosis keys based on signed one time token of Health Authorities | ```Contact Tracing Backend``` | ```Health Authorities``` | 
| Registry | discover and register new ```Contact Tracing Backend``` instances | ```Contact Tracing Backend Registry``` | ```Contact Tracing Backend``` | 

### Push-Interface

#### Technology

| Key | Description |
| --- | --- |
| application protocol | HTTP |
| encryption | SSL 1.2 |
| authorization | None |
| interface | GraphQL |
| schema | GraphQL SDL |
| additional HTTP headers | None |

#### Schema

```javascript
type Mutation {
  push(keys:[String], token:TokenInput!, geohash:String, daynumber: Int) : Boolean
}

input TokenInput {
  token: String,
  signature: String,
  haId: ID!
}
```

### Pull-Interface

#### Technology

| Key | Description |
| --- | --- |
| application protocol | HTTP |
| encryption | SSL 1.2 |
| authorization | None |
| interface | GraphQL |
| schema | GraphQL SDL |
| additional HTTP headers | None |

#### Schema

```javascript
type Query {
  keys(geohash: String!, daynumber:Int, first:Int, after:ID) : KeysResult
  provider: Provider
}

type KeysResult {
  keys:[Key]
  cursor:ID!
}

type Key {
  id: ID!
  hex: String!
  geohash: String!
  daynumber: Int
  signature: String  
}

type Provider {
  id: ID!
  keyChain: [PublicKey]
  web: String
}

type PublicKey {
  id: ID!
  pem: String
  validFrom: Int
  replacedBy: PublicKey
}
```

### Sync-Interface

#### Technology

| Key | Description |
| --- | --- |
| application protocol | HTTP |
| encryption | SSL 1.2 |
| authorization | None |
| interface | REST |
| schema | ProtoBuf |
| additional headers | ETag, If-None-Match |

#### Schema

``` java
syntax = "proto2";
package covid19.ctb.sync;

message Signature {
  required String providerId = 1; // provider ID as issued by CTB Registry
  required uint32 daynumber = 2; // Day number
  required uint32 signature = 3; // Signature for Sync message of day
}

message Sync {
  required uint32 timestamp = 1; // Date of creation as seconds since 1.1.1970
  required uint64 etag = 2; // Etag
  required String providerId = 3; // provider ID as issued by CTB Registry
  repeated Key keys = 4; // list of keys
}

message Key {
  required String geohash = 1; // Geohash reduced to 3 characters
  required uint32 daynumber = 2; // Day number according to https://www.blog.google/documents/56/Contact_Tracing_-_Cryptography_Specification.pdf
  required uint64 key = 3; // Daily Tracing Key aka Diagnosis Key
}
```

#### Endpoints

| Endpoint | Method | Returns | Request Headers | Response Headers | Status-Codes
| --- | --- | --- | --- | --- | --- | 
| ```/sync/:daynumber``` | GET | ```covid19.ctb.sync.Sync``` | If-None-Match  | ETag | 200, 404, 303 | 
| ```/sync/:daynumber``` | GET | ```covid19.ctb.sync.Signature``` | ./.  | ./. | 200, 404, 303 | 


### HA-Interface

#### Technology

| Key | Description |
| --- | --- |
| application protocol | HTTP |
| encryption | SSL 1.2 |
| authorization | OAUTH2, Bearer |
| interface | GraphQL |
| schema | GraphQL SDL |
| additional HTTP headers | none |

#### Schema

````javascript
type Mutation {
  changePublicKey(haId: ID!, pem:String!): [PublicKey]
}

type Query {
  ha(id:ID!): HA
  has: [HA]
}

type PublicKey {
  id: ID!
  pem: String
  validFrom: Int
  replacedBy: PublicKey
}

type HA {
  id: ID!
  web: String
  keyChain:[PublicKey]
}
````


### Registry-Interface

#### Technology

| Key | Description |
| --- | --- |
| application protocol | HTTP |
| encryption | SSL 1.2 |
| authorization | None |
| interface | GraphQL |
| schema | GraphQL SDL |
| additional HTTP headers | none |

#### Schema

````javascript
type Mutation {
  register(endpoint: String!, pem:String!, contact:ContactInput!): Provider
}

type ContactInput {
  email: String
  web: String
  phone: String
  street: String
  zip: String
  city: String
  country: String
  name: String
}

type Query {
  providers:[Provider]
}

type Provider {
  id: ID!
  status: Status
  contact: Contact
  keyChain: [PublicKey]
  endpoint: String
}

type Contact {
  email: String
  web: String
  phone: String
  street: String
  zip: String
  city: String
  country: String
  name: String
}

type PublicKey {
  id: ID!
  pem: String
  validFrom: Int
  replacedBy: PublicKey
}

enum Status {
  NEW
  INAPPROVAL
  ENROLLED
  CANCELED
  SUSPENDED
}
````

## Open topics

* Authorization exchange for HA
* Approval of registries
* Reference implementations of interfaces
* Reference architecture
* Key Interactions
