# Kamaji:
![kamaji](https://github.com/XLabs/kamaji/assets/12457451/bf60762f-3028-4611-b1db-e0cc66706103)
## What:
Kamaji is a relay-engine modeled as a "service registry" that aims to make it easy to run an **ecosystem of relayers**.

## Why:
Running an off-chain relayer for cross-chain applications *built on wormhole** is no easy task. There are multiple things you need to juggle simultaneously: 
- ensuring **every vaa** is redeemed
- managing contract interfaces across ecosystems
- securely providing funds to pay for relaying
- calculating the correct fee for a relay based on current network conditions
- tracking the relayer tokens inflows and outflows
- mantaining relay-protocol contracts configuration up to date
- and many, many more...

This gets particularly difficult when you need to run **many** production grade relayers, like we do at [xLabs](https://xlabs.xyz/).

## High Level Architecture:
Kamaji is modeled as a service-registry, where offchain clients can register themselves to be assisted by services that abstract away the complexity of interacting with blockchains.
The operator, Kamaji, will make sure that services are provided to interested clients.
From this perspective, you can think of the relayer ecosystem in as being formed of two types of components:
- *Clients:* Mainly Relayers, Oracles and Watchers. These contain the business logic and the engine focuses on simplifying the spawning and management of this components with minimal code, making it easy to spawn dozens if not hundreds of reliable, predictable and observable relayers.
- *Services:* Components purposed to assist clients in performing their tasks by managing all the environment related complexity (rpc pools, wallets, security, pnl tracking, error handling, monitoring, alerting).

This aims to abstract all environment complexity out of a relayer, with two main purposes:
- Allowing the relayer to focus only on the business logic
- Reusing all infrastructure related code. (TODO: link quote to why)

![Kamaji](https://github.com/XLabs/kamaji/assets/12457451/7fa4a851-2f7b-46fc-9a99-a437931766ea)

### Client Components:

- **Relayer:** Clients that perform relay. They can be uniquely identified by an ID provided. This ID will be used to login against keycloak This id will have a KMS role associated with it, which will determine what wallets the relayer can use, and how it can use them.
- **Oracle:** Worker running contract update tasks when this is required. The hooks to understand when this is required are provided by the infrastructure.
- **Observer:** 

### Service Components:

- **VAA Stream Service:** Subscribe to a stream of VAAs from an emitter. Stream service provides an API to register to `emitted`, `signed`, `re-orged` and `finalized` events.
- **KMS:** Uses an authorization service to authenticate each component and assign a role to it. Each role has a set of keys associated to it and policies on how can this keys be used.
- **Accountant:** Provides an easy way to keep track of the cost and benefit of operations performed.
- **Observer:** Provides hooks to execute tasks based on different criterias applied to the blockchain state, such as triggering a hook when certain threshold is reached.
- **Rpc Authority:** Provides of healthy RPC nodes for an specified set of chains. Potentially uses the authorization service to allow access to paid RPCs only for certain components.

## Components:

### Relayer:

A relayer can be considered a lambda function that needs to be serviced with:

- A stream of VAAs for certain emitter combinations (emitter addresses + emitter chains)
- Funds to pay for the relays
- Monitoring of the outcome of the VAAs processed including its economic result.

In a fully automated relay engine, starting a relayer would require:

- Registering a list of emitters to listen to
- Registering a list of chains wallets will be needed for
- Registering a function that can parse the body of a VAA
- Registering a function that can route a VAA to a queue
- Registering a function that given a VAA and a wallet pool, knows how to use the wallets in the pool to redeem the VAA
- Registering a function that given a VAA and its RelayTx knows how to recover the fee charged for relaying that VAA

```tsx

type RelayTx {}; // represents the cost of a tx.
type RelayFee {};
type VAA { id, hash, bytes };
type QueueDescriptor { name, arn };

abstract class Relayer<VaaFormat> {
	emitters: EmitterFilter[]
	targets: SupportedChain[]
	parseVaa(vaa: VAA): VaaFormat {} // maybe we don't need to enforce this one :thinking:
	route(vaa: VaaFormat): QueueDescriptor {}
	handle(vaa: VaaFormat): RelayTx[] {}
	calculateReceivedFee(vaa: VaaFormat, relayTxs: RelayTx[]): RelayFee {}
}
```

With this simple six properties interface, Kamaji could service a relayer to help it go through the entire life-cycle of a relay, by:

- Hooking to VAA streams and forwarding them to the appropriate queue
- Providing the relayer with a wallet pool that always contains necessary funds
- Providing the relayer with an RPC pool that always contains healthy RPCs
- Monitoring and broadcasting the result of VAA execution, including its economic results

**Relayer Bootstrap:**

Relayer will send Kamaji and declare the resources it needs to be served with, and Kamaji will create, update or delete resources as needed.

1. Kamaji will authenticate the relayer and get its KMS role.
2. Relayer will send a manifest:
    1. emitter combinations it’s interested in
    2. chains it will operate on
    
    Using this data, Kamaji will:
    
    1. Verify that VAA Stream Registry is up to date with the relayer requirements
    2. Verify that the KMS role has the wallets required by the relayer. If some wallet is not available it can create it along with its alert.
    3. Return to the relayer the list of wallets it might use for each chain, along with a list of  the most current RPCs
3. Relayer will use the wallets and rpcs returned in the step above to instantiate wallet-manger and start waiting for VAAs
4. Relayer will start a worker that reads from the pertinent queues.

**Relayer Operations:**

1. VAA Streamer will use the stream registry to stay up to date on the VAA for those emitters using as many sources as possible (initially spy and missed-vaa v4)
2. When a VAA is detected for an emitter, VAA streamer will call the `route()` method to understand what queue the vaa should be routed to. Upon success, it'll forward the vaa to such queue.
3. If the call to `route()` repeatedly fails, VAA streamer can mark the VAA as stuck and trigger an alert
4. **Relayer** will process the VAA and post its results to a new **topic** so that they can be post processed by interested parties, such as recording the atomic pnl, recording the vaa status in the vaa-stream-registry or gathering specific metrics.

### Oracles:
**Oracle bootstrap**
**Oracle operations**
