# zenoh-vendor

The `zenoh` crate from crates.io with two routing fixes, used by
[zenoh-deno](https://github.com/jeff-hykin/zenoh-deno). Each patched version is a tag, with the crate at
the repository root:

| tag | crate |
|---|---|
| `zenoh-1.10.1-patch.2` | zenoh 1.10.1, both fixes |
| `zenoh-1.6.2-patch.2` | zenoh 1.6.2, both fixes |
| `zenoh-1.10.1-patch.1` | zenoh 1.10.1, the interest fix only |
| `zenoh-1.6.2-patch.1` | zenoh 1.6.2, the interest fix only |

```toml
[patch.crates-io]
zenoh = { git = "https://github.com/jeff-hykin/zenoh-vendor", tag = "zenoh-1.10.1-patch.2" }
```

Everything else is unchanged from the published crate; the interest fix's lines are marked `zenoh-deno patch`.

The second fix (patch.2) is upstream's for a startup deadlock,
[eclipse-zenoh/zenoh 646f2d1b](https://github.com/eclipse-zenoh/zenoh/commit/646f2d1b730e584a570015a77bee9f6db08be9d1)
(issue [2581](https://github.com/eclipse-zenoh/zenoh/issues/2581)), merged after 1.10.1 shipped: during
gossip autoconnect the RX thread held `ctrl_lock` and parked in `block_in_place` on the `StartConditions`
mutex while the Net runtime's only worker waited on `ctrl_lock`, so a session could freeze at startup.
It applies to 1.10.1 as is; on 1.6.2 it is hand-ported to the `p2p_peer` hat.
## The bug

A client (or peer) declares a publisher, so it sends the router an *interest* in that key's
subscribers. Once the router answers (`DeclareFinal`), `InterestState::set_finalized` marks the key's
resource with `subscriber_interest_finalized`. While the interest lives, the router keeps the client
told about subscribers, so the client can skip sending puts that no one wants.

When the publisher is undeclared, the interest is undeclared, but the flag stays on the resource.
The router then stops sending subscriber news, yet the client still trusts its stale (empty) list.
If the key's resource outlives the publisher and the subscribers change, every later `put` on that
key is dropped before it leaves the client.

Reproduced with plain zenoh (no Deno): client A declares a publisher on `k`, waits ~50 ms, undeclares it;
client B's subscriber on `k` is undeclared and a new one declared; A's `session.put("k")` never arrives.
Present in 1.6.2, 1.10.1 and zenoh's main branch (2026-10-06).

## The fix

`InterestState::unset_finalized` (in `src/net/routing/dispatcher/face.rs`) clears the flag again, and it is
called wherever a local interest is removed (`hat/client/interests.rs`, `hat/peer/interests.rs`, and in
1.6.2 `hat/p2p_peer/interests.rs`). Clearing it can only make the client send more, never less.
