# 1. GraphQL stays, and starts earning its keep

Date: 2026-09-23
Status: proposed (needs the platform owner's sign-off)

## Context

Every product gateway exposes GraphQL through gqlgen. Measured across the
eight gateways on 2026-09-12:

- 703 root fields: 315 queries, 376 mutations, 12 subscriptions;
- **zero field resolvers** — every resolver hangs off `queryResolver`,
  `mutationResolver` or `subscriptionResolver`;
- 320 399 lines of generated gqlgen code, eight schemas, a codegen step in
  every build.

So the graph is never traversed. Each field is a flat call into one gRPC
method and back. The thing GraphQL is chosen for — asking for a shape and
letting the server assemble it — is not in use; what is in use is "typed RPC
over HTTP with a schema the web client can generate a client from".

The cost is not hypothetical: the generated code is most of each gateway's
diff noise, and a screen that needs five entities still issues five round
trips, exactly as a REST client would.

The upside of the current shape is real too: with no traversal there is no
N+1 problem by construction, the web client's codegen gives end-to-end
types, and subscriptions already work where they are needed (menu orders,
atrace QR).

## Options

1. **Use the graph.** Add relations to existing types (`Order.client`,
   `Task.assignee`, `Booking.master`), resolve them with dataloaders, and
   let one query replace the five a screen makes today. Cost: dataloader
   plumbing per gateway and an N+1 risk that has to be watched from then on.
2. **Drop GraphQL for typed RPC** (connect-go or gRPC-Web). The same typed
   client generation without the 320k lines and without the schema layer.
   Cost: rewriting eight gateways and the whole web client's data layer.
3. **Keep the shape, stop growing it.** No new flat root fields where a
   relation on an existing type would do; relations get added as the screens
   that need them come up.

## Decision

Option 3 now, option 1 incrementally.

Concretely, from this date:

- a new read does not become another root query when it is reachable as a
  field on a type the caller already has — `Client.events` rather than
  `clientEvents(clientId:)`;
- a screen that needs several entities is a candidate for one query with
  relations, and the first such relation in a gateway brings a dataloader
  with it;
- mutations stay flat: they are commands, and the graph adds nothing there.

Option 2 is explicitly not taken. The migration cost is weeks and buys no
capability the platform is short of; the 320k generated lines are a cost
that is paid once per build, not per request.

## Consequences

- The generated-code volume stays. It is regenerated in CI and checked
  against what is committed (audit A1/I2), so it is at least not drifting.
- The first gateway to grow a relation also grows a dataloader and the
  N+1 watch that comes with it. Until then the current "one field, one
  call" property holds and is worth keeping deliberately rather than by
  accident.
- If a future product needs a genuinely graph-shaped API (deep nesting,
  client-chosen shapes), that is the trigger to revisit option 1 wholesale
  rather than field by field.

## Revisit when

- a screen needs more than ~5 round trips and the relations to collapse
  them exist;
- or the generated code becomes a build-time problem rather than a
  diff-noise problem.
