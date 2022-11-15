# Writing Safe Canisters

> This post is written by [CanScan](../), we audit Rust canisters. Reach out!

The Internet Computer is a new platform, hence what we are witnessing, the performance, the functionalities and more importantly the software design patterns applied in the ecosystem are only the tips of the iceberg.

Due to this reason, it may still not be clear what kind of bugs to look for when checking the codebase for an Internet Computer canister, this post is intended to be a small contribution 

## Unit testing of the canister

This should come naturally given the very foundation of what a canister really is, it is software, and like any other software, it is not worthy of any particular or mission-critical role if it’s not unit tested properly.

We recommend a separation of concern between your application logic, and the interface that exposes that logic to the public.

Make sure to reach code coverage of at least 75% in your projects, and apply fuzz testing when applicable.

## Cycles

One large point of concern is related to cycle draining attacks or points of failure related to the reverse-gas model.

1. Ensure that your canister is always filled with cycles by any means necessary, until when there are protocols put in place for this.
2. Make sure you’re not sending cycles to other canisters without any conditions, as logical mistakes in this area can be expensive, it is recommended to take a comprehensive detailed unroll of your code reaching any inter-canister call which sends cycles.
3. Filter the messages you want your canister to accept by implementing proper `inspect_message` and reason about the acceptable payload size of each method.
4. Double-check that you’re not causing any infinite loops by unconstrained retry logic. 

## Trapping

Trapping in the IC is like a panic, but your canister’s state changes are also rollbacked, as if the call never happened, traps can be expensive, and they can cause difficulty when it comes to reasoning about the correctness of a certain piece of code, ensure that you do understand the implications of what a trap does and only use when it makes sense, prefer returning an error or rejecting the call in place of traps when applicable.

## Concurrency & State Management

Traps are one of the many reasons concurrency is hard to reason about, one important note to remember when checking concurrent IC code is that a trap only rollbacks the state that was changed during the current callback (anything after the `await`).

Additionally, any other entry point of your codebase can be executed while your awaiting the response for an inter-canister call (this of course includes the callbacks as well), test your logic for this property, or employ proper design patterns common to distributed systems such as 2 phase commits, WALs or similar patterns.

## DDoS Attacks

You do not need to worry about this at the protocol level, Internet Computer is already supposed to scale, what you need to worry about is resource allocation for your own data and users.

Make sure you are using your heap properly, and use the stable storage as well if you know your data is more than the heap (which is 4GB).

Some other patterns to implement are:

1. Multi-canister achieving.
2. Horizontal scaling.
3. Removing old data.
4. Using other canisters as storage and cache in your main.
5. Using stable storage as the main storage.
6. Limiting the user inputs that need to be preserved.
7. Having a garbage collector might help as well.

## Authentication & Access Control

If part of the functionality of you canister is limited to a certain group of controllers/admins, make sure you’re not missing any of those anywhere, this seems obvious, but is actually something you might forget, so during an audit always check for this problem, have a clear definition of the access structure of your canister.

## Misuse of WASM System API calls

Surprise! you are not allowed to use all of the system API calls in all of the entry points of your canister, the most common API misuse to look for is the use of `ic::caller()` in reply/reject callbacks, this results in your canister trap.

Once in a while, it might *actually* be a good idea to check out the [IC Interface Spec](https://internetcomputer.org/docs/current/references/ic-interface-spec) 🙂

## Upgradability & Migrations

Keep the operation needed to be executed in your pre/post upgrade code to a minimum, these functions should NEVER fail under any circumstances, this will simply lock your canister, and forces you to `reinstall` your canister.

## The consequence of Untrusted Calls

When calling another canister, make sure you trust that canister, a malicious canister can simply a prevent reply later to your code late on purpose, for example after you’ve completed an upgrade, this will result for broken callback tables in the canister, this has undefined behaviour and can ruin your canister logic.

When making a call to an untrusted canister, make sure you’re not passing any callbacks, use the values of -1, and -1 for reply/reject callback when doing a one-way call.
