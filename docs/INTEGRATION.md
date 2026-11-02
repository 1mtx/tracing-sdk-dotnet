# .NET SDK: integration guide

This guide is for a client (the party that commits tickets) that sends tickets to the tracing backend with the .NET SDK,
`VMatrix.Tracing.Sdk`. It says what the SDK does for you, what you still do
yourself, how to recover when batch numbering goes wrong, where the API
reference is, how to run the examples, and how to check that your
integration hashes and sends exactly what the backend and the portal expect.

There is one SDK package, on the sending side. Verifying a ticket is the job
of the verification portal, not of a package you install.

## Contents

- [What the SDK does, and what it does not](#what-the-sdk-does-and-what-it-does-not)
- [What you do yourself](#what-you-do-yourself)
- [Handling batch numbering errors](#handling-batch-numbering-errors)
- [API reference](#api-reference)
- [Running the examples](#running-the-examples)
- [Conformance test](#conformance-test)

## What the SDK does, and what it does not

| The SDK | You |
|---|---|
| Canonicalizes each ticket (RFC 8785 for JSON) and hashes it to a leaf, `keccak256(0x00 \|\| canonical bytes)` | Keep the ticket data, the ticket ids and the order of tickets in each batch |
| Checks a request against the event type before sending it; a bad field never leaves the process | Choose `draw`, `channel`, `sequence` and `value` for each batch |
| Sends `batch`, `lock`, `lockts` and the record types to `POST /api/anchors?eventType=...` | Decide when to retry, and retry with the same request |
| Returns the backend's answer, or its error code, unchanged | Store `anchorId`, `batchId`, `batchRoot` and `txHash`, and match them to webhooks |
| Builds one portal link per ticket | Get the RFC 3161 token for `lockts` from a timestamp authority |

The SDK does **not**:

- number batches for you, or check that numbering is continuous;
- retry a request on its own (see [Error handling](USAGE.md#6-error-handling) in the SDK guide);
- build the Merkle tree, check the `batchRoot` it gets back, or fetch proofs. The backend builds the tree, and the portal checks it against the chain.

## What you do yourself

1. **Keep one counter per draw and channel.** `sequence` is the number of the
   batch's first ticket. Tickets are numbered from 0 within a draw and
   channel, so the next batch starts at the previous `sequence` plus the
   previous count. See [The life of a draw, in code](USAGE.md#75-the-life-of-a-draw-in-code).
2. **Write the batch down before you send it.** Store the draw, channel,
   `sequence` and the ordered ticket hashes of a batch before the call, so
   that after a crash you can send the very same batch again.
3. **Move the counter only on `200`.** Do not take the next `sequence` until
   the previous batch has answered `200`. Send batches of one draw and
   channel one after the other, never in parallel.
4. **Retry with the same request.** After a network error, a timeout,
   `503`, or `504 CONFIRMATION_TIMEOUT`, send the identical request again:
   same `sequence`, same tickets, same order, same `value`. The backend
   answers the same tree and anchors nothing twice. Changing the order or the
   content makes a different batch.
5. **Never put a ticket in two batches.** The backend refuses a hash
   repeated inside one batch (`409 DUPLICATE_HASH`), but it does not stop the
   same hash from appearing in two batches. Keep a record of which batch each
   ticket went into, and resend a failed batch instead of moving its tickets
   into a new one.
6. **Lock after the last batch.** Send `lock` only once every batch of the
   draw has answered `200`. If you add a batch after a lock, lock again and
   get a new timestamp token: a token covers one drawRoot only.
7. **Match webhooks to your batches.** A webhook carries `anchorId` and
   `batchId`.

## Handling batch numbering errors

The backend stores `sequence` as sent and does not check it today. A gap or
an overlap is not refused when you send it; it shows later, to anyone who
reads the `Batched` logs of the draw. Keeping the numbering right is the
sender's job.

| What happened | What to do |
|---|---|
| The call timed out, or the connection dropped, and you do not know whether the batch was anchored | Send the identical request again. If it was anchored, you get the same answer and nothing new is anchored. |
| `504 CONFIRMATION_TIMEOUT` | The batch is stored and still anchoring. Send the identical request again. |
| Your process stopped after taking a `sequence` but before the answer | Read the batch you wrote down (step 2 above) and send it again unchanged. |
| A batch never answered `200` and later batches went out anyway | Send the missing batch with the `sequence` it had, so the numbering keeps no hole. Do not renumber the later batches. |
| You are not sure what the backend holds for a ticket | Look the ticket hash up: `GET /api/anchors?eventType=batch&hash=<ticket hash>` returns `state` and, in `meta`, the `draw`, `channel`, `sequence` and `count` of its batch. |
| A batch with a wrong `sequence` was anchored | It cannot be removed: the log is on chain. Stop sending for that draw and channel and contact VMatrix before you go on. |

## API reference

| What | Where |
|---|---|
| Event types, their fields and the life of a draw | [SDK guide, Event types](USAGE.md#7-event-types-tickets-and-other-schemas) |
| SDK classes and methods | [SDK guide, API reference](USAGE.md#9-api-reference) |
| SDK exceptions and error codes | [SDK guide, Error handling](USAGE.md#6-error-handling) |
| Verification links to the portal | [SDK guide, Verification links](USAGE.md#74-verification-links) |

## Running the examples

The SDK repository has one runnable example per step of a draw.

1. Install the .NET 10 SDK.
2. Clone the SDK and enter it:

   ```bash
   git clone https://github.com/1mtx/tracing-sdk-dotnet.git
   cd tracing-sdk-dotnet
   ```

3. Set the backend address in
   `examples/Tracing.Sdk.Examples/ExampleSettings.cs` (default
   `http://localhost:8080`), and pass your API key in `TRACING_API_KEY` so it
   is never written into the code.
4. Run the three tickets examples in the order of a draw:

   ```bash
   export TRACING_API_KEY=<your key>
   dotnet run --project examples/Tracing.Sdk.Examples -- tickets-batch
   dotnet run --project examples/Tracing.Sdk.Examples -- tickets-lock
   dotnet run --project examples/Tracing.Sdk.Examples -- tickets-records
   ```

   `tickets-batch` anchors two consecutive batches and prints one portal
   link per ticket. `tickets-lock` locks the draw and anchors a timestamp
   token (a mock one when no `resp.tsr` is present; never use a mock token for
   a real draw). `tickets-records` anchors `events`, `recon` and `subagent`
   records.

The examples can run again safely: `tickets-batch` resends the same tickets
and anchors nothing twice. Details are in the
[SDK guide, Runnable examples](USAGE.md#8-runnable-examples).

## Conformance test

Two checks, depending on how you integrate.

### You use the SDK

Run the SDK's own tests. They need the .NET 10 SDK only, no backend:

```bash
dotnet test tests/Tracing.Sdk.Tests
```

To run the two parts that matter most to a client on their own:

```bash
# hashing: RFC 8785 canonicalization and leaf hashes, against testdata/
dotnet test tests/Tracing.Sdk.Tests --filter "FullyQualifiedName~CanonicalizerTests|FullyQualifiedName~HashesTests"

# requests: batch, lock, lockts built as the backend expects; errors returned unchanged; no retry by the SDK
dotnet test tests/Tracing.Sdk.Tests --filter "FullyQualifiedName~TicketsFlowTests|FullyQualifiedName~TransportTests|FullyQualifiedName~FieldValidatorTests|FullyQualifiedName~TracingSdkTests"
```

All tests pass on a supported setup. A failure means your environment does
not hash or send the way the backend expects; do not anchor real tickets
until it passes.

### You hash tickets in your own code

If you compute ticket hashes yourself (for example in another language), your
code must produce the same bytes as the SDK. Check it against the shared
vectors in the SDK repository:

| File | Your code must |
|---|---|
| [`testdata/json-canonicalize.json`](../testdata/json-canonicalize.json) | Canonicalize every input to `expectedOutput`, byte for byte (RFC 8785) |
| [`testdata/keccak256.json`](../testdata/keccak256.json) | Hash every input to `expectedHash` with Keccak-256 (the original Keccak, not FIPS-202 SHA3-256) |

Then build a leaf as `keccak256(0x00 || canonical JSON bytes)`. For example,
the ticket `{ "n" : 1 }` canonicalizes to `{"n":1}`, and its leaf is:

```
0x72a611a14d569d9c11a9a77b5e0249eb9dd4bb55e4267c7fc3643a4bc03fc541
```

A quick check of the hash function alone: Keccak-256 of the empty string is
`0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`. If you
get `0xa7ffc6f8...`, you are using SHA3-256.

Last, compare with the SDK on your own tickets: hash a sample of real
tickets with `sdk.LeavesFor(Tickets.Batch, tickets)` and with your code, and
check that every leaf is equal.
