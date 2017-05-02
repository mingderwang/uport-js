# uport-js

## Integrate uport in your javascript application

Uport provides a simple way for your users to login to your website and provide private credentials such as identity information and contact details to you.

You can also “attest” credentials they provide to you or that you yourself have about them. This can be shared back to your customers so you can help them build their digitial identity.

Uport.js provides a simple way for you to integrate uport.js into your javscript application. You can also interact with your uport users directly in the browser.

We have an easy to use browser library [uport-connect](https://github.com/uport-project/uport-connect) which can help you do so.

## Setup your uport application identity

First make sure you have your uport app installed and you've setup your own uport identity.

### What is a uport identity?

A uport identity is a special kind of ethereum Smart Contract known as a proxy contract. It has an ethereum address, can hold value and can interact with any other Ethereum smart contracts.

The identity is controlled by a Controller Contract, which is basically an access control layer. You can develop your own custom access control layers and replace them, while keeping the primary identity intact (or permanent as we like to call it).

Our default uport controller contract is controlled by a single device, but you can add multiple recovery contacts to help retain control if you lose that device. Feel free to replace the controller contract with your own contract.

### The uport registry

A uport identity also optionally has a public profile stored on ipfs and linked to your identity through the uport registry. This profile consists of JSON using the [Schema.org](http://schema.org/) conventions.

When interacting privately with a user you will be interchanging signed JWT. To verify the signature of the JWT you and your users will be fetching your public key from the public profile.

## Configure your application

In your application you must first configure your Uport object.

```javascript
import { Credentials, SimpleSigner } from 'uport'

const signer = SimpleSigner(process.env.PRIVATE_KEY)
const credentials = new Credentials({
  appName: 'App Name',
  address: 'MNID Encoded uPort Address For Your App'
  signer: signer,
  networks: networks
})
```

Going forward all uPort application ID addresses must be [MNID encoded](https://github.com/uport-project/mnid). MNID will encode the network with the address. Use of hex encoded addresses is deprecated. Using a hex encoded address will indicated you are on ropsten using our deprecated registry, if you require this use case then continue to pass a hex encoded address. If you are on ropsten but using our latest registry, pass a MNID encoded address with ropsten.

The networks object includes a set of networks for which JWTs will be verified over. JWT verification includes an on-chain lookup for the public key mapped to the issuers identity, the MIND encoding of the issuer's address defines the network and registry to use for lookup. If you are interested in verifying JWTs over additional networks, pass in a network configs object, defined as follows:

```javascript
 const networks = { id: '0x2a' :
                      { registry: '0x5f8e9351dc2d238fb878b6ae43aa740d62fc9758',
                        rpcUrl: 'https://kovan.infura.io' },
                   id: ....   : { ... }
                 }
```

Look in [uport-lite](https://github.com/uport-project/uport-lite) for the default networks and registries which will be queried for JWT verification.

## Authentication

Having an asymmetric keypair on an end user's mobile device which can sign data blobs and is linked to an on chain identity allows for strong authentication through a challenge-response protocol. Allowing you as a developer to request the user you are interacting with to prove they are who they say they are or prove that that they control a given uPort Identity. You may this use similarly to traditional authentication by issuing session tokens upon successful authentications, mapping data for you service to a given uPort id and authorizing additional actions. If you are issuing claims you will likely want to authenticate a user before issuing a claim for their identity. If you are consuming claims you will likely want to authenticate a user as well to be sure the claims you received were sent by the claim's subject and not replayed by an attacker.

To use authentication, instantiate AuthCredentials instead of Credentials. AuthCredentials wraps Credentials with additional functionality used for authentication.  This includes adding a challenge during `createRequest` and verifying the challenge during `receive`. It also requires a persistence layer, we provide a minimal implementation using Redis. Redis must be running, to install and start redis in the background, follow the [instructions here](https://redis.io).

```javascript
import { AuthCredentials, SimpleSigner } from 'uport'

const signer = SimpleSigner(process.env.PRIVATE_KEY)
const authCredentials = new Authcredentials({
  ...
})
```

To implement authentication you must run a sufficiently secure server which you can receive requests and return responses over HTTPS. You will need an endpoint to receive requests to login. When a login request is received, use `AuthCredentials.createRequest` to prepare a request wrapped with additional authentication params. Be sure to set the callback to an endpoint on your server which you can receive responses from the mobile app. Return the request token to the browser to be displayed in a QR code and passed to the phone.

Once the mobile device receives and verifies the request, it will prepare a response and post it to your callback endpoint. On receiving that response use `AuthCredentials.receive` to verify and parse the response. If the response is valid you can pass a response to the target client (browser). The target client (browser) should be polling for the given response.

#### Setup Outline

Browser makes GET request to the server, JWT challenge request token is returned

```javascript
// GET https://yourserver.com/authrequest
authCredentials.createRequest({requested: [..., ...], callback: 'https://yourserver.com/authresponse'}).then(res => {
  // return the response to the browser
})
```

Mobile device makes POST request with the response of the challenge to the server.

```javascript
// POST https://yourserver.com/authresponse
authCredentials.receive(jwt, response).then(res => {
  // Auth Successful
}).catch(err => {
  // Auth failed
})
```

Browser polls with a GET request for the response from the server.

```javascript
// GET https://yourserver.com/authresponse?pairId=<paidId>
authCredentials.authReponse(pairId).then(res => {
  // return the response to browser
}).catch(err => {
  // No response available, not available yet, or authentication failed
})
```

#### Storage

Authentication requires a persistent layer. We provide a interface which is a simple Redis client wrapper. The storage object can be instantiated as follows.

```javascript
import { Storage } from 'uport'
const Storage = new Storage({host: 'redis-host', port: 'redis-port'})
// Additional valid redis options can also be passed
```
If you already you have your own persistence layer, simple implement a wrapper that implements the same interface and set it when instantiating the AuthCredentials object.

```javascript
const authCredentials = new AuthCredentials({storage: yourStorageObject, ...})
```

#### Security Notes

Use HTTPS on your server and follow [redis best security practices](https://redis.io/topics/security).

## Requesting information from your users

To request information from your user you create a Selective Disclosure Request JWT and present it to your user in the web browser.

The most basic request to get a users public uport identity details:

```javascript
credentials.createRequest().then(requestToken => {
  // send requestToken to browser
})
```

You can ask for specific private data like this:

```javascript
credentials.createRequest({
    requested: ['name','phone','identity_no'],
    callbackUrl: 'https://....' // URL to send the response of the request to
  }.then(requestToken => {
  // send requestToken to browser
  })
```

In your front end use [uport-connect](https://github.com/uport-project/uport-connect) to present it to your user either as a QR code or as a uport-button depending on whether they are on a desktop or mobile browser.

```javascript
const connect = new uportconnect.Connect('app name')
connect.showRequest(requestToken).then(response => {
  // send response back to server
})
```

Back in your server code you receive the token:

```javascript
credentials.receive(responseToken).then(profile => {
  // Store user profile
})
```

For more information about the contents of the profile object see the uport-persona documentation.

### Requesting Push notification tokens from your users

As part of the selective disclosure request you can ask for permission from your users to communicate directly with their app.

```javascript
credentials.createRequest({
  requested:[...],
  notifications: true
}).then(requestToken => {
  // send to browser
})
```

Present it to the user like before. On the server you can receive the push token like this:

```javascript
credentials.receive(responseToken).then(profile => {
  // Store user profile
  // Store push token securely
  console.log(profile.pushToken)
})
```

## Attesting information about your users

Attesting information about your users helps add real value to your application. Your users will use uport to build up their own digital identity and your business is an important part of this.

If you're a financial institution you may be able to attest to KYC related information such as national identity numbers. If you're an educational application you may want to attest to your users achievements in a way that they can securely share.

### What are attestations

Attestations are shareable private information that one party can sign about another party. They are designed to be shared privately by you to your users and by them to other users.

### Creating an attestation

```javascript
credentials.attest({
  sub: '0x...', // uport address of user
  exp: <future timestamp>, // If your information is not permanent make sure to add an expires timestamp
  claims: {name:'John Smith'}
}).then(attestation => {
  // send attestation to user
})
```

As before you will want to send this to your user. You can do this in the browser

```javascript
const connect = new uportconnect.Connect('app name')
connect.showRequest(attestation) // no response is needed for an attestation
```

If you requested a push notification token in the above selective disclosure step you can also send attestations directly to your users app in real time.

```javascript
// Coming soon, not yet implemented
credentials.push(pushToken, `me.uport:add?attestation=${attestationjwt}`, message).then(response => {

})
```

## Asking users to sign Ethereum transactions

Ethereum smart contracts live on the blockchain and at a certain address. The application interface is known as the abi and can be created by the Solidity compiler.

Our Contract class will let you create a javascript object modelling the SmartContract allowing you to create uport uri's that you can send to the user.

```javascript
import { Contract } from 'uport'
const abi = // import from json or have directly in code
const contract = Contract(abi).at(contractAddress)
// creates a request for the user to call the transfer() function on the smart contract
const txRequest = tokenContract.transfer(....)
```

In your front end use 'uport-connect' to present it to your user either as a QR code or as a uport-button depending on whether they are on a desktop or mobile browser.

```javascript
const connect = new uportconnect.Connect('app name')
connect.sendTransaction(txRequest).then(txResponse => {
  // send response back to server
})
```

Back in your server code you receive the `txResponse`. This is a standard ethereum transaction object that you can verify.

## Creating Custom Signers for integrating with HSM

You can easily create custom signers that integrates into your existing signing infrastructure.

```javascript
function sign(data, callback) {
    const signature = '' // send your data to your back end signer and return DER signed data
    callback(null, signature)
}
```
