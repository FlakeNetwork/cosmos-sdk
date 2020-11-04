<!--
order: 4
-->
# `Msg` Services

A `Msg` Service processes [`message`s](./messages-and-queries.md#messages). `Msg` Services are specific to the module in which they are defined, and only process `message`s defined within the said module. They are called from `baseapp` during [`DeliverTx`](../core/baseapp.md#delivertx). {synopsis}

## Pre-requisite Readings

- [Module Manager](./module-manager.md) {prereq}
- [Messages and Queries](./messages-and-queries.md) {prereq}

## Implementation of a module `Msg` service

All `Msg` processing is done by a `MsgServer`](messages-and-queries.md#msg-services) protobuf service. Each module should define a `Msg` service, which will be responsible for request and response serialization. 

As further described in [ADR 031](../architecture/adr-031-msg-service.md), this approach has the advantages of clearly specifying return types and generating server and client code.

When possible, the existing module's [`Keeper`](keeper.md) should implement `MsgServer`, otherwise a `msgServer` struct that embeds the `Keeper` can be created, typically in `./keeper/msg_server.go`:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc1/x/bank/keeper/msg_server.go#L14-L16

At the moment, we use the SDK's [existing routing mechanism](#handler-type) to route a `Msg` to the appropriate `MsgServer` method. But this is intended to involve to satisfy the use cases discussed in [\#7093](https://github.com/cosmos/cosmos-sdk/issues/7093)
and [\#7122](https://github.com/cosmos/cosmos-sdk/issues/7421).

`NewHandler` function dispatches a `Msg` to appropriate `MsgServer` RPC function, usually by using a switch statement:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc1/x/bank/handler.go#L10-L30

First, `NewHandler` function sets a new `EventManager` to the context to isolate events per `msg`.
Then, a simple switch calls the appropriate `msgServer` method based on the `Msg` type. `msgServer` methods can retrieve the `sdk.Context` from the `context.Context` parameter method using the `sdk.UnwrapSDKContext`:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc1/x/bank/keeper/msg_server.go#L27

`Msg` processing usually follow this 2 steps:

- First, they perform *stateful* checks to make sure the `message` is valid. At this stage, the `message`'s `ValidateBasic()` method has already been called, meaning *stateless* checks on the message (like making sure parameters are correctly formatted) have already been performed. Checks performed in the `msgServer` method can be more expensive and require access to the state. For example, a `msgServer` method for a `transfer` message might check that the sending account has enough funds to actually perform the transfer. To access the state, the `msgServer` method needs to call the [`keeper`'s](./keeper.md) getter functions. 
- Then, if the checks are successfull, the `msgServer` method calls the [`keeper`'s](./keeper.md) setter functions to actually perform the state transition. 

Before returning, `msgServer` methods generally emit one or multiple [`events`](../core/events.md) via the `EventManager` held in the `ctx`:

```go
ctx.EventManager().EmitEvent(
		sdk.NewEvent(
			eventType,  // e.g. sdk.EventTypeMessage for a message, types.CustomEventType for a custom event defined in the module
			sdk.NewAttribute(attributeKey, attributeValue),
		),
    )
```

These `events` are relayed back to the underlying consensus engine and can be used by service providers to implement services around the application. Click [here](../core/events.md) to learn more about `events`. 

The invoked `msgServer` method returns a `proto.Message` response and an `error`. These return values are then wrapped into an `*sdk.Result` or an `error` using `sdk.WrapServiceResult(ctx sdk.Context, res proto.Message, err error)`. This method takes care of marshaling the `res` parameter to protobuf and attaching any events on the `ctx.EventManager()` to the `sdk.Result`.

+++ https://github.com/cosmos/cosmos-sdk/blob/d55c1a26657a0af937fa2273b38dcfa1bb3cff9f/proto/cosmos/base/abci/v1beta1/abci.proto#L81-L95

`NewHandler` can then be registered from [`AppModule.Route()`](./module-manager.md#appmodule) as shown in the example below:

+++ https://github.com/cosmos/cosmos-sdk/blob/228728cce2af8d494c8b4e996d011492139b04ab/x/gov/module.go#L143-L146

## Legacy Amino `Msg`s

### `handler` type

The `handler` type defined in the Cosmos SDK will be deprecated in favor of [`Msg` Services](#implementation-of-a-module-msg-service).

Here is the typical structure of a `handler` function:

+++ https://github.com/cosmos/cosmos-sdk/blob/7d7821b9af132b0f6131640195326aa02b6751db/types/handler.go#L4

Let us break it down:

- The [`Msg`](./messages-and-queries.md#messages) is the actual object being processed. 
- The [`Context`](../core/context.md) contains all the necessary information needed to process the `msg`, as well as a cache-wrapped copy of the latest state. If the `msg` is succesfully processed, the modified version of the temporary state contained in the `ctx` will be written to the main state.
- The [`*Result`] returned to `baseapp`, which contains (among other things) information on the execution of the `handler` and [`events`](../core/events.md).


+++ https://github.com/cosmos/cosmos-sdk/blob/d55c1a26657a0af937fa2273b38dcfa1bb3cff9f/proto/cosmos/base/abci/v1beta1/abci.proto#L81-L95

Module `handler`s are typically implemented in a `./handler.go` file inside the module's folder. The [module manager](./module-manager.md) is used to add the module's `handler`s to the
[application's `router`](../core/baseapp.md#message-routing) via the `Route()` method. Typically,
the manager's `Route()` method simply constructs a Route that calls a `NewHandler()` method defined in `handler.go`.

### Implementation

In this case, `handler`s functions need to be implemented for each module `Msg` and should be used in `NewHandler` in the place of [`msgServer` methods]((#implementation-of-a-module-msg-service)). This will also involve manual handler registration of `Msg` types.
`handler`s functions should return a `*Result` and an `error`.

## Telemetry

New [telemetry metrics](../core/telemetry.md) can be created from `msgServer` methods when handling messages for instance. 

This is an example from the `x/auth/vesting` module:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc1/x/auth/vesting/msg_server.go#L73-L85

## Next {hide}

Learn about [query services](./query-services.md) {hide}