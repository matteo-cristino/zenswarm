![Zenswarm](docs/zenswarm.svg)

# Zenswarm
**Zenroom based Swarm of Oracles**

After announcing to a DID controller, the Oracles provide a number of services, including:
 * Cryptographic signature
 * Blockchain notarization
 * Smart Contract execution

<h1 align="center">
Zenswarm
  </br>
  <sub>Zenroom based Swarm of Oracles</sub>
</h1>

<p align="center">
  <a href="https://dyne.org">
    <img src="https://img.shields.io/badge/%3C%2F%3E%20with%20%E2%9D%A4%20by-Dyne.org-blue.svg" alt="Dyne.org">
  </a>
</p>

<br><br>


<details id="toc">
 <summary><strong>🚩 Table of Contents</strong> (click to expand)</summary>

* [Install](#-install)
* [Monitoring](#-monitoring)
* [Diagrams](#-oracles-diagrams)
* [APs](#-apis)
* [License](#-license)
</details>

## 💾 Install

### Requirements
*  Linux based machine 
* the hostname on the host machine, must be reachable from the internet (can be an IP), the oracle use the hostname to announce their identities to the Controller
* Ports between the 20000 and 30000 must be open on the host machine
* On the host machine, add to /root/.ssh/authorized_keys the pubkey you will use to deploy from your workstation: ansible will use this pubkey to obtain root privileges on the host (not required if using the Linode scripts)
* Open an ssh port for ansible, can be configured in hosts.toml (default: *ansible_port=22254*) (not required if using the Linode scripts)
* The Oracles' ansible installs an SSL certificate using Letsencrypt and the oracle currently comunicates via https. This can generate issues on machines behind a proxy (e.g. a virtual machine).  
* (DID-Controller only) redis running on port 6379

### How to install

* *git clone https://github.com/dyne/zenswarm/*
* edit hosts.toml to set: 
  * address of target machine for deployment (default: zenswarm.zenroom.org ) 
  * port that ansible will use to connect to the host (default: *ansible_port=22254*)
  * amount of oracles to be deployed on that machine (default: *nodes=3*)
* edit *./ansible/init-instances.yml* in case you want to use a different DID Controller to perform the announce and deannounce (defaults: **announce_url: "https://did.dyne.org/api/W3C-DID-controller-create-oracle-DID.chain"** and **deannounce_url: "https://did.dyne.org/api/W3C-DID-controller-remove-oracle"**)  
* edit *./ansible/subscription.csv* to define which Oracle will notarize from which L1 to which L0
* **make setup-tls** to install the SSL certificate using Letsecncrypt
* (If deploying on Linode): execute **make one-up IMAGE=(name of the linode image)** 
* (If NOT deploying on Linode): execute **make install**

### How to run
After installation, run:
 * **make init** to generate the secret keys of the Oracle(s)
 * **make start** to generate the identity and trigger the announce of the Oracle(s)
 * **make kill** performs a graceful shutdown that deannounces the Oracle(s), unregistering the Oracle(s) from the DID Controller

### Announce and Deannounce process

Once an Oracle is deployed, each Oracles tries to *announce* to the W3C-DID controller. In this stage, the Oracle will communicate its pubkeys, its version and some metadata. The W3C-DID Controller will register a DID per each Oracle, and store the DID-document on a database and blockchain. 

Upon graceful shutdown, done via *make kill* from your workstation or *pm2 delete [instansce-name]* on the host machine, the Oracle will *deannounce* itself, which will prompt the W3C-DID Controller to remove the Oracle from the database.

Specs about the DID implementation is in [Dyne.org's W3C-DID](https://github.com/dyne/W3C-DID).


## 📉 Monitoring

* On the machines where the oracles are deployed, use **pm2 list** to see how many instances of restroom_mw are running.
* A GUI-based monitoring service for the Oracle is the [Zenswarm-Dashboard](https://github.com/dyne/Zenswarm-Dashboard). The GUI retrieves a list of the active Oracles from the W3C-DID controller




## 🖧 Oracles diagrams

Below a list of the main Oracle flows involving:
 * Controller provisioning: provisioning of the DID Controller producing signed DID Document for the Oracles
 * Oracle Provisioning: key issuance, DID document creation
 * Oracle consensus based query
 * Oracle update

## Controller creation

```mermaid
sequenceDiagram
autonumber
  participant A as Admin
  participant C as Cloud
  participant I as Controller
  A->A: Admin keygen (aSK + aPK)
  A->>C: Setup a Cloud provider
  C->>I: Issuer Install
  C->>A: Grant Issuer setup access
  A->>I: Issuer init + aPK
  I->I: Keygen (iSK + iPK)
  I->I: DID Document creation
  I->>A: DID publishing (containing iPK)
```

1. Admin is the control terminal and generates a new keypair (aSK + aPK)
1. Admin sets up a Cloud provider (one or more) can be remote or on-premises
1. Issuer is created by the Cloud provider and installed with a signed OS
1. Cloud grants to Admin setup access to the Issuer
1. Admin initialized the Issuer machine with signed scripts and the Admin public key
1. Controller generates an issuer keypair (iSK + iPK)
1. Controller generates DID Document
1. Controller shares its DID containing (iPK)


## Oracle creation

```mermaid
sequenceDiagram
autonumber
  participant A as Admin
  participant C as Cloud
  participant V as O1..O2..On
  participant I as Controller
  participant B as Blockchain

  A->>C: Deploy Oracle request
  C->>A: Grant Oracle setup access
  A->>C: Setup SSL
  C->>V: Oracle provisioning (oSK + oPK)
  V->>I: announce of Oracle
  I->I: Ephemeral keygen (eSK + ePK) or sidechannel
  I->>V: iSK signed request + eSK
  V->>I: eSK signed answer: IP + oPK
  I->I: ePK verify and store Oracle IP + oPK
  I->I: Create Oracle DID 
  I->>B: Notarize Oracle DID 
  B->>I: Return txId with DID
  I->I: Store txId as DID
```

1. Admin orders the creation of Oracle to the Cloud provider
1. Cloud provider grants the Admin setup access to deploy the Oracle (IP + SSH)
1. Admin set up SSL certificates in Cloud
1. Cloud provider creates the Oracle on the host machine, and generates keys (oSK, oPK)
1. Oracle announces its identity to the Controller (URL + oPK)
1. Controller generates ephemeral secret key (eSK) (this step can moved to a side channel)
1. Controller sends eSK signed with its private key (iSK), its public key (iPK) is known 
1. Cloud verify signed eSK and signs it with its private key (oSK)
1. Controller verifies eSK signed from Oracle using Oracle's publick key (oPK)
1. Controller creates DID document for Oracle (containins DID of future txId), DID id is oPK
1. Controller notarizes DID document of Oracle on Blockchain
1. Blockchain returns txId storing DID document Oracle 
1. Controller stores txId as DID document (txId DID is contained in Oracle's DID Document)


## Oracle distributed query 
```mermaid
sequenceDiagram
autonumber
  participant U as User
  participant I as Controller
  participant O as Oracle
  participant V as O1..O2..On
  participant B as Blockchain
  
  
  U->>I: Ask active Oracles
  I->>U: Returns list of Oracles
  U->>O: Asks Oracle distributed query
  O->>+V: Oracles queries Swarm-of-Oracles
  V->V: Execute queries 
  V->>-O: Return ECDSA signed result or error
  O->O: Consensus on results or errors
  O->>U: return ECDSA signed collective result or error
  U->>B: (optional) queries Oracles' txId containing W3C-DID
  B->>U: (optional) returns txId containing W3C-DID
```

1. User queries to Oracle Controller requesting list of registered Oracles in the Swarm of Oracles (SoO)(or an event or a time trigger)
1. Controller returns list of registered SoO
1. User makes to request to one Orace to perform a distributed query
1. Oracles queries Swarm-of-Oracles (SoO) chosen from a trusted random
1. SoO perform POST to endpoint
1. SoO returned ECDSA signed output to Oracle
1. Oracle verifies output and signatures of SoO
1. Oracle returns ECDSA signed aggregated output of SoO 
1. (Optional) User queries blockchain to read notarized DID Documents of Oracle(s)
1. (Optional) Blockchain returns DID Document notarized in tx

## Oracle update
```mermaid
sequenceDiagram
autonumber
  participant A as Admin
  participant I as Controller
  participant V as O1..O2..On
  
  A->>I: aSK signed update ZIP
  I->I: aPK verify update ZIP
  I->>V: iSK signed update ZIP 
  V->V: iPK verify and install ZIP
```

1. Admin signs and uploads a ZIP with updated scripts
1. Issuer verifies the ZIP is signed by the Admin
1. Issuer signs and uploads the update ZIP to all VM
1. VM verifies the ZIP is signed by the Issuer and installs the scripts






## 🌐 APIs 

Below a list of the APIs available on an Oracle

----
### Get Identity

  Returns json data containing the Oracle's identity: 
 * Identity
  * uid: contains URL and HTTPS port 
  * baseUrl: the URL 
  * HTTPS port
  * ECDSA, EDDSA, Schnorr, Dilithium, public keys
  * Ethereum and Bitcoin address
  * List of available APIs
  * Country
  * State (region)
  * subscriptions: list of blockchain(s) the Oracles has a websocket subscription to
  * L0: blockchain the Oracle is notarizing onto
  

* **URL**

  /api/zenswarm-oracle-get-identity

* **Method:**

  `GET` or `POST` 
  
* **Data Params**

  None

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "identity": {
    "API": [
      "/api/zenswarm-oracle-announce",
      "/api/ethereum-to-ethereum-notarization.chain",
      "/api/zenswarm-oracle-get-identity",
      "/api/zenswarm-oracle-http-post",
      "/api/zenswarm-oracle-key-issuance.chain",
      "/api/zenswarm-oracle-ping",
      "/api/sawroom-to-ethereum-notarization.chain",
      "/api/zenswarm-oracle-get-timestamp",
      "/api/zenswarm-oracle-update",
      "/api/zenswarm-oracle-get-signed-timestamp",
      "/api/zenswarm-oracle-sign-dilithium",
      "/api/zenswarm-oracle-sign-ecdsa",
      "/api/zenswarm-oracle-sign-eddsa"
    ],
    "Country": "FR",
    "L0": "ethereum",
    "State": "NONE",
    "baseUrl": "https://swarm1.dyne.org",
    "bitcoin_address": "bc1q2eje20v242z9walf35vv78e9t364qat7gptl33",
    "description": "restroom-mw",
    "dilithium_public_key": "WR6I8Y2/D7pN9wUypkNqoG1ivSozcezyLTovGDBpJYxn3PIO7m2sDJYHx8cr9tJVpCcpdSdregQEWG7iHdgyMwXic60JajTQfifyZCHOStJRYwEXKBA36fMnu/0rK6kjGjImXCGIfGO8gDdvVtWe1x2HmN4XBoLsE+J7Qbwqir+qub3AssQKg+xyq/4DYRGyrqG0kiablY5RaUR66eaVVORgTR2eHza58nf/iCDUkol3Y//yM8CS7BS4bPh+ARd+3Dk2P++XLXzi8kV4Vrj5S4nIVv7D5AbPBUYQc6uTKyw7ybhON2x21MHGWfF0s83J4P1h/yMtObnYg9DxJQAHGZrpc6RvH2fND2hUB0PobdrzRQLz79jm+Pn1oFbhq7LsQep6CFsO2iqQquab3Qes41W16V3gVd2aNtzhbTqaTRr+hU7/T8bYf0dv1r0jZuZvaMnfpbiPpesQ0izSh7lO1l7TrMHNZbVPU/vB9P53stGuqcEXrsmz6W/ExoVCusj2L6DCMo3q2y42XRT2tA3JXsjMrFJKzc2DI1UdOkOuP7jzuc9WxlwFSTHvIdZDrG8SuWiRZYGda6ZzQBvCPgkqpaDRmsZvrC4IGNFeuAcedNZWMI6W+fvw+csOToLOwRUqUmJqhrjjc8dZ0EfyKM34PMp1z8TKcPh/wUWeXOZ6HJCfbEcyNZHNpBXtabhA/bMS3dhVnUR2hDgF5/Ch3wgevXB22VlpQECkRIFkZ6C3q8MD5mVIVQ9hSsHp9hy/mqzCkuRiIrVNfDEglgpMJCtimX5l0prnQPyB5I1B2zWNNJDxuzFGhRn7Nuj5l7Xq0rJN+wa5JlPQmOrTR55YKWi5HrP/r3Z6VAanf23fsWuaNayIDhsDv21Jgg3x+vo2aCx5kuYPnx67ci/3CDOK8YRfAHKuZm2LojEs0GB+FW4H71wQW/46p7LKlvHbU4/XjZFaW3gbrErKIlSVj4eL3kxi9/eCXEoRSueLBips/RWnS+Nrttf6jFLogOyJERFmLYMq+RpC2oViPgRDc3AGfsMlkJUrDY71LkrxbfMCAJXC94OE6A4egUzZTOkch/6dVWSLvOEX4z8ojSEyS5EwlOuVLoa9p+k+VqNRniqGXgto7083RKFu+6HGfQ5HG4CBurpzm7FJ3POM5h246urjyXB6CYVb7qgIa0MhjdNqAZSX+Uhwl0z/Q329vqbM4OZfO3tPO2M6TCPKDb/fcao22bYoNx4P0WfGGtHlzw8TpSQLPnsLpGX7da7LLKsBty2EDCElSlyGDWpoALNui02SGsAiSSF7MDg/ili/GJrYUPlV336X/hg/qqmd+gTVWK88qSyF6NEf7+jMxyHw4a7O7wrRi+PuzH1OJYSTINYGFIwc22MwDZ01gN38s/9i72FQQnI+7Jybt06JZtu/l9cnJp+17qmnZYGskswF1FM6YFGiIxS2++NmW9Vj1iIb1atpcfc9nU7P3RzgOy39Pcc3LH/32eN38Gc4oGB6t2uGqK0UeYAm+hj17+rGrOQGAd4FnJTWmk2euRuGtzU3Xps4YNIxU5kVEe1zMGyb+ueZGLRfu8CsLXqKv8jDWd19Ek1cSceUgUUhdJFXdu4gpiSw4w+QkSrok3A/4dYa30/W/SSyBbDRZ1U2p2ANYhz04oTJe1fEvasf9P9EQsArxQQMPMSdozhHiR3DJ8eks927/eT6YHUzUyL3PgvnOBlmsNxjJQEWARG9eC2NdD+Nvw/L/xZBdg==",
    "ecdh_public_key": "BKQKsKZsau62MJAk3nsEK/NoKdS80+a0j7TezfbwrWdAEPJzWk5/yAPldkVL3eiQkz3x2z6FDcIex7cu9W5aYf8=",
    "eddsa_public_key": "FFB2sQifRQNRLmetRroJ8PvcudBxBuRoYQpMSYdQhg6L",
    "ethereum_address": "e785f9e188f8137ec13ecea52ed753d4c7c7c064",
    "ip": "swarm1.dyne.org",
    "port_https": "20003",
    "reflow_public_key": "E+QVn6d+mrTDPl8g/a98CL9K+CVG1LRG1mdFvYb1nhAFHtMOVw+t3Y6gc+zzTKO7AFRwHyaYI9moXCKanHdcLS37+ebRuxoxB9qOwZhPM6IWJj9opQPdql8xdMz7T1yKBHnq7uy4rkywwUkSgG32nQXA7zPJKwHq+ieLaD65ePzi1n21L1vjIlNBVVDTjGmHD3/xTmgxSVcM8eYswOBSxv+EsU6YhAj9EAgp+OoTW2h7bSIPTXgI8i1COtcw2emA",
    "schnorr_public_key": "D54MEEyah5gC76dQscft9ggFt29tENpcp4Ms+6z5ZBaChQeu3iZee5/81Mq9MJEg",
    "tracker": "https://apiroom.net/",
    "uid": "swarm1.dyne.org:20003",
    "version": "2"
  },
  "subscriptions": {
    "iota_devnet": {
      "api": "https://api.lb-0.h.chrysalis-devnet.iota.cafe/",
      "name": "iota_devnet",
      "sub": "mqtt://mqtt.lb-0.h.chrysalis-devnet.iota.cafe:1883",
      "type": "iota"
    }
  }
}
```

 
* **Error Response:**

	None

* **Sample Call:**

```shell
 curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-get-identity' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {},
  "keys": {}
}'
```



----

### Ping

  Returns json data with a string.

* **URL**

  /api/zenswarm-oracle-ping

* **Method:**

  `GET` or `POST` 
  
* **Data Params**

  None

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "output": [
    "I_am_alive!"
  ]
}
```

 
* **Error Response:**

	None

* **Sample Call:**

```shell
 curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-ping' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {},
  "keys": {}
}'
```

----

### Get timestamp

Returns json data with a string, containing the timestamp fetched using the JavaScript method **getTime()** from the host machine

* **URL**

  /api/zenswarm-oracle-get-timestamp

* **Method:**

  `GET` or `POST` 
  
* **Data Params**

  None

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "myTimestamp": "1656931966016"
}
```

 
* **Error Response:**

	None

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-get-timestamp' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {},
  "keys": {}
}'
```

----

### Get signed timestamp

Returns json data with a string, containing the timestamp fetched using the JavaScript method **getTime()** from the host machine, along with its ECDSA signature, produced by the Oracle its ECDSA sk

* **URL**

  /api/zenswarm-oracle-get-signed-timestamp

* **Method:**

  `GET` or `POST` 
  
* **Data Params**

  None

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "ecdsa_signature": {
    "r": "ivjynFRQXm4EYKGwYpXSejoZvLNmEYHb3O5pFh2I+F8=",
    "s": "DWLIWDtSTxfyuKRuj2d0uIkRjKRaaJwvoBbU+qmFZJQ="
  },
  "myTimestamp": "1656932308339"
}
```

 
* **Error Response:**

	None

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-get-signed-timestamp' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
    "keyring": {
      "ecdh": "mukeqwntoJPtAN94jgahUA/ID7NptMLNL84sMPJ++eY="
    }
  },
  "keys": {}
}'
```

----

### Dilithium signature

  Returns json data contained in **asset** along with the [Dilithium QP signature](https://pq-crystals.org/dilithium/), produced by the Oracle its Dilithium sk. 

* **URL**

  /api/zenswarm-oracle-sign-dilithium

* **Method:**

  `POST` 
  
* **Data Params required:**
   
  `asset={data}`


* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "asset": {
    "array": [
      1,
      2,
      3
    ],
    "dictionary": {
      "number": 1969,
      "string": "hello world again!"
    },
    "number": 42,
    "string": "hello world!"
  },
  "dilithium_signature": "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"
}
```

 
* **Error Response:**

 **Code**: 500 Error: Internal Server Error <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-sign-dilithium]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-sign-dilithium' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
    "asset": {
     "string": "hello world!",
	 "number": 42,
	 "array": [1,2,3],
	 "dictionary": {"string": "hello world again!","number":1969}
    }
  },
  "keys": {}
}'
```

----

### ECDSA signature

   Returns json data contained in the **asset** along with the **ECDSA signature**, produced by the Oracle its ECDSA sk 


* **URL**

  /api/zenswarm-oracle-sign-ecdsa

* **Method:**

  `POST` 
  

* **Data Params required:**
   
  `asset={data}`

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
  "asset": {
    "array": [
      1,
      2,
      3
    ],
    "dictionary": {
      "number": 1969,
      "string": "hello world again!"
    },
    "number": 42,
    "string": "hello world!"
  },
  "ecdsa_signature": {
    "r": "VSWtbgWid2bMvXDVd2pemTlYb204E3+TgjhQh1nu38M=",
    "s": "EH16QToBcWVqasQ8+3uTSgklvvk6odXw/ut1KuDDHkk="
  }
}
```

 
* **Error Response:**

 **Code**: 500 Error: Internal Server Error <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-sign-ecdsa]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-sign-ecdsa' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
    "asset": {
     "string": "hello world!",
	 "number": 42,
	 "array": [1,2,3],
	 "dictionary": {"string": "hello world again!","number":1969}
    }
  },
  "keys": {}
}'
```

----

### EDDSA signature

   Returns json data contained in the **asset** along with the **EDDSA signature**, produced by the Oracle its EDDSA sk 


* **URL**

  /api/zenswarm-oracle-sign-eddsa

* **Method:**

  `POST` 
  
* **Data Params required:**

  `asset={data}`

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "asset": {
    "array": [
      1,
      2,
      3
    ],
    "dictionary": {
      "number": 1969,
      "string": "hello world again!"
    },
    "number": 42,
    "string": "hello world!"
  },
  "eddsa_signature": "3E7obC3UDtVsf5v8c52pYbqv39pJeFHGyFbqnK2shZdmtPB3EjfpwAKwYdZ8jrDGe6buHXUDvD9ZVeADLErLdMv6"
}

```

 
* **Error Response:**

 **Code**: 500 Error: Internal Server Error <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-sign-eddsa]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-sign-eddsa' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
    "asset": {
     "string": "hello world!",
	 "number": 42,
	 "array": [1,2,3],
	 "dictionary": {"string": "hello world again!","number":1969}
    }
  },
  "keys": {}
}'
```

----

### Schnor signature

Returns json data contained in the **asset** along with the **Schnorr signature**, produced by the Oracle its Schnorr sk 


* **URL**

  /api/zenswarm-oracle-sign-schnorr

* **Method:**

  `POST` 
  
* **Data Params required**

  `asset={data}`

* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
   "asset": {
      "array": [
         1,
         2,
         3
      ],
      "dictionary": {
         "number": 1969,
         "string": "hello world again!"
      },
      "number": 42,
      "string": "hello world!"
   },
   "schnorr_signature": "GO2MvxSUezuUGnNXJ8715MisUCn+bXzjISI311MhdWiXTfilahlFwUkwqFQ1MvBMBD/sA52Jq3h4T5zFMzSaZJfsMQZeTrJ64fG38oBw4Qw="
}

```

 
* **Error Response:**

 **Code**: 500 Error: Internal Server Error <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-sign-schnorr]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-sign-schnorr' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
    "asset": {
     "string": "hello world!",
	 "number": 42,
	 "array": [1,2,3],
	 "dictionary": {"string": "hello world again!","number":1969}
    }
  },
  "keys": {}
}'
```

----

### HTTP Post

Returns json data containing the result of the POST performed by the Oracle 


* **URL**

  /api/zenswarm-oracle-http-post

* **Method:**

  `POST` 
  

* **Data Params**

  `post{data={data to post}}` 
  
  `endpoint={URL}`
  
* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "output": {
    "result": {
      "User123456": {
        "keyring": {
          "ecdh": "a5b0Zw3RNUlUcKk9ZSmt53uesVJW5PHQ8gcKISbQjvo="
        }
      }
    },
    "status": 200
  }
}
```

 
* **Error Response:**

 **Code**: 200 Error: Internal Server Error <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-http-post]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm1.dyne.org:20003/api/zenswarm-oracle-http-post' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {	
      "post": {
		    "data": {
			    "myName": "User123456"
		}
	},
	"endpoint": "https://apiroom.net/api/dyneorg/API-generate-keyring-passing-user-name"
},
  "keys": {}
}'
```


----

### Execute Zenroom on Planetmint

Executes Zenroom in Planetmint, Returns json data containing Planetmint fullfillment and transaction id. Zenroom script is passed in the post, in the example, the script verifies a Dilithium QP signature.  


* **URL**

  /api/zenswarm-oracle-execute-zencode-planetmint.chain

* **Method:**

  `POST` 
  

* **Data Params**

  `asset={data}` 
  
  `data={data}`

  `script={zencode}`



* **Success Response:**

  * **Code:** 200 <br />
    **Content:** 

```json
{
  "output": {
    "result": {
      "success": true,
      "tx": {
        "asset": {
          "data": {
            "houses": [
              {
                "name": "Harry",
                "team": "Gryffindor"
              },
              {
                "name": "Draco",
                "team": "Slytherin"
              }
            ],
            "number": 42
          }
        },
        "id": "469481d390dbdb53cf718481b47a65b677f038d89951e14609e87a58fa784409",
        "inputs": [
          {
            "fulfillment": "pYIVMYCCAW9TY2VuYXJpbyAncXAnOiBCb2IgdmVyaWZpZXMgdGhlIHNpZ25hdHVyZSBmcm9tIEFsaWNlCkdpdmVuIEkgaGF2ZSBhICdkaWxpdGhpdW0gcHVibGljIGtleScgZnJvbSAnQWxpY2UnCkdpdmVuIHRoYXQgSSBoYXZlIGEgJ3N0cmluZyBkaWN0aW9uYXJ5JyBuYW1lZCAnaG91c2VzJyBpbnNpZGUgJ2Fzc2V0JwpHaXZlbiBJIGhhdmUgYSAnZGlsaXRoaXVtIHNpZ25hdHVyZScgbmFtZWQgJ2RpbGl0aGl1bSBzaWduYXR1cmUnIGluICdvdXRwdXQnCldoZW4gSSB2ZXJpZnkgdGhlICdob3VzZXMnIGhhcyBhIGRpbGl0aGl1bSBzaWduYXR1cmUgaW4gJ2RpbGl0aGl1bSBzaWduYXR1cmUnIGJ5ICdBbGljZScKVGhlbiBwcmludCB0aGUgc3RyaW5nICdvaycKgYIMt3siZGlsaXRoaXVtX3NpZ25hdHVyZSI6ICIwTVN1VlRHaTZORzV6c2ZKRFdVU0dEeUE2NUhWaVNMc1krcnl1T25ZOEFiWm4wRjd2ZjBMTUNVaG1wM2xiZ3VBOVZHYVAweXZZcWp4KzRCTkQ2V1NSbHpRa0phbk5RK2V1RHVXS3FKZHh1M2FnYzM5UGxqTitQUGgwOWxCRXBWVHNONlhaNzYrZVhmSzdFVytWd0dCc0Y3RGQvSU5KYWZscXEvNDZRakZ4bUlITHYydU9PeTVNTEpXYlFrMlJqc1RyTjdRRVVFQ05GN0FDWGlhbExZZXp6TWpqaFJVY0tqMkhGTHNQejFrcjRPcWxtMlpXb0tISzRjZXFCVUQrZ1ZlM1MvTm9WK1c1RDhvVVd1OW5NTDQ2UFQ5dTN3NGo3dTBFTzdJVXMxWmRaT2YyY1pBbHpHMmpIT0FLd3JrNlBtek9tQ1hSZ2lKZ1I1eTZFUENFb0ZBY0g2SGNTbUFpUXF1VUV4dURvZi85cFNGSXJHamZ2OXVOV2xPdzQ0c3VqWVphdkNNQnNxRlhzWFJUdDYvcWpnYkVwU2R6ZXJteU9VTmxqd2hWa0lUNXFvMHVYVE5EWnUxQVlqRlYvdSswQnBtb21JZHJ6UnhKcGJiVk1OcVgwQlhEREhZTlZnQVlmbUpZR3duNVE4UWVBNS8zN1Z2TXRqc2ZGenNVM051eFo2QnROTXdhWTh0elNYNmJjTE4zdGN0YlRZb0pKbDZnVEFxVFM3ZHVTVkZGMGpCNG9BM0luSDVkd2JPcjNBMEtoUEYrN0JyMll1U0EzTzErdFJPVGpNaGRCTXlCRDZSUjRWR0lFaTR0K05GbFR5YWxXUW9NaStBUGxnK2F5cjB3TDhWQ2grdHREWFRCc1huYkZ5NlBuUkpOUkE5UHo5VExnc3pwYUFmZUgzeisrRFRXbVJCOVpqaHRWdzBCb3BmN2lWaXYxaGZ0d1ZVUERJM2VQdzdFMjJKVjF4THRMQUd4dUhlTUhUT3QvSHkrZzUxdzMySGpFcVd5Qm9oYWRpQkxaYWxxdHhTWGJlV216YlkwMnQ1K0FQV3pFcXdJekFWSjBnMXI2Q3ZiMGNRMXV5SjhZazZDYS9xQjZvOHpZcWRyTDNHVVY3OVUxb3NORFl1cFZTTzg0TzRxb0drQWo2cDd6LzJtVi9QMkhha2YyVUxPU1RaaEhUODkvQ29vc3FQek5QanJsQ2hzbk5NWWgySGRFUG5ZQmRibGJHWlo3ZTlBS3JxWGQ2NUFJbU01VTIrU2dobGhKbkdNOWFOdSs0WTJwa1lZM2hlT1RUY09Pbm9xUk5adGVqYzV4RmI5MHZjUDJDN2s4RllvOXZRMG1jdFdRMlFUZVRRbm92Sko1S1ZPNUMxdnZUL1ZDcFNoUlpMNUYvcnhoOXdCMGx4L1BoSEtZSG9qZkVVd3YwTFYxR3llUGZhZ1VVTVMyVkJ3VFR3cmhOdzhhN2oxVUdxT1I4S0hpUXhnRHEwamo5bjd2RlZSVzdwaVZ1SjRBMWdEUkpZMVM0OVJYVC90a2xRb0RMQWZGOVN5ZEVEbVFmak10UFVQb3NadW5nK2gxMk9qVFE4eGhoeWM0WEVYQ0RMRUtOcS92bzlPeEFRUkJhL0lGVTFBa0FiZkpsQnVFVGpGMVlmbFE3M09xVG0xVHQxbEVpRkp6SkgxZXRGZU5GNWxES1Y2SlVQa3B0UHRZaTgyZXU5MWUxbDBFcE1GRU9XRlpqZnV1cXBJZWpCV2ozNWpqRUFiRXN4cFo4WlpjR25GbnBHZ09TWklFWVJkeEw4ZE1CSVBFVjBJUkxDR3VZU2hDWitGeTd1cEFjZTMxUEdJVDZ2Q2FvU3hoQmFNOHpiTWpkckd0ekJHU3dpams0aUZiRmVSbVFRTWg0SjlqcTVnYVFIMWE1amdEVnFJYllsT1RXME9aaDlZcHFPTW9TU1lUejlYUlFvQVBGYjQ2aTd5dWRjTE0zL3NGakpEWGNJT0ppUWlTN1ZIdFJqaTEzb2dQRG1MYnA3Z1hMdldQVzg5UmQzK3h3emhob1V5eWE0YzQ1SjNTc0U5bE4vT2pMWmlVaEh5bU5qWjEwZGc5SU1sL0ZjMmlmYmh1WCtYZDB3dFVCOXpJSmgzSFdMVGxsTkYzY3pRUGx0bit5cnJVK0tlZmVxRjAxUDhXaDQ5dDZxZENTb1diZ2puMGR4M1lrYURMTCtHcnRSNjIzM3hWdHpMSFZzaXFxVzEzYVQ5SmF6elFGNzdqdk9zdm0vdWdGOU1pdHl3MVhmMHNqYUZpWUJFL1RWRVRlemVKVEQyekNmOWYzMWFRTXpUNXFRVGI3dEJUUE90cENnZFlpQUY5Snl1YU5WekVKRUFYN2pqS2ZTaXg0bDRKN0xuVGNJZGRaZ1FBTXdqMW9MdHJqRXZXQm9Xb1NDNEdKMWs3OFhqY0hhSHRtdGxvTmRNM3dKTGJ4WTF4bC80UkNEb2ZZK2d5eTRsNjVKbTBzSHZqOVZmMUl2M2tycjAyZjhoNDBCa0Eva2tkdEcxdU9yT0hpNkNsZld0VGR3djhGUW4vSm1xM0wxU0VscTNPWGtLeWpYTjBLUnRJbml3dHgwRGFtSXZXQzRxay9jOGl6Mi9YOElVVjRiQTBObUw3ajRHVDRqbThlYzJ0ay9UbExacFRVUTFGVWVaNHF0aHpXRXJDTDdSUGJ6N1FXbFVVcm1ibE1Vb1luZVlHLzcxMjlhdVBvT2hleUFoR1djRUtGK0FNamtCWTl1anBMNGlreTYzZlcrK21kVUo4UVBKaUVtdElJWVBDWUJjV3drMXdEcTIrRlU1VDk0YmRQRTZlSnJwSVVLUW5sK0pJaXF5M0preFVtNms4YTkyZHM1Z2YxQXp4UkpEdU9IMkowV2FVQXNSenk5YzV1ait6dXJCcURuM2s3Z3VHQmZIKzFzZXZoR25neXRRWTFtWm11QUkxSzJPNU5OSWJlaHhPZ2I5NHlCVTdNVEFnUy82dDJDVi9naXRLQThuVWpsOStxcExWZm5Dd2ZYMWJpYTZwVkxPWUJaSlM2WDNNVUVCVFJXK2ZaenZubVVITDJjVHFNLzVIeXg2NDZucmE3NVU0c1pKck5Qd3FqWG1DQWNyN2xVaGZOcWMzeDJSdnBFRTdHMm5NdzZTRUFFTGJQaTladXJBWGZaNDNUc2tVa1UyakRqSlRzRHRZZkw1bXpUWnh1ZmdEbnBFYmR1TzdoTHJRdHVUY2Rydm01YVhXaWx2WXJ3Rk81RXhST2NjWitBUXJMcnFhTXVuVmhwT24vMXhXSkN4cEkydlBTOTFJY050UlAwZ1k5VGtlWUdqNGRuTFdjMnpnc0k1UGY0YVRXNUl3ZzlTejhJVERQazFBZ3VJaEhJWVZ6cXJETEUybkR6WU0vODI4by9hQzN0Rkw3blUrd0t2SlBya2xBQldWNlFwLzhvbGI2WFdwN29GVkFrMzFyS1VDaEJaSU1uY1RDNzJmZTY4RS8xWGNlN0RYQis5Wmg5QVJic0xscWdiUDAxOWlYZmN0K0NTSkQ5aS9RbG0xVVJMR0krODZORFFoZVIwQ0w5QUdpR2ROQ21LSmsxUjVyVFFxdE5ZRFp6allVcG9GZ1l4MGx1ZllZVVNhTVNuc1lsbEhqSDlIcHNWNGZLQXJ2VU1Wc00rOThmQmk0T0czSURiQ0FKWlh1OGVDN0FRVmV6UmxLOVdoZkNRZ3dkbE15aVJUVWJhRy9lR2s2c2FWWm1CZ0w1eVVHbUFpZ2VrQ1BCa1ZvQlBpYXBHemVnK05TVzYweVNZcTFIU0VnNmtFWW9leXFiTmwrbkJCeGZtb3U1QUNEZUI1RFdsbmRqZktCMTduQ0lXWjFVL2hqVjdBbWxsWllZVWNsSVBaRStiTGlpODNxWHlnVFJEeUNQVktFcHNYRTh5OTN1VWNkZUg3Q0dNWk9OYmw4Rk5FR0dKc0MrVEpudVUwVHZKNjFtcnF4UE1xenNsQWxMZ1VrcGhhRjhHbittcXpoaVg3NXlEYnFJZE1oS2x5VDFWWnJ4ZFZZSTU5WklJMnpJMTk1Ylh2cnV1ajJDd0hzTW1iKzFscHFPanZYRXc4UkZNUFh0eG1hSzRXcE5YL0hIOEIxQjJTbEhTQXk0MU9HMUpCR1Z2TGkrRTJsblE0ME03dFVEWlh3Vkl6QTFPa0ZaWTJpRWhiM0p5dUx4QXc0YlhGMWplb1dPd3VEOUZFU1BuNlNtckxUUDZQUDVCQ0FqSmo1QVEyTnNrWnFidGJyYzRlTHY5Z0FBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUJBY0tEcz0ifYGCBv97IkFsaWNlIjogeyJkaWxpdGhpdW1fcHVibGljX2tleSI6ICJ1b0dONnFLQnpRZk5PZjRtK0ZxTTc3UVlnQXRwR1JxNjVXNTh2amhaOGhzSkcwZXVzUC9RazdBL3dTdkhIN0pVNzQ5aklnNGpidnVGak91Z1oyUG5uTkxvd1VlRmk5ODNscmt0ZWtZbXp1M3NJaUhFZlB2UFE0WURkTGhIdEZJNnFyVzF6T2s2TXp6VUxvQndQOG5IajJJRXA5UEpQNUU0WUVwVFE2NnRFY1RROWtyZitXdHBneWkwNy9GOVo0NEdtdUpqN0I4Z00xL3M4R0F5NTc2VkNPaHY2Myt6a29MZTFpYW12Y0ZkdGNVU3VYWmJTaldHQzJhYjd3WkwrSWM2OCsxdVUvbGl5WHdvSW4zMDdSM29NWGdNbGdUVFh4dEo3eWEzN1dkbkJKNHBrR3Y1WUxOMEtIV0REUFd2V0ZjcWMzM2pNTTJXVWVpbEtJN1pIeHIyMk9RRFZHVmFxYmZzVEhCaXpjSlZESjUyTjZDYU16ZWg1c3FPRDlEZzV2Y01kT1dQTUxMRHF3U2ttL0VhU2RQSzZzSy9oSVFLbmpwcm02VnluemswZGlocjdWSVdFb2wxSWxnVEJBY1BUNFZOTlJZU3FDbVo3VGdlZXFPZ3lWeUE5MnJNazhJaHhuWW5RNGRnNnRVZmpvVGxsaWE5WmZCWWdiNW4wRW54OWdROHY3c1hHT0RlcjZQdzhHSzYyaFRsMiswY1lmOHBOb0NjVHZxckhXOXM0ZmtkZENyUXRQVTdHVUZGUkZqOVBMamVyNVB1VEtZTnNJTGFFMmRlVS9zZXJoTDVBSFZwamt4K0NUYmpnMURkMVdZL01wNVM1QkEvSkhYWnRocFMyeGFUMko0bUJhenhOWEs0NkxxV1EvT3J4Q1ZGTFpxM2tJd0l0Z1hwM0wxYmg3N0FkZHVlWHBJOFRCMnk3bzE2RTY0Z212aHRwQ0cxS2FkbWRUYkQ2SzVhMjZUQ1owMGRKRFNYRFI3c1QzOFBtUVBua1dheDMyYWlVVGFzT2gveXZhSnRmc1BvMUZxM0x4WDd4S1JZQ2pOTHQ3UWRsTjlxZWdoMXBnOXlac1Q2NkpqL0FIZ3d3SGNJcE9Ib3dJNjhhaFlmYmFvQkVUMHljdUFSTjQ3Y2kyQUJaZmNZcFFIZy9oRmRpcXd3Um5RMUx1ZHlhVDk5QmozMkZFeVFEV1ZwOHhEUGd3Z3RKNy9Fd0lSQTlaNm45dmx1Yk1hZ3lhdWh3YmtUSEJJZy9BYUZDYjZsSUMxRFpIMFFRc1NRYVN6Sk9nbmZ1QldsU1BxMXMyblZKUzRTYkhSUGVHdWtJLzJUZXkvUnNqMzRuZGlrTUJJV3haaHA5TWcxd0ZrcC82M21Xbm5ET2JOc3MwMitFdmRzd29hbU41d08vSDJGS3VPN20wNE5MOHIvOFZMa1cxdVdEaFVFNTVERGk5ZHVpZC84V1N5dlErUmtkS1dWMkxzUm1XaHZBaWRjUGlTS2pDUVZSY09CYmtsUkFnTnNVWTQ0aUZYMUJ5UG92S1NGTHdhTlJ4VDhHbWFsd25aQ3lBOTlPQ2taRHBrUUZnSUt2VFFBVDZZVkQ2UysySUo4YXk1SmZhdnJRWm1CV3RGeUVETEJKWTFKUGdDOUlxUjRqRDFlRUpBMmNmQ05PZm5uVHJSZWw1WWRZczBEajJKTlM1cnhNWlNseXBIUkR4WG5jcXhIcDFEYXhRU0NDSDlmKyt1cDgwWmh5ZXpZR0hONWRtUWJhdE5OUjRTeVorNUF6Z1dkKzBkalNSUDdLSXlTenc3c1JjOVZPdE9aYWlZV0o5VkxXQldPV0lFcko5bExqeHdPRlFXRkxUNnBnd2RQcmw2SUNxMGJmaGM3MlQxNW1RQWdTckhIc1hYYTFyUUxPbDFoUHBWK1VBMC9ZdXdZSzNzLzZrTVZuU0RadUkxYzRDanlKQ1BvbUNaU0NVeTk0bmJwQUZ2MTNkRWRPS2RqT0NZdno0NDY0NmVvbVZNeGxJSFBuK3ROQkpEWWNyMGg5NC9UR1ZhaW1wenY4MnU4Q2dlbnZMK0VOWVEwQkRmNkVvd3c0V3hWQzgvMlhmSjJ5ZGNkMnRUVUIyd01UZ3FBdFluUUg3Tm5zd2xGSFdWZlhndXhwL0pRMENUcExkQ2JXbG5GYlRNVVVFYjNmdW9Sa0J5b0VrL0d0NjZaSkdFNU45Q0N2dHEvZVVpNUE1OFAxL3RjWUM1ZTFJYlJxUnhoRVo2RkRnam9uZHNXS3IweXliQWs3ZXJXM0hGZlNaQnJ0dXV1SVd0TjdKc0xmQVc3WGdxUnVQVHhKci9EUWcxeGZRMU9uMEE4WC9sSHpBPT0ifX0",
            "owners_before": [
              "5Uv4KLinWJpKFagfab9r3P9jMRJhWMZv3CgRdBZZ4vvb"
            ]
          }
        ],
        "metadata": {
          "data": [],
          "oracleTimestamp": "1657027260276",
          "result": {
            "output": [
              "ok"
            ]
          }
        },
        "operation": "CREATE",
        "outputs": [
          {
            "amount": "1",
            "condition": {
              "details": {
                "public_key": "JLmXctACFNiYRMatskgW2AM8nJa93kxonqYB21mDxMDkhajmrJRogWZfJfhs",
                "type": "zenroom-sha-256"
              },
              "uri": "ni:///sha-256;M5IggboqrqqJiPNo13xwJ2Ov5K4XCOgLam79EAGIYSE?fpt=zenroom-sha-256&cost=131072"
            },
            "public_keys": [
              "5Uv4KLinWJpKFagfab9r3P9jMRJhWMZv3CgRdBZZ4vvb"
            ]
          }
        ],
        "version": "2.0"
      },
      "zenroom": {
        "data": {
          "asset": {
            "houses": [
              {
                "name": "Harry",
                "team": "Gryffindor"
              },
              {
                "name": "Draco",
                "team": "Slytherin"
              }
            ],
            "number": 42
          },
          "output": {
            "dilithium_signature": "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"
          },
          "result": []
        },
        "keys": {
          "Alice": {
            "dilithium_public_key": "uoGN6qKBzQfNOf4m+FqM77QYgAtpGRq65W58vjhZ8hsJG0eusP/Qk7A/wSvHH7JU749jIg4jbvuFjOugZ2PnnNLowUeFi983lrktekYmzu3sIiHEfPvPQ4YDdLhHtFI6qrW1zOk6MzzULoBwP8nHj2IEp9PJP5E4YEpTQ66tEcTQ9krf+Wtpgyi07/F9Z44GmuJj7B8gM1/s8GAy576VCOhv63+zkoLe1iamvcFdtcUSuXZbSjWGC2ab7wZL+Ic68+1uU/liyXwoIn307R3oMXgMlgTTXxtJ7ya37WdnBJ4pkGv5YLN0KHWDDPWvWFcqc33jMM2WUeilKI7ZHxr22OQDVGVaqbfsTHBizcJVDJ52N6CaMzeh5sqOD9Dg5vcMdOWPMLLDqwSkm/EaSdPK6sK/hIQKnjprm6Vynzk0dihr7VIWEol1IlgTBAcPT4VNNRYSqCmZ7TgeeqOgyVyA92rMk8IhxnYnQ4dg6tUfjoTllia9ZfBYgb5n0Enx9gQ8v7sXGODer6Pw8GK62hTl2+0cYf8pNoCcTvqrHW9s4fkddCrQtPU7GUFFRFj9PLjer5PuTKYNsILaE2deU/serhL5AHVpjkx+CTbjg1Dd1WY/Mp5S5BA/JHXZthpS2xaT2J4mBazxNXK46LqWQ/OrxCVFLZq3kIwItgXp3L1bh77AddueXpI8TB2y7o16E64gmvhtpCG1KadmdTbD6K5a26TCZ00dJDSXDR7sT38PmQPnkWax32aiUTasOh/yvaJtfsPo1Fq3LxX7xKRYCjNLt7QdlN9qegh1pg9yZsT66Jj/AHgwwHcIpOHowI68ahYfbaoBET0ycuARN47ci2ABZfcYpQHg/hFdiqwwRnQ1LudyaT99Bj32FEyQDWVp8xDPgwgtJ7/EwIRA9Z6n9vlubMagyauhwbkTHBIg/AaFCb6lIC1DZH0QQsSQaSzJOgnfuBWlSPq1s2nVJS4SbHRPeGukI/2Tey/Rsj34ndikMBIWxZhp9Mg1wFkp/63mWnnDObNss02+EvdswoamN5wO/H2FKuO7m04NL8r/8VLkW1uWDhUE55DDi9duid/8WSyvQ+RkdKWV2LsRmWhvAidcPiSKjCQVRcOBbklRAgNsUY44iFX1ByPovKSFLwaNRxT8GmalwnZCyA99OCkZDpkQFgIKvTQAT6YVD6S+2IJ8ay5JfavrQZmBWtFyEDLBJY1JPgC9IqR4jD1eEJA2cfCNOfnnTrRel5YdYs0Dj2JNS5rxMZSlypHRDxXncqxHp1DaxQSCCH9f++up80ZhyezYGHN5dmQbatNNR4SyZ+5AzgWd+0djSRP7KIySzw7sRc9VOtOZaiYWJ9VLWBWOWIErJ9lLjxwOFQWFLT6pgwdPrl6ICq0bfhc72T15mQAgSrHHsXXa1rQLOl1hPpV+UA0/YuwYK3s/6kMVnSDZuI1c4CjyJCPomCZSCUy94nbpAFv13dEdOKdjOCYvz44646eomVMxlIHPn+tNBJDYcr0h94/TGVaimpzv82u8CgenvL+ENYQ0BDf6Eoww4WxVC8/2XfJ2ydcd2tTUB2wMTgqAtYnQH7NnswlFHWVfXguxp/JQ0CTpLdCbWlnFbTMUUEb3fuoRkByoEk/Gt66ZJGE5N9CCvtq/eUi5A58P1/tcYC5e1IbRqRxhEZ6FDgjondsWKr0yybAk7erW3HFfSZBrtuuuIWtN7JsLfAW7XgqRuPTxJr/DQg1xfQ1On0A8X/lHzA=="
          }
        }
      },
      "zenroom_fulfillment": {
        "zenroomSha256": {
          "data": {
            "dilithium_signature": "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"
          },
          "keys": {
            "Alice": {
              "dilithium_public_key": "uoGN6qKBzQfNOf4m+FqM77QYgAtpGRq65W58vjhZ8hsJG0eusP/Qk7A/wSvHH7JU749jIg4jbvuFjOugZ2PnnNLowUeFi983lrktekYmzu3sIiHEfPvPQ4YDdLhHtFI6qrW1zOk6MzzULoBwP8nHj2IEp9PJP5E4YEpTQ66tEcTQ9krf+Wtpgyi07/F9Z44GmuJj7B8gM1/s8GAy576VCOhv63+zkoLe1iamvcFdtcUSuXZbSjWGC2ab7wZL+Ic68+1uU/liyXwoIn307R3oMXgMlgTTXxtJ7ya37WdnBJ4pkGv5YLN0KHWDDPWvWFcqc33jMM2WUeilKI7ZHxr22OQDVGVaqbfsTHBizcJVDJ52N6CaMzeh5sqOD9Dg5vcMdOWPMLLDqwSkm/EaSdPK6sK/hIQKnjprm6Vynzk0dihr7VIWEol1IlgTBAcPT4VNNRYSqCmZ7TgeeqOgyVyA92rMk8IhxnYnQ4dg6tUfjoTllia9ZfBYgb5n0Enx9gQ8v7sXGODer6Pw8GK62hTl2+0cYf8pNoCcTvqrHW9s4fkddCrQtPU7GUFFRFj9PLjer5PuTKYNsILaE2deU/serhL5AHVpjkx+CTbjg1Dd1WY/Mp5S5BA/JHXZthpS2xaT2J4mBazxNXK46LqWQ/OrxCVFLZq3kIwItgXp3L1bh77AddueXpI8TB2y7o16E64gmvhtpCG1KadmdTbD6K5a26TCZ00dJDSXDR7sT38PmQPnkWax32aiUTasOh/yvaJtfsPo1Fq3LxX7xKRYCjNLt7QdlN9qegh1pg9yZsT66Jj/AHgwwHcIpOHowI68ahYfbaoBET0ycuARN47ci2ABZfcYpQHg/hFdiqwwRnQ1LudyaT99Bj32FEyQDWVp8xDPgwgtJ7/EwIRA9Z6n9vlubMagyauhwbkTHBIg/AaFCb6lIC1DZH0QQsSQaSzJOgnfuBWlSPq1s2nVJS4SbHRPeGukI/2Tey/Rsj34ndikMBIWxZhp9Mg1wFkp/63mWnnDObNss02+EvdswoamN5wO/H2FKuO7m04NL8r/8VLkW1uWDhUE55DDi9duid/8WSyvQ+RkdKWV2LsRmWhvAidcPiSKjCQVRcOBbklRAgNsUY44iFX1ByPovKSFLwaNRxT8GmalwnZCyA99OCkZDpkQFgIKvTQAT6YVD6S+2IJ8ay5JfavrQZmBWtFyEDLBJY1JPgC9IqR4jD1eEJA2cfCNOfnnTrRel5YdYs0Dj2JNS5rxMZSlypHRDxXncqxHp1DaxQSCCH9f++up80ZhyezYGHN5dmQbatNNR4SyZ+5AzgWd+0djSRP7KIySzw7sRc9VOtOZaiYWJ9VLWBWOWIErJ9lLjxwOFQWFLT6pgwdPrl6ICq0bfhc72T15mQAgSrHHsXXa1rQLOl1hPpV+UA0/YuwYK3s/6kMVnSDZuI1c4CjyJCPomCZSCUy94nbpAFv13dEdOKdjOCYvz44646eomVMxlIHPn+tNBJDYcr0h94/TGVaimpzv82u8CgenvL+ENYQ0BDf6Eoww4WxVC8/2XfJ2ydcd2tTUB2wMTgqAtYnQH7NnswlFHWVfXguxp/JQ0CTpLdCbWlnFbTMUUEb3fuoRkByoEk/Gt66ZJGE5N9CCvtq/eUi5A58P1/tcYC5e1IbRqRxhEZ6FDgjondsWKr0yybAk7erW3HFfSZBrtuuuIWtN7JsLfAW7XgqRuPTxJr/DQg1xfQ1On0A8X/lHzA=="
            }
          },
          "script": "Scenario 'qp': Bob verifies the signature from Alice\nGiven I have a 'dilithium public key' from 'Alice'\nGiven that I have a 'string dictionary' named 'houses' inside 'asset'\nGiven I have a 'dilithium signature' named 'dilithium signature' in 'output'\nWhen I verify the 'houses' has a dilithium signature in 'dilithium signature' by 'Alice'\nThen print the string 'ok'\n"
        }
      }
    },
    "status": 200
  }
}
```

 
* **Error Response:**

 **Code**: 200  <br />
    **Content:**
    
```json
{
  "zenroom_errors": {
    "result": "",
    "logs": " ***Zenroom ERROR logs*** " 
  },
  "result": "",
  "exception": "[ZENROOM EXECUTION ERROR FOR CONTRACT zenswarm-oracle-http-post]\n\n\n Please check zenroom_errors logs"
}
```

* **Sample Call:**

```shell
curl -X 'POST' \
  'https://swarm0.dyne.org:20003/api/zenswarm-oracle-execute-zencode-planetmint.chain' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "data": {
	"asset": {
		"data": {
			"houses": [
				{
					"name": "Harry",
					"team": "Gryffindor"
				},
				{
					"name": "Draco",
					"team": "Slytherin"
				}
			],
			"number": 42
		}
	},
	"metadata": {
		"data": {}
	},
	"keys": {
		"Alice": {
			"dilithium_public_key": "uoGN6qKBzQfNOf4m+FqM77QYgAtpGRq65W58vjhZ8hsJG0eusP/Qk7A/wSvHH7JU749jIg4jbvuFjOugZ2PnnNLowUeFi983lrktekYmzu3sIiHEfPvPQ4YDdLhHtFI6qrW1zOk6MzzULoBwP8nHj2IEp9PJP5E4YEpTQ66tEcTQ9krf+Wtpgyi07/F9Z44GmuJj7B8gM1/s8GAy576VCOhv63+zkoLe1iamvcFdtcUSuXZbSjWGC2ab7wZL+Ic68+1uU/liyXwoIn307R3oMXgMlgTTXxtJ7ya37WdnBJ4pkGv5YLN0KHWDDPWvWFcqc33jMM2WUeilKI7ZHxr22OQDVGVaqbfsTHBizcJVDJ52N6CaMzeh5sqOD9Dg5vcMdOWPMLLDqwSkm/EaSdPK6sK/hIQKnjprm6Vynzk0dihr7VIWEol1IlgTBAcPT4VNNRYSqCmZ7TgeeqOgyVyA92rMk8IhxnYnQ4dg6tUfjoTllia9ZfBYgb5n0Enx9gQ8v7sXGODer6Pw8GK62hTl2+0cYf8pNoCcTvqrHW9s4fkddCrQtPU7GUFFRFj9PLjer5PuTKYNsILaE2deU/serhL5AHVpjkx+CTbjg1Dd1WY/Mp5S5BA/JHXZthpS2xaT2J4mBazxNXK46LqWQ/OrxCVFLZq3kIwItgXp3L1bh77AddueXpI8TB2y7o16E64gmvhtpCG1KadmdTbD6K5a26TCZ00dJDSXDR7sT38PmQPnkWax32aiUTasOh/yvaJtfsPo1Fq3LxX7xKRYCjNLt7QdlN9qegh1pg9yZsT66Jj/AHgwwHcIpOHowI68ahYfbaoBET0ycuARN47ci2ABZfcYpQHg/hFdiqwwRnQ1LudyaT99Bj32FEyQDWVp8xDPgwgtJ7/EwIRA9Z6n9vlubMagyauhwbkTHBIg/AaFCb6lIC1DZH0QQsSQaSzJOgnfuBWlSPq1s2nVJS4SbHRPeGukI/2Tey/Rsj34ndikMBIWxZhp9Mg1wFkp/63mWnnDObNss02+EvdswoamN5wO/H2FKuO7m04NL8r/8VLkW1uWDhUE55DDi9duid/8WSyvQ+RkdKWV2LsRmWhvAidcPiSKjCQVRcOBbklRAgNsUY44iFX1ByPovKSFLwaNRxT8GmalwnZCyA99OCkZDpkQFgIKvTQAT6YVD6S+2IJ8ay5JfavrQZmBWtFyEDLBJY1JPgC9IqR4jD1eEJA2cfCNOfnnTrRel5YdYs0Dj2JNS5rxMZSlypHRDxXncqxHp1DaxQSCCH9f++up80ZhyezYGHN5dmQbatNNR4SyZ+5AzgWd+0djSRP7KIySzw7sRc9VOtOZaiYWJ9VLWBWOWIErJ9lLjxwOFQWFLT6pgwdPrl6ICq0bfhc72T15mQAgSrHHsXXa1rQLOl1hPpV+UA0/YuwYK3s/6kMVnSDZuI1c4CjyJCPomCZSCUy94nbpAFv13dEdOKdjOCYvz44646eomVMxlIHPn+tNBJDYcr0h94/TGVaimpzv82u8CgenvL+ENYQ0BDf6Eoww4WxVC8/2XfJ2ydcd2tTUB2wMTgqAtYnQH7NnswlFHWVfXguxp/JQ0CTpLdCbWlnFbTMUUEb3fuoRkByoEk/Gt66ZJGE5N9CCvtq/eUi5A58P1/tcYC5e1IbRqRxhEZ6FDgjondsWKr0yybAk7erW3HFfSZBrtuuuIWtN7JsLfAW7XgqRuPTxJr/DQg1xfQ1On0A8X/lHzA=="
		}
	},
	"data": {
		"dilithium_signature": "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"
	},
	"script": "Scenario '\''qp'\'': Bob verifies the signature from Alice\nGiven I have a '\''dilithium public key'\'' from '\''Alice'\''\nGiven that I have a '\''string dictionary'\'' named '\''houses'\'' inside '\''asset'\''\nGiven I have a '\''dilithium signature'\'' named '\''dilithium signature'\'' in '\''output'\''\nWhen I verify the '\''houses'\'' has a dilithium signature in '\''dilithium signature'\'' by '\''Alice'\''\nThen print the string '\''ok'\''\n"
},
  "keys": {}
}'



```



### Reserved APIs

The following APIs contain the *business logic* of the Oracles and are used only internally, therefore not documented:

/api/ethNotarization-0-newhead

/api/ethNotarization-1-newhead

/api/ethNotarization-2-filter-newhead

/api/ethNotarization-4-ethereum-store

/api/ethereum-to-planetmint-3

/api/ethereum-to-planetmint-4

/api/iota-notarization-to-eth-0

/api/iota-notarization-to-eth-2

/api/iota-to-planetmint-1

/api/sawroom-notarization-0

/api/sawroom-notarization-1

/api/sawroom-notarization-2

/api/sawroom-notarization-4

/api/sawroom-to-planetmint-3

/api/zenswarm-oracle-announce

/api/ethNotarization-3-ethereum-store

/api/iota-notarization-to-eth-1

/api/sawroom-notarization-3

/api/ethereum-to-ethereum-notarization.chain

/api/ethereum-to-planetmint-notarization.chain

/api/iota-to-ethereum-notarization.chain

/api/iota-to-planetmint-notarization.chain

/api/sawroom-to-ethereum-notarization.chain

/api/sawroom-to-planetmint-notarization.chain

/api/zenswarm-oracle-key-issuance.chain

/api/zenswarm-oracle-generate-all-public-keys

/api/zenswarm-oracle-key-issuance-1

/api/zenswarm-oracle-key-issuance-3

/api/zenswarm-oracle-update

/api/zenswarm-oracle-key-issuance-2




### Update instances 

The instances now have update mechanism, triggered via an API (currently working only with 6 instances). The update flow to the instances is triggered by the API:

https://apiroom.net/api/dyneorg/consensusroom-update-all-instances

The API needs no parameter.


***
## 💼 License
    Zenswarm - {tagline}
    Copyleft (ɔ) 2021 Dyne.org foundation, Amsterdam

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU Affero General Public License as
    published by the Free Software Foundation, either version 3 of the
    License, or (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Affero General Public License for more details.

    You should have received a copy of the GNU Affero General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.

**[🔝 back to top](#toc)**
