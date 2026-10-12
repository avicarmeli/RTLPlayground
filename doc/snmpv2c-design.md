# Optional read-only SNMPv2c: implementation design

Design only; no listener, BER implementation, MIB dispatch or SNMP build flag is implemented in this branch. The eventual feature must be disabled by default and compile its translation units only when SNMP=1. No switch is accessed during preparation.

## Shared integration points

| Concern | Current source seam | Proposed boundary |
|---|---|---|
| UDP161 | udp_apps.c:udp_callbacks; uip/uip.c:uip_udp_new | Reserve one of ten UDP slots after initialization; bind local161 with a wildcard peer. Account for DHCP, syslog, DNS and NTP. |
| Reply peer | uip/uip.c receive lookup around1137; udp_send around1165; rtlplayground.c:handle_tx | Copy sender IP/port before overwriting uip_buf. Prefer an explicit reply-to helper whose peer override is consumed by udp_send; test that it does not narrow the wildcard listener. A temporary ripaddr/rport alternative requires a post-output restore hook, never restoration inside the callback before headers are generated. |
| Packet storage | uip_appdata/uip_buf, custom Realtek/VLAN LLH layout | Decode into bounded descriptors, stage response separately, then copy into uip_appdata. Never overwrite unread request data or emit syslog while holding shared packet storage. |
| System objects | machine.machine_name, VERSION_SW, hostname, monotonic ticks | sysDescr, sysName, sysUpTime; contact/location empty or bounded configured strings. Define agent-start epoch and atomic tick reads. Resolve sysObjectID ownership before advertising it; no invented enterprise number. |
| Interfaces | machine min_port/max_port, log_to_phys_port, is_sfp | Enumerate physical user ports only with stable front-panel ifIndex; eight local stable KeepLink interfaces, nine Hasivo/Horaco/official V3.2. Exclude CPU MAC9 initially. |
| State/speed | rtl837x_port.c stats/link readers; LINKS_STS0x63E8; PHY enable and MAC speed registers | Add typed, bounded value-returning readers; distinguish configured admin state, physical link and SFP presence. Avoid parsing console strings. |
| Counters | STAT_GET and httpd/page_impl.c counter selector/read helpers | Bounded register poll; snapshot high32 at0x0F64 and low32 at0x0F68 after selector0x0F60. Preserve native64-bit octets/packet totals as two words; validate selectors, latch/reset and discard semantics. No fabricated Counter64 from sparse low32 polling. |

Existing STAT_GET/SMI waits are unbounded. A bounded SNMP reader cannot claim a bounded latency until every underlying operation it invokes also returns on failure. Add these shared readers and UDP reply routing as separately tested prerequisites, before the agent.

## Protocol and processing bounds

Start with GET and GETNEXT, then add capped GETBULK. Numeric lexicographic OID traversal, scalar .0/table instances, noSuchObject/noSuchInstance/endOfMibView and tooBig behavior follow [RFC3416](https://www.rfc-editor.org/rfc/rfc3416). SET must never reach CLI/register writes; traps, notifications and dynamic allocation are excluded initially. Do not claim full MIB-II/IF-MIB conformance for a subset.

Use a definite-length iterative BER parser with parent-end bounds, no recursion. Reject truncation, indefinite/overflow lengths, OID overflow, unsupported versions/PDUs and excessive varbinds before hardware access. Initial complete-message cap484 bytes, at most8 input varbinds, bounded OID arcs and community length32, and a fixed per-request work/output cap. Positive unsigned BER values need correct sign padding; Counter64 encodes the two native words without requiring SDCC uint64 support. Wrong community silently drops; community is unset until configured, not logged, and only read access exists.

Prioritize ifHCIn/OutOctets and validated HC packet counts. Counter32 uses low32 wrap semantics. High-speed ifSpeed/ifHighSpeed, discontinuity timestamps and link-transition bookkeeping must follow the actual object definitions in [RFC2863](https://www.rfc-editor.org/rfc/rfc2863); unknown data is omitted rather than guessed.

## Fresh resource baseline and estimated budget

All measurements below are from fresh CI=1 builds with SDCC4.5.0 #15242, not historical artifacts. Full manifests remain outside public Git. The integrated Hasivo/Horaco builds have BANK1 free603, BANK2 free5652, BANK3 free27418, XDATA free33470, initial stack158 bytes. The separate private stable variant has BANK3 free16371; with optional SFP diagnostics it has BANK3 free13399, XDATA free33011 and initial stack140 bytes. These are static linker allocations, not runtime stack high-water measurements.

| New component | Estimated allocation / acceptance target |
|---|---|
| BER, MIB, transport implementation and constant OID tables | 8–9 KiB in BANK3 total initially; hard initial target9216 bytes leaves4183 bytes even beside private SFP diagnostics. This is an estimate requiring a compiled prototype. |
| Common hooks | At most256 code bytes; no new SNMP strings/code in BANK1; keep BANK2 glue below192 bytes. |
| Static response, descriptors, peer/community and scratch | About800–1024 XDATA bytes:484 response + up to160 descriptors +32 community + about96 peer/parser/counter state, plus alignment. |
| Internal RAM/stack | No persistent DATA allocation and no recursion; initial additional stack-depth target16 bytes, to be measured on host and later on non-production hardware. Do not treat linker stack size as high-water proof. |
| Disabled feature | No UDP slot, translation unit or static SNMP workspace allocated. |

The narrow BANK1 headroom rules out placing the agent in the HTTP bank. If the initial subset exceeds9 KiB, reduce scope or revise placement before enabling it with diagnostics; do not assume the older8–16 KiB estimate fits all combinations.

## Validation sequence

Host tests: malformed/fuzzed BER, length/OID/sign boundaries, overlapping input/output, exact errors and OID ordering; two managers with different source ports; wildcard listener still works after each reply; ARP miss and sequential polls; socket exhaustion; coexistence with DHCP/DNS/NTP/syslog; counter timeout/64-bit carry/reset cases; distinct eight/nine-port enumerations. Firmware CI builds for all profiles with feature off/on and optional diagnostics, per-bank/XDATA/stack deltas and image hashes. Physical validation is outside this phase and must use a spare/lab device; the production KeepLink must remain untouched.
