# Leviathan
> Reconstructed from Codex task [019f9d81-75da-7f93-a454-14abd9c8c7a1](thread://019f9d81-75da-7f93-a454-14abd9c8c7a1). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

- **Exact name:** `Leviathan`
- **Points shown:** `450`
- **Author:** `ravenspar`
- **Solves shown in the screenshot:** `0`
- **Exact description:**

  > A capture from a compromised host. Something left through the noise. The loud way out is a lie.

- **Provided challenge file:** `capture.pcap`, supplied to Codex inside:

  [leviathan.zip](C:/Users/kingg/Downloads/leviathan.zip)

- **Provided screenshot:**

  [challenge screenshot](C:/Users/kingg/AppData/Local/Temp/codex-clipboard-217f87ee-c295-4f13-a52d-d16c72d560bb.png)

## Category and status

- **Category:** Network forensics, covert-channel analysis, and cryptography.
- **Status:** **Unresolved**
- **Confirmed flag:** **No confirmed flag**

Two flag-shaped strings were submitted by the user and rejected. Neither may be reported as a solution.

## Artifact integrity and basic facts

```text
leviathan.zip size:      61,575,427 bytes
leviathan.zip SHA-256:   D467D4F9CC767DCDF9458C18BF07F78CF21B5963D7B285AD89147B919445404A
capture.pcap size:       94,433,948 bytes
capture.pcap SHA-256:    77ECC54998B06E12283DC940C92D18B826E8AC17354044281FDE122CC768F66F
```

The capture contains 431,063 packets spanning roughly eight hours. The compromised host is `10.10.10.37`, with DNS server `10.10.10.1`.

Observed traffic:

- 337,681 DNS/UDP packets
- 93,382 TCP packets
- 175,980 DNS queries across 51 suffixes
- 1,304 TCP flows: 1 HTTP flow and 1,303 fake-TLS flows on port 443

## High-level analysis

The obvious HTTP and DNS lanes contain deliberately convincing flag-shaped strings. The HTTP string is a loud Rickroll-style decoy. A highly regular DNS lane hides a second flag-shaped sentence after a repeating XOR, but the user submitted it and the platform rejected it. That second string is best treated as an instruction: the historical LEVIATHAN stream cipher may be relevant “beneath” the DNS entropy.

A correct LEVIATHAN implementation was built and independently validated against the official NESSIE test vectors, including a correction to a typo in the published prose specification. Applying it to the DNS material using the obvious thematic keys, common hashes, byte orders, and offsets did not recover a flag. Extensive payload, timing, DNS-response, record-size, entropy, and direct-XOR analyses also did not yield a confirmed solution. The remaining credible area is a quieter relationship among the TLS/header streams, DNS-derived key material, or a rejection-sampled statistical channel.

## Detailed procedure and findings

### 1. Parse the PCAP without relying on Wireshark

The PCAP was parsed in Python with `mmap` and `struct`, preserving timestamps, flow direction, IP/TCP/UDP fields, DNS records, TLS-shaped records, and payload boundaries.

The reusable parsed DNS corpus was saved as:

```text
leviathan_work\dns_records.pkl
```

### 2. Identify and reject the HTTP decoy

The only HTTP flow contains:

```text
pbctf{n3v3r_g0nna_g1v3_y0u_up}
```

This is the “loud way out” and was rejected. It is not the flag.

### 3. Isolate the anomalous DNS lane

The standout domain is:

```text
sync-telemetry-io.net
```

It carries:

- 1,800 A queries
- 30 unique 32-character Base32 labels
- Each unique label decodes to 20 bytes
- Each label repeats 60 times
- Concatenating the 30 unique blocks gives 600 bytes

XORing the first 50 bytes with repeating key:

```text
kraken
```

produces:

```text
pbctf{th3_l3v14th4n_lurks_b3n34th_th3_dns_3ntr0py}
```

The user submitted this and reported it wrong. It must be treated as a rejected candidate or an instruction, not a confirmed flag.

The rest of the DNS lane is statistically random-looking. Responses also appear synthetically generated:

- A records use TEST-NET prefixes `192.0.2.0/24`, `198.51.100.0/24`, and `203.0.113.0/24`.
- Response delays are uniformly selected from 1–40 ms.
- Transaction IDs, TTLs, and final address octets appear random.
- 155 of the 1,800 queries lack a response, consistent with random loss.

Batch timing, query order, response presence, qtype, TTL, transaction ID, delay, address-octet aggregates, row/column layouts, and bitplanes were tested without producing a flag.

### 4. Analyze the fake TLS cover

Each fake client hello contains:

- A TLS-shaped record
- 32 random-looking bytes
- A plaintext fake hostname
- An invalid handshake length

The following application records are valid-shaped but their payloads are uniformly high-entropy. Analysis covered:

- Per-flow and global outbound/inbound content
- Record lengths and length differences
- First/last bytes
- Byte XOR, sum, and bit counts
- Entropy, chi-square, compression ratio, printable ratio
- Raw payload bitplanes and per-flow summaries
- Direct XOR relationships between paired directions and records
- Correlation of DNS batch times with nearby TLS handshakes

No independently decodable flag was recovered.

### 5. Implement and verify the actual LEVIATHAN cipher

The title and rejected DNS sentence motivated implementing the historical NESSIE LEVIATHAN stream cipher.

The published prose specification contains a functional typo in its `D` mapping. The implementation that matches the authors' official vectors uses:

```text
D_z = 2x + y + 2z  (mod 2^32)
```

not `2x + y + z`.

The implementation was validated against both official vector sets:

```text
key: de c0 c0 c5 57 fa
words:
1861600e 88244832 2a6d8201 ffd0f37d
b8767ce6 e7bd8954 b3fc97f0 e88caba1
```

```text
key: 01 23 45 67 89 ab cd ef fe ca be ba be ba fe ca
words:
44a7742e ba1625e3 1c00e70a 71fffd3b
9aa8ea1f d02a5f08 ef52ffd7 8e3b31e7
```

That establishes that the local cipher code is correct; failure to decrypt is not explained by a broken implementation.

### 6. Test thematic keys and transforms

The 600 DNS bytes were tested under:

- Observed block order
- Reversed bytes
- Reversed blocks
- Fully reversed material
- Sorted labels
- Offsets around the 50-byte instruction boundary
- Big- and little-endian LEVIATHAN words
- Nearby word seeks

Candidate keys included:

- `kraken`
- `leviathan` in several cases
- `ravenspar`
- `entropy`
- fragments and the full rejected DNS sentence
- `sync-telemetry-io.net`
- other visible domain and decoy phrases
- MD5, SHA-1, SHA-256, and SHA-512 digests of those candidates

No flag or structurally convincing plaintext emerged.

### 7. Remaining unconfirmed avenues

The latest continuation created a more exhaustive TCP-option/timestamp and sequence-residual scanner, but its run was interrupted and `quiet_channels_output.txt` is empty. Therefore, TCP timestamps and the full quiet-header channel are **not** proven dead.

The most defensible next tests are:

1. Use the 600 DNS bytes, the 550 bytes after the rejected instruction, each 20-byte DNS block, and their hashes as LEVIATHAN keys against TLS application records.
2. Try record- or flow-reset keystream use, not only one global stream.
3. Exhaust TCP timestamp options, per-flow sequence residuals, IP-ID deltas, and inter-arrival quantization.
4. Test cross-direction record comparisons and one-bit-per-flow statistical/rejection-sampling channels.
5. Require a clean plaintext with a structurally complete flag before submission.

## Skills used

- `solve-challenge`
- `ctf-forensics`
- `pdf`
- The `ctf-forensics` network-traffic reference was also used.

## Tools, utilities, libraries, and services used

- PowerShell
- Python 3.12/3.14
- Python modules including `mmap`, `struct`, `collections`, `ipaddress`, `pickle`, `base64`, `hashlib`, `zlib`, `re`, `statistics`, `heapq`, and NumPy
- 7-Zip/tar-style ZIP extraction
- SHA-256 hashing
- Web search for the historical NESSIE LEVIATHAN submission and clarification of the specification typo
- PDF text extraction and page rendering
- Custom PCAP, DNS, TCP, and TLS parsers
- Entropy, chi-square, compression, bitplane, timing, aggregation, correlation, XOR, and stream-cipher analysis

## Rejected candidates and why

### HTTP decoy

```text
pbctf{n3v3r_g0nna_g1v3_y0u_up}
```

- Located in the single obvious HTTP flow.
- Directly matches the “loud way out is a lie” warning.
- Rejected and must not be resubmitted.

### DNS-derived sentence

```text
pbctf{th3_l3v14th4n_lurks_b3n34th_th3_dns_3ntr0py}
```

- Cleanly obtained by XORing the first 50 DNS bytes with repeating `kraken`.
- Submitted by the user and rejected by the challenge platform.
- Best interpreted as a hint toward LEVIATHAN and the DNS entropy, not the final answer.

### Other unsuccessful approaches

- Simple repeating XOR, additive shifts, Base32/Base64 variants, transposition, bitplanes, batch timing, answer-field aggregation, and qtype encoding
- HMAC/hash chunk checks
- Direct reuse of TLS pads between directions or records
- Entropy/chi-square/compressibility outlier selection
- Obvious thematic LEVIATHAN keys and hash-derived keys

## Surviving local artifacts

- [capture.pcap](C:/Users/kingg/Desktop/codex/leviathan_work/capture.pcap)
- [DNS parse cache](C:/Users/kingg/Desktop/codex/leviathan_work/dns_records.pkl)
- [verified LEVIATHAN implementation](C:/Users/kingg/Desktop/codex/leviathan_work/leviathan_impl.py)
- [LEVIATHAN-over-DNS test harness](C:/Users/kingg/Desktop/codex/leviathan_work/try_leviathan_dns.py)
- [LEVIATHAN test output](C:/Users/kingg/Desktop/codex/leviathan_work/try_leviathan_dns_output.txt)
- [TCP field analysis](C:/Users/kingg/Desktop/codex/leviathan_work/tcp_analyze.py)
- [TLS record analysis](C:/Users/kingg/Desktop/codex/leviathan_work/tls_records.py)
- [TLS flow outlier analysis](C:/Users/kingg/Desktop/codex/leviathan_work/flow_outliers.py)
- [DNS batch/answer analysis](C:/Users/kingg/Desktop/codex/leviathan_work/batch_answers.py)
- [payload bitplane analysis](C:/Users/kingg/Desktop/codex/leviathan_work/payload_bits.py)
- [unfinished quiet-channel scanner](C:/Users/kingg/Desktop/codex/leviathan_work/quiet_channels.py)
- [NESSIE specification PDF](C:/Users/kingg/Desktop/codex/leviathan_work/leviathan_cipher_src/leviathan/B/specification.pdf)

## Diagram recommendation

**Yes.** A branch diagram is useful because it separates validated decoys from the unresolved carrier:

```text
capture.pcap
  |-- HTTP/80
  |     `-- obvious Rickroll flag --> rejected decoy
  |
  |-- DNS/53
  |     `-- sync-telemetry Base32 --> XOR "kraken"
  |           `-- LEVIATHAN/DNS sentence --> rejected as flag; retained as hint
  |
  `-- 1,303 fake-TLS flows + TCP metadata
        |-- payload/record entropy and direct transforms --> no flag
        |-- verified LEVIATHAN decryption attempts --> no flag
        `-- full header/timestamp/statistical covert channel --> unresolved
```

---

## Final High-Level Overview

The PCAP contains an obvious HTTP decoy and a second DNS-derived flag-shaped instruction that the checker rejected. DNS, fake-TLS, timing, payload, and verified LEVIATHAN-cipher passes did not produce an accepted result, while the final TCP quiet-channel scan was interrupted. No flag is confirmed.
