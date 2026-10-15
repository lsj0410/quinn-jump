# quinn client Jump patch — edits explained

This patch lets the quinn example client perform a Jump handshake with the example server.
Jump defends against Harvest-Now-Decrypt-Later attacks by separating TLS key exchange material (K) from application ciphertext (C) across two network paths.
A one-path adversary sees either K or C, never both.
No quinn internals are modified in this patch.

---

## How to apply

```bash
git clone https://github.com/quinn-rs/quinn.git
cd quinn
git apply ../quinn-client-jump.patch
```

## How to run

```bash
# Terminal 1
cargo run --example server ./

# Terminal 2
cargo run --example client -- --jump --jump-addr ::1 https://localhost:4433/Cargo.toml
```

---

## 1. New CLI flags (`Opt` struct)

Two flags are added:

```rust
#[clap(long = "jump")]
jump: bool,

#[clap(long = "jump-addr", required_if_eq("jump", "true"))]
jump_addr: Option<std::net::IpAddr>,
```

**`--jump`** — enables the Jump protocol. When set, the client completes the
handshake on the initial socket, migrates to the address given by `--jump-addr`,
performs an RFC 9001 key update, and only then opens a QUIC stream.

**`--jump-addr <IP>`** — the local IP address to bind the Path 2 socket to.
The user provides an IP directly (e.g. `::1`, `10.0.0.2`) rather than an
interface name, so no platform-specific interface resolution is needed. clap
enforces this is present whenever `--jump` is set via `required_if_eq`.

---

## 2. Jump block in `run()`

Inserted between `connected at` and `open_bi()`. Only runs when `--jump` is set.

```rust
if options.jump {
    let path2_ip = options.jump_addr.unwrap();
    let path2_socket = std::net::UdpSocket::bind(SocketAddr::new(path2_ip, 0))?;
    let path2_local = path2_socket.local_addr()?;
    eprintln!("jump: migrating to {path2_local}");
    endpoint
        .rebind(path2_socket)
        .map_err(|e| anyhow!("rebind failed: {e}"))?;
    if endpoint.local_addr()?.ip() != path2_ip {
        return Err(anyhow!(
            "rebind verification failed: local addr is {}, expected {path2_ip}",
            endpoint.local_addr()?.ip()
        ));
    }
    conn.force_key_update();
    eprintln!("jump: migrated and key update queued on {path2_local}");
}
```

**Step by step:**

1. **Bind a new socket on Path 2** — `std::net::UdpSocket::bind(SocketAddr::new(path2_ip, 0))`
   creates a UDP socket bound to the user-provided IP with an ephemeral port
   chosen by the OS.

2. **Rebind the endpoint** — `endpoint.rebind(path2_socket)` atomically swaps the
   underlying UDP socket inside the quinn `Endpoint`. All subsequent sends go out
   through the new socket. This is the QUIC connection migration mechanism; no
   quinn internals need to be patched because `rebind` is a public API.

3. **Verify the rebind** — `endpoint.local_addr()?.ip()` is compared against
   `path2_ip`. On macOS, a full-tunnel VPN intercepts `bind()` at the kernel level
   and silently redirects all traffic back through the tunnel. This check detects
   that failure mode.

4. **Force a key update** — `conn.force_key_update()` queues an RFC 9001 key
   update. Quinn flips the Key Phase bit on the next outgoing packet and derives
   new AEAD keys using HKDF. After this point, any 1-RTT packet captured on Path 2
   is protected by keys that were never transmitted on Path 1.

5. **Gate application data** — `conn.open_bi()` is called only after this block
   completes. No QUIC stream is opened until migration and key update have both
   succeeded.

---

## Patch summary

| Location | Lines | Purpose |
|----------|-------|---------|
| `Opt` struct | +10 | `--jump`, `--jump-addr` flags |
| `run()` body | +20 | Jump block: bind, rebind, verify, key update |
| **Total** | **~30** | Single file, no new dependencies |
