# KILT Credential API (Draft Spec version 1.0)

## Definitions

### Extension

A browser extension that stores and uses the identities and credentials of the user. 
When the user visits a webpage, the extension injects its API in this webpage.

### dApp

Decentralized application. Examples of dApp in this specification are Attester and Verifier.
It is a website that can interact with the extension via the API it exposes.


## Setting up the communication session

### Types

`extensionId` references the extension on the `GlobalKilt` object but is not used by the dApp. `name` should be a human-readable string.

```typescript
interface GlobalKilt {
    [extensionId: string]: InjectedWindowProvider
}

interface InjectedWindowProvider {
    startSession: (dAppName: string) => Promise<PubSubSession>
    name: string
    version: string
    specVersion: '0.1.0'
}

interface PubSubSession {
    listen: (callback: (message: Message) => Promise<void>) => Promise<void>
    close: () => Promise<void>
    send: (message: Message) => Promise<void>
}
```


### DApp consumes the API exposed by extension 

The dApp can get all the available extensions via iterating over the `window.kilt` object.

```typescript
function getWindowExtensions(): InjectedWindowProvider[] {
    return Object.values(window.kilt || {});
}
```

The dApp should list all available extensions it can work with. 
The user selects the extension on this list, and the communication starts from there.

```typescript
async function startExtensionSession(
    extension: InjectedWindowProvider,
    dAppName: string,
): Promise<PubSubSession> {
    try {
        return await extension.startSession(dAppName);
    } catch (error) {
        console.error(`Error initializing ${extension.name}: ${error.message}`);
        throw error;
    }
}
```


### Extension injects its API into a webpage

```typescript
window.kilt as GlobalKilt = window.kilt || {};

window.kilt[extensionId] = {
    startSession: async (dAppName: string): Promise<PubSubSession> => {
        // Extension enables itself
        return { /*...*/ };
    },
    name,
    version,
    specVersion: '0.1.0'
} as InjectedWindowProvider;
```

`startSession` function gives the extension the possibility to initialize itself (maybe ask the user for permission to communicate with the page) and the URL of the page (which can be directly accessed by the extension) can be checked against an internal whitelist/blacklist.


### Processing the messages

Messages should be queued until the dApp calls `listen`.

The Promise should be resolved after the dApp or the extension have finished processing the message.
If they can't handle the received message, they can reject the Promise.


## Messaging Protocol

### General

It is recommended, that users can allow the extension to use the keys for x minutes, so that the users doesn't have to enter his/her password everytime a message (even an error message) is sent.

#### Error

|||
|-|-|
| direction | `Extension <-> dApp` |
| message_type | `ERROR` |
| description | General error, which should abort and reset the current workflow/protocol |
| encryption | any |

content:

```typescript
interface IError {
    code: number
    reason: string
}
```

> Error codes will be provided at a later time. For now, when receiving an error, the extension and dApp should reset. // @tjwelde

### Handshake Workflow

1. **DApp introduces itself**

*Entrypoint*

|||
|-|-|
| direction | `dApp -> Extension` |
| message_type | `SEND_DID` |
|encryption | false |

content: `string`

example_content: `did:kilt:5CqJa4Ct7oMeMESzehTiN9fwYdGLd7tqeirRMpGDh2XxYYyx`

2. **Extension requests authentication**

|||
|-|-|
|direction | `Extension -> dApp` |
|message_type | `REQUEST_AUTHENTICATION` |
|encryption| anonymous|

content

```typescript
interface IRequestAuthentication {
    cType: string
    trustedAttesters: string[]
    temporaryEncryptionKey: string
}
```

example_content:

```json
{
    "cType": "kilt:ctype:0x5366521b1cf4497cfe5f17663a7387a87bb8f2c4295d7c40f3140e7ee6afc41b",
    "trustedAttesters": [
        "did:kilt:5CqJa4Ct7oMeMESzehTiN9fwYdGLd7tqeirRMpGDh2XxYYyx"
    ],
    "temporaryEncryptionKey": "0x2203a7731f1e4362cb21ff3ef7ce79204e1891fc62c4657040753283a00300d8"
}
```

3. **DApp authenticates with DID**

Message includes a counter-challenge for the extension to sign.

|||
|-|-|
| direction | `dApp -> Extension` |
| message_type | `SUBMIT_AUTHENTICATION` |
| encryption | authenticated (to temporary key) |

content:

```typescript
interface ISubmitAuthentication { 
    credential: AttestedClaim
}
```

example_content

```json
{ 
    "credential": {}
}
```

> signing the challenge is not strictly necessary bc the message is authenticated/signed // @rflechtner 

> Might be very good to indicate that from this point on, the extension is 100% sure to be talking to a legit attester/verifier, so it is finally possible to reveal its DID (next step).

### Attestation Workflow

1. **Attester proposes credential**

|||
|-|-|
| direction | `dApp -> Extension` |
| message_type | `SUBMIT_TERMS`|

content:

```typescript
interface ISubmitTerms {
    cType: string
    claim: Partial<IClaim>
    delegationId?: string
    legitimations?: IAttestedClaim[]
    // quote?: IQuoteAttesterSigned
    //prerequisites?: {
    //    ctype: string
    //    trustedAttesters: string[]
    //    required: boolean
    //}[]
}
```

> The interface basically is the SubmitTerms message type, but I wasn't sure whether quotes and prerequisite claims are relevant for this use case. Prerequisite claims may better be handled via nested [Verification Workflows](#Verification-Workflow) after the extension submitted the RequestForAttestation. // @rflechtner 

> `prerequisites` is just an information for the user, that there will be a verification flow happening after the `request for attestation`, where the attester asks for credentials of specific ctypes to authenticate the user. // @tjwelde 

> For now we leave `prerequisites` out and see how applications use the verification flow and watch out for usefulness. // @tjwelde 

example_content:

```json
{
  "cType": "kilt:ctype:0x5366521b1cf4497cfe5f17663a7387a87bb8f2c4295d7c40f3140e7ee6afc41b",
  "claim": {
    "cTypeHash": "0xd8ad043d91d8fdbc382ee0ce33dc96af4ee62ab2d20f7980c49d3e577d80e5f5",
    "contents": {
      "grade": 12,
      "passed": true
    }
  },
  "delegationId": "4tEpuncfo6HYdkH8LKg4KJWYSB3mincgdX19VHivk9cxSz3F"
}
```

2. **Extension requests credential**

Only send with active consent of the user.

|||
|-|-|
| direction | `Extension -> dApp` |
| message_type | `REQUEST_CREDENTIAL`|
| encryption | authenticated |

content: `IRequestForAttestation`

```typescript
interface IRequestForAttestation {
    claim: IClaim
    claimNonceMap: Record<Hash, string>
    claimHashes: Hash[]
    claimerSignature: string
    delegationId: IDelegationBaseNode['id'] | null
    legitimations: IAttestedClaim[]
    rootHash: Hash
}
```

3. **Optional: Attester requests prerequisite credentials**

One or more instances of the [Verification Workflow](#Verification-Workflow) may happen before attestation of the credential, if the Attester needs to see prerequisite credentials.

4. **Attester submits credential**

|||
|-|-|
| direction | `dApp -> Extension` |
| message_type | `ATTESTED_CREDENTIAL`|
| encryption | authenticated |

content: `IAttestedClaim`

```typescript
interface IAttestedClaim {
    request: IRequestForAttestation
    attestation: {
        claimHash: string
        cTypeHash: ICType['hash']
        owner: IPublicIdentity['address']
        delegationId: IDelegationBaseNode['id'] | null
        revoked: boolean
    }
}
```

5. **Attester rejects attestation**

Send [Error type](#Error) message 

### Verification Workflow

*Prerequisite:* Authentication has finished via [Handshake Workflow](#Handshake-Workflow)

Repeat for multiple required credentials.

1. **DApp requests credential**

*Entrypoint*

|   |   |
| -------- | -------- |
| direction | `dApp -> Extension`|
| message_type | `REQUEST_CREDENTIAL` |
| encryption | authenticated |

content:

```typescript
interface IRequestCredential {
    cTypes: {
        [key: string]: {
            trustedAttesters: string[]
            requiredAttributes: string[]
        }
    }
}   
```

example_content:

```json
{
    "cTypes": {
        "kilt:ctype:0x5366521b1cf4497cfe5f17663a7387a87bb8f2c4295d7c40f3140e7ee6afc41b": {
            "trustedAttesters": [
                "did:kilt:5CqJa4Ct7oMeMESzehTiN9fwYdGLd7tqeirRMpGDh2XxYYyx"
            ],
            "requiredAttributes": [
                "name"
            ]
        }
    }
}
```

2. **Extension sends credential**

Only send with active consent of the user.
This closes the thread.

|||
|-|-|
| direction | `Extension -> dApp` |
| message_type | `SUBMIT_CREDENTIAL`|
| encryption | authenticated |

content: 

```typescript
interface ISubmitCredential {
    credential: IAttestedClaim
}
```
