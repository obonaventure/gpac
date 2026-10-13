# AMT support in GPAC's ROUTE input filter

## What this is

[AMT (Automatic Multicast Tunneling, RFC 7450)](https://www.rfc-editor.org/rfc/rfc7450) lets a
host receive multicast traffic over a network path that does not carry native multicast — a
typical situation for laptops, cloud VMs, and anything behind a non-multicast-capable ISP
link.  Instead of joining the multicast group directly, the host opens a UDP tunnel to an
**AMT relay** that does have real multicast connectivity, and the relay re-encapsulates the
multicast traffic and delivers it over that tunnel.

This document describes the integration of AMT into GPAC's ROUTE/ATSC-3 input filter
(`routein`), so that a `route://` URL can be received anywhere an AMT relay is reachable —
even on a laptop with no native multicast.

The AMT implementation lives in the `libamt/` submodule
(`git@forge.uclouvain.be:inl/multicast-quic/libamt.git`), a small, standalone C library that
implements the AMT gateway role (RFC 7450 §3.2).  The same library is also used by the `norm`
project in this repository.

---

## Files changed

| File | What changed |
|------|-------------|
| `.gitmodules` | Added `libamt` submodule at `libamt/`, remote `git@forge.uclouvain.be:inl/multicast-quic/libamt.git` |
| `configure` | New `--enable-route-amt` feature flag (disabled by default). Emits `#define GPAC_HAS_ROUTE_AMT` and `ENABLE_ROUTE_AMT=yes` into `config.mak` |
| `src/Makefile` | When `ENABLE_ROUTE_AMT=yes`: compiles `../libamt/amt.o` into libgpac and adds `-DGPAC_HAS_ROUTE_AMT -I$(SRC_PATH)/libamt` |
| `src/media_tools/route_dmx.c` | Core integration — see below |
| `include/gpac/route.h` | New public API: `gf_route_dmx_set_amt_relay()` |
| `src/filters/in_route.h` | Two new option fields in `ROUTEInCtx`: `amt_relay`, `amt_source` |
| `src/filters/in_route.c` | Two new filter options; calls `gf_route_dmx_set_amt_relay()` after creating the demuxer |

### Changes in `route_dmx.c` in detail

1. **Include guard**: `amt.h` and the POSIX socket headers it needs are included only when
   `GPAC_HAS_ROUTE_AMT` is defined, keeping the build side-effect-free when AMT is off.

2. **`amt_gw` field**: `struct __gf_routedmx` gains an `amt_gateway_t *amt_gw` field (under
   the guard).  It is `NULL` until `gf_route_dmx_set_amt_relay()` is called.

3. **Cleanup**: `gf_route_dmx_del()` calls `amt_gateway_close()` when the field is non-NULL.

4. **Refactored LCT processors**: the inner packet-parsing bodies of
   `dmx_process_service_route` and `dmx_process_service_dvb_flute` have been extracted into
   new static helpers `dmx_process_service_route_lct(routedmx, s, nb_read)` and
   `dmx_process_service_dvb_flute_lct(routedmx, s, nb_read)`.  These helpers assume
   `routedmx->buffer` is already populated and process exactly `nb_read` bytes from it.
   The original functions now just read from the socket and then call the helpers — the
   existing behaviour is unchanged.

5. **AMT drain loop in `gf_route_dmx_process()`**: at the top of the process function, before
   the native-socket `select()`, an `#ifdef` block checks `routedmx->amt_gw`:
   - Calls `amt_gateway_recv()` in a tight loop until it returns `< 0` (EAGAIN / drained).
   - Each returned payload is placed in `routedmx->buffer`; the destination address (`to`)
     is resolved to an IP string and port and matched against the registered services by
     `s->dst_ip` and `s->port`.
   - The matching service's `_lct` helper is called directly (bypassing the socket read).
   - After draining, `amt_gateway_next_timeout_ms()` is checked and `amt_gateway_timeout()`
     is called when due, to keep the tunnel's IGMPv3 membership state alive at the relay.
   - Returns `GF_OK` if at least one packet was delivered, `GF_IP_NETWORK_EMPTY` otherwise.
   - When `amt_gw` is set the native-socket path is **skipped entirely**, so there is no
     double-read.

6. **New API `gf_route_dmx_set_amt_relay()`**: resolves the relay, group and optional SSM
   source addresses via `getaddrinfo()` and calls `amt_gateway_open()` to perform the AMT
   handshake (Relay Discovery → Advertisement → Request → Membership Query → Membership
   Update).  Can be called more than once to switch relays.

---

## Building with AMT support

The `libamt` submodule must be checked out first:

```sh
git submodule update --init libamt
```

Then configure and build GPAC with:

```sh
./configure --enable-route-amt
make
```

The `--enable-route-amt` flag:
- defines `GPAC_HAS_ROUTE_AMT` in the generated `config.h`
- sets `ENABLE_ROUTE_AMT=yes` in `config.mak`, causing `src/Makefile` to compile
  `libamt/amt.c` directly into libgpac

Everything is guarded by `#ifdef GPAC_HAS_ROUTE_AMT`; a build **without**
`--enable-route-amt` is completely unaffected.

---

## Using AMT from the command line

The `routein` filter gains two new options:

| Option | Type | Description |
|--------|------|-------------|
| `amt_relay` | string | AMT relay host name or IPv4 address. When set, the filter uses the AMT tunnel instead of native multicast. |
| `amt_source` | string | Optional SSM (source-specific multicast) source address. Omit for ASM. |

### Basic example — ASM join via AMT

```sh
gpac -i route://239.1.2.3:4567:amt_relay=amt-relay.example.org [pipeline...]
```

### SSM join via AMT

```sh
gpac -i route://239.1.2.3:4567:amt_relay=amt-relay.example.org:amt_source=198.51.100.1 [pipeline...]
```

### DVB-MABR via AMT

```sh
gpac -i mabr://239.1.2.3:4567:amt_relay=amt-relay.example.org [pipeline...]
```

### Known public AMT relays

Several R&E networks operate public AMT relays.  The GEANT relay
(`amt-relay.geant.org`) is reachable from the public Internet and provides
access to multicast streams on the GEANT backbone.

```sh
gpac -i route://239.1.2.3:4567:amt_relay=amt-relay.geant.org [pipeline...]
```

---

## API (for embedders)

```c
/* include/gpac/route.h */

/*! Opens an AMT gateway tunnel so that multicast traffic can be received
 * over a path without native multicast.  Once set, gf_route_dmx_process()
 * drains the AMT gateway instead of reading from native multicast sockets.
 *
 * \param routedmx   the ROUTE demultiplexer
 * \param relay_host AMT relay host name or dotted-decimal IPv4 address
 * \param relay_port AMT relay UDP port (0 = default 2268)
 * \param group_ip   multicast group address (must match the dmx address)
 * \param group_port multicast group UDP port
 * \param source_ip  optional SSM source address; NULL for ASM
 * \return GF_OK on success, error code otherwise
 */
GF_Err gf_route_dmx_set_amt_relay(GF_ROUTEDmx *routedmx,
                                    const char *relay_host, u32 relay_port,
                                    const char *group_ip, u32 group_port,
                                    const char *source_ip);
```

Calling this function performs the full AMT handshake synchronously (with a
5-second timeout, 3 retries).  On success the demuxer's `gf_route_dmx_process()`
loop starts draining the tunnel automatically.  The gateway is torn down and freed
by `gf_route_dmx_del()`.

---

## Limitations / future work

- **ATSC 3.0 (`atsc://`)**: the AMT option is currently wired only for `route://` and
  `mabr://` URLs.  ATSC 3.0 uses a fixed bootstrap multicast address (`224.0.23.60:4937`)
  that would need to be tunnelled separately before service-specific groups are known.
- **IPv6**: `libamt` and this integration are IPv4-only.
- **Multiple services on different groups**: each service is matched by destination IP+port
  inside the process loop, so a single AMT tunnel (joined to one group) covers only that
  group.  Receivers that need multiple groups would need one `gf_route_dmx_set_amt_relay()`
  call per group — the API supports this but `in_route.c` currently exposes only one relay
  option.
