# protocols — reverse engineering what talks over a wire (or in a file)

> Merged from: wshobson/agents protocol-reverse-engineering (capture/dissection/documentation workflow, boofuzz), zhaoxuya520/reverse-skill (packet-capture routing). Credit in ATTRIBUTION.md.

Protocol RE serves interoperability, debugging, and security research on systems you are authorized to analyze. The same workflow applies to wire protocols and to undocumented file formats - both are "message grammars with fields".

## Capture

```bash
tshark -i eth0 -w capture.pcap                 # capture to file
tshark -i eth0 -f "port 443" -w capture.pcap    # BPF capture filter
tcpdump -i eth0 -s 0 -w capture.pcap            # full-packet snap length
mitmproxy --mode transparent -p 8080            # HTTP(S) interception (authorized targets)
```

Capture multiple sessions of the same flow - one session shows one path; several reveal the constant-vs-variable fields. For TLS-protected traffic on your own test clients, log the pre-master secret (`SSLKEYLOGFILE`) and load it in Wireshark's TLS preferences rather than fighting the crypto.

## Dissect

```bash
tshark -r capture.pcap -q -z conv,tcp          # conversation statistics
tshark -r capture.pcap -q -z io,phs            # protocol hierarchy
tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e tcp.port
tshark -r capture.pcap -Y "tcp.port == 8888" -T json > flow.json
# Wireshark: Follow > TCP Stream - the single highest-value view for unknown protocols
```

Scapy for scripted dissection: `rdpcap()`, walk layers, inspect `Raw` payloads, reassemble streams. Compare messages pairwise: what is constant (magic, version, type enums), what varies (lengths, counters, session ids), what looks derived (checksums, MACs).

## Identify

| Signature | Protocol |
| --- | --- |
| `HTTP/1.` / `GET ` / `POST ` | HTTP |
| `0x16 0x03` record header | TLS/SSL |
| UDP 53 + header shape | DNS |
| `SSH-2.0` banner | SSH |
| `*` RESP prefix | Redis |
| `\xff\x53\x4d\x42` | SMB |
| BSON header shape | MongoDB |

Generic header grammar to expect in an unknown binary protocol:

```
+-----------------------------------+
|  magic / signature                |
+-----------------+-----------------+
|  version        |  flags          |
+-----------------+-----------------+
|  length         |  message type   |
+-----------------+-----------------+
|  sequence / session id            |
+-----------------------------------+
|  payload ...                      |
+-----------------------------------+
```

## The workflow

1. **Capture** several sessions, different scenarios (login, query, error)
2. **Find message boundaries** - magic numbers, length fields, framing
3. **Map the structure** - fixed header vs variable payload
4. **Identify fields** - compare samples; variable-with-pattern = counter, constant-per-session = session id, position-dependent = sequence
5. **Document the format** - a written spec, field by field
6. **Validate** - implement a parser and a generator; round-trip real messages through both
7. **Fuzz** (authorized targets) - boofuzz against a test instance:

```python
from boofuzz import *
session = Session(target=Target(connection=TCPSocketConnection("labhost", 8888)))
s_initialize("HELLO")
s_static(b"\x50\x52\x4f\x54")        # magic
s_word(1, name="version"); s_word(0x01, name="type")
s_size("payload", length=4)
s_block_start("payload")
s_dword(0x12345678, name="client_id"); s_word(0, name="flags")
s_block_end()
session.connect(s_get("HELLO")); session.fuzz()
```

## Replay and modify (authorized targets)

```python
from scapy.all import *
packets = rdpcap("capture.pcap")
for pkt in packets:
    if pkt.haslayer(Raw):
        pkt[Raw].load = pkt[Raw].load.replace(b"client", b"CLIENT")
        del pkt[IP].chksum; del pkt[TCP].chksum   # let scapy recompute
        send(pkt)
```

Replay teaches protocol state machines fast: drop message 3 and watch whether message 4 is accepted (stateless framing vs a stateful session), alter the type byte and watch error responses (they enumerate the type space for you).

## File formats

The same dissection applies to undocumented file formats: magic header, version, offsets/lengths as "message boundaries", records as messages. Multi-sample comparison finds the constant grammar; a parser+generator round-trip validates it; fuzzing the parser (with a harness, on your copy) probes robustness.

## Documentation standard

The output is a written specification: fields with offsets, widths, endianness, enums with observed values, and the state machine you inferred (which message follows which). A Wireshark Lua dissector is the gold-standard artifact for wire protocols - it forces precision and makes every future capture readable.
