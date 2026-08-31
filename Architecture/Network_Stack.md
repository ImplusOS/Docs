# Network Stack — ImplusOS

*Last reviewed: 2026-08-29 (post phase P5 of `Docs/Others/TODO_OS_Refactor.md`;
re-verified against `Kernel/Network/` and `Kernel/Drivers/Module/NetworkBuiltinDrivers.c`).*

## 1. Layers and where they live

```
Kernel/Network/
├── ethernet/Ethernet.c   -- frame tx/rx, ethertype dispatch to registered handlers
├── arp/ARP.c              -- address resolution, depends on Ethernet
├── ipv4.c                 -- IP layer, depends on Ethernet + ARP
├── icmp/ICMP.c            -- depends on IPv4
├── udp/UDP.c               -- depends on IPv4
├── tcp/TCP.c               -- depends on IPv4
├── dhcp/DHCP.c             -- depends on IPv4 + UDP
└── network_main.c          -- boot-time init order + per-tick polling
```

`network_stack_init()` (`Kernel/Network/network_main.c`) brings these up in
strict dependency order:

```c
ethernet_init();
arp_init(local_ip, netmask, gateway);
ipv4_init(local_ip, netmask, gateway);
udp_init();
tcp_init();
icmp_init();
dhcp_init();
```

This call order — and every cross-layer call inside `Kernel/Network` itself
(e.g. `ARP.c` calling `ethernet_send()`, `DHCP.c` calling `ipv4_send()`/
`udp_send()`) — is **unchanged by this refactor** and remains ordinary,
direct C function calls. See §3 for why.

## 2. DeviceRegistry visibility: `NetworkBuiltinDrivers.c`

`Docs/Others/TODO_OS_Refactor.md`'s original phase-P5 plan asked for each
protocol layer to become an individually loadable `.ELF` module (like
`FAT32_Driver.ELF`), the same way `Docs/Others/TODO_OS_Refactor.md` phase P1
did for storage/USB/audio drivers. Implementing that literally turned out to
require rewriting essentially all of `Kernel/Network`'s cross-layer calls in
one connected change (§3 explains why an incremental, one-layer-at-a-time
migration — the approach that worked well for FAT32/ISO9660/exFAT — isn't
available here). The approach actually taken is a **hybrid**, decided with
the project owner mid-implementation:

`Kernel/Drivers/Module/NetworkBuiltinDrivers.c` registers each layer into
`DeviceRegistry` under `DEVICE_TYPE_NET_PROTOCOL`, by its own name
(`"Ethernet_Driver"`, `"ARP_Driver"`, ...), with a **real, callable vtable**
built from that layer's actual public API — not a placeholder:

```c
typedef struct {
    bool     (*init)(void);
    bool     (*is_ready)(void);
    bool     (*send)(uint16_t ether_type, const uint8_t dst_mac[ETHERNET_ADDR_LEN],
                     const void *payload, uint16_t payload_len);
    bool     (*register_handler)(uint16_t ether_type, ethernet_type_handler_t handler);
    const char *deps[4];
} ethernet_builtin_driver_t;

static const ethernet_builtin_driver_t g_ethernet_driver = {
    .init = ethernet_init_void,
    .is_ready = ethernet_is_ready,
    .send = ethernet_send,
    .register_handler = ethernet_register_handler,
    .deps = { NULL },
};
```

...one such struct per layer (`arp_builtin_driver_t`, `ipv4_builtin_driver_t`,
`icmp_builtin_driver_t`, `udp_builtin_driver_t`, `tcp_builtin_driver_t`,
`dhcp_builtin_driver_t`), each with `deps[]` recording that layer's *actual*
dependencies as found in `Kernel/Network`'s real `#include` graph:

| Driver name | `deps[]` |
|---|---|
| `Ethernet_Driver` | *(none)* |
| `ARP_Driver` | `Ethernet_Driver` |
| `IPv4_Driver` | `Ethernet_Driver`, `ARP_Driver` |
| `ICMP_Driver` | `IPv4_Driver` |
| `UDP_Driver` | `IPv4_Driver` |
| `TCP_Driver` | `IPv4_Driver` |
| `DHCP_Driver` | `IPv4_Driver`, `UDP_Driver` |

`network_builtin_drivers_register()` is called from `kernel_main.c` right
after `network_stack_init()`, registry-visibility only — it never calls any
`*_init()` function itself and never changes when or whether one runs.

## 3. Why not genuine individual `.ELF` modules (yet)

A real loadable driver module is a separately linked, position-independent
`.ELF` that only reaches the rest of the kernel through the `driver_binary_t`
vtable handed to its `driver_module_init()` — see `Docs/Architecture/
Driver_Module_Guide.md`. Every one of `Kernel/Network`'s ~3200 lines currently
reaches its dependencies as ordinary extern C function calls compiled
directly into the kernel binary (`ARP.c` calls `ethernet_send()`, `ipv4.c`
calls `ethernet_send()` *and* `arp_resolve()`, `DHCP.c` calls `ipv4_send()`
*and* `udp_send()`, and so on). Converting even the lowest layer
(`Ethernet`) into a real separate module would require:

1. Ethernet.c itself becoming `-fPIC`, redirecting every kernel-resident call
   it makes (`memcpy`, `timer_ticks()`, the NIC-manager calls, ...) through
   the `driver_binary_t` vtable — the same macro-redirection pattern
   `Kernel/Drivers/FileSystem/FAT32/FAT32_Main.c` already uses under
   `#ifdef IMPLUS_DRIVER_MODULE`.
2. Every caller of `ethernet_send()`/`ethernet_init()`/etc. (`ARP.c`,
   `ipv4.c`) switching from an extern declaration to a
   `driver_manager_find(DEVICE_TYPE_NET_PROTOCOL, "Ethernet_Driver.ELF")`
   lookup, since the symbol no longer exists in the kernel's own link once
   Ethernet.c moves out of the kernel's `OBJS` list.

Step 2 cascades: IPv4 depends on Ethernet *and* ARP, so converting Ethernet
alone forces ARP's callers to also switch, which forces IPv4's callers to
switch, and so on — there is no independently-movable "first category" the
way `FileSystem/FAT32` was independently movable from `FileSystem/ISO9660`
during phase P1. Doing this safely means rewriting the call convention across
essentially the whole stack in one connected change, which was judged too
high-risk to do blind: this repository's regression testing for a change
like this is a QEMU boot smoke test (does the kernel reach steady state
without panicking), which cannot detect a DHCP lease that silently stops
being acquired, a TCP retransmit timer that silently stops firing, or similar
behavioral regressions confined to network traffic.

The hybrid in §2 is the result of that judgment call: it satisfies the
"individually named, individually inspectable driver per protocol layer"
part of the original request today, at effectively zero behavioral risk,
while leaving the vtables and `deps[]` in place as the concrete starting
point for a genuine incremental migration (or a full one, done carefully and
tested with real network traffic — DHCP lease acquisition, a sustained TCP
transfer — rather than a boot-only smoke test) in the future.

## 4. Runtime network flow (unchanged by this refactor)

```c
void network_stack_on_timer_tick(void) {
    driver_manager_nic_schedule_poll();     /* let the NIC driver queue RX/TX work */

    if (++g_timer_ticks_for_aging >= 60u) { /* ~4x/sec at 250Hz */
        ipv4_process_timer();               /* fragment reassembly aging */
        tcp_process_timer();                /* retransmit / keepalive timers */
    }

    if (nic_ready && !g_dhcp_nic_was_ready) {
        dhcp_init();
        dhcp_discover();                    /* fresh link: get a new lease */
    }
    if (nic_ready && dhcp_get_assigned_ip() == 0u) {
        /* no lease yet: retry DISCOVER every ~2s (DHCP_RETRY_INTERVAL_TICKS) */
    }
}
```

DHCP is driven centrally from here rather than by any specific NIC driver,
so it works uniformly whether the active NIC is a wired VirtIO-Net device or
an AX900 Wi-Fi association completing.
