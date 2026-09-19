# MIRAGE 2.0 — Running the Challenges Locally

This README explains how to unzip and run every challenge in this folder on your own
laptop (macOS, Windows, or Linux) so you can practice before/without a live event
server. It only covers **deployment** — how to get each challenge running and how to
connect to it. Solving them is on you.

## 1. Prerequisites

- **Docker Desktop** (macOS/Windows) or **Docker Engine** (Linux) — all challenge
  binaries are compiled for **Linux x86-64**. Docker is the easiest way to run them
  on any OS, including Apple Silicon Macs.
- **Python 3** — used below as a portable `nc` replacement and for the forensics
  challenges. Any recent Python 3 works; `nc`/`ncat`/`socat` work equally well if you
  prefer them.
- `unzip` (preinstalled on macOS/Linux; use "Extract All" on Windows, or 7-Zip).

### Apple Silicon / ARM Macs — read this first

Every provided binary is `x86-64` Linux. Docker Desktop can run these via emulation,
but **you must force the `linux/amd64` platform explicitly** — otherwise Docker will
silently pull an `arm64` base image, the x86-64 binary won't have matching libraries,
and you'll see errors like `rosetta error: failed to open elf` or the container will
just exit. Two ways to force it:

```bash
# Option A: set it once per shell session
export DOCKER_DEFAULT_PLATFORM=linux/amd64

# Option B: pass --platform linux/amd64 on every `docker run`/`docker build`
# (already included in every command below)
```

If you're on an Intel Mac, Windows, or a native x86-64 Linux box, the `--platform
linux/amd64` flags below are harmless no-ops — leave them in.

## 2. Quick reference

| Challenge | Zip | Type |
|---|---|---|---|
| P3 — Overload Containment Field | `P3-overload-containment-field.zip` | Offline binary (pwn) | 
| P4 — Nova Corps Scramble | `P4-nova-corps-scramble.zip` | Offline binary (logic) | 
| P5 — Corvus Glaive's Edge | `P5-corvus-glaives-edge.zip` | Offline binary (pwn) | 
| P6 — Worldminds Archive | `P6-worldminds-archive.zip` | Offline binary (pwn, ships its own libc) | 
| Shuri's Last Attempt | `Shuris_Last_Attempt_PARTICIPANT.zip` | Offline binary (RE) | 
| The Final Rip | `The_Final_Rip_DEPLOYMENT.zip` | Network service, port 1666 | 
| The Mind Stone | `The_Mind_Stone_DEPLOYMENT.zip` | Network service, port 1711 | 
| Corrupted by Thanos's Attack | `Corrupted_by_Thanos_Attack.zip` | Forensics (no execution) | 
| Corrupted by Thanos's Attack (v2) | `thanos_keks.zip` | Forensics (no execution) |

## 3. Standalone offline binaries (P3, P4, P5, Shuri)

These are simple stdin/stdout programs — no server, no networking. Unzip, then run
inside a throwaway Linux container:

```bash
# Example shown for P3 — repeat the same pattern for P4, P5, and Shuri,
# just swap the zip name / binary name.
unzip P3-overload-containment-field.zip -d P3-overload-containment-field
cd P3-overload-containment-field/P3-overload-containment-field

docker run --rm -it --platform linux/amd64 \
  -v "$PWD":/ch -w /ch debian:bookworm-slim \
  bash -c "chmod +x containment_field && ./containment_field"
```

Binary names / extracted paths for the other three:

- **P4**: `P4-nova-corps-scramble/P4-nova-corps-scramble/nova_corps_scramble`
- **P5**: `P5-corvus-glaives-edge/P5-corvus-glaives-edge/corvus_glaives_edge`
- **Shuri**: `Shuris_Last_Attempt_PARTICIPANT/challenge/shuris_last_attempt`

If you'd rather work interactively inside the container (to poke around with
`objdump`, `gdb`, `strace`, etc. instead of just running the binary once), drop the
`bash -c "..."` part and get a shell instead:

```bash
docker run --rm -it --platform linux/amd64 -v "$PWD":/ch -w /ch debian:bookworm-slim bash
# then inside the container:
chmod +x <binary>
apt-get update && apt-get install -y gdb strace binutils   # optional, if you want them
./<binary>
```

On native Linux (x86-64) you can skip Docker entirely: `chmod +x <binary> &&
./<binary>` works directly.

### 3.1 P6 — Worldminds Archive (special case: needs its own libc)

P6 ships a specific `libc.so.6` alongside the binary — it's built for **Ubuntu
22.04's glibc 2.35**, so it needs a matching userland or it will crash with things
like "stack smashing detected". The simplest fix is to run it inside an actual
Ubuntu 22.04 container and point `LD_LIBRARY_PATH` at the challenge folder (which
contains the matching `libc.so.6`) instead of relying on the container's own libc:

```bash
unzip P6-worldminds-archive.zip -d P6-worldminds-archive
cd P6-worldminds-archive/P6-worldminds-archive

docker run --rm -it --platform linux/amd64 \
  -v "$PWD":/ch -w /ch ubuntu:22.04 \
  bash -c "chmod +x worldmind_archive && LD_LIBRARY_PATH=/ch ./worldmind_archive"
```

## 4. Network-service challenges (Final Rip, Mind Stone)

These ship a `Dockerfile` (and, for Final Rip, a `docker-compose.yml`) that stand up
a TCP service, the same way they'd run on the real event VM. You build/run them
locally and then connect over the network — just to `localhost` instead of a remote
IP.

### 4.1 The Final Rip (port 1666)

```bash
unzip The_Final_Rip_DEPLOYMENT.zip -d The_Final_Rip_DEPLOYMENT
cd The_Final_Rip_DEPLOYMENT/The_Final_Rip_DEPLOYMENT

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose up -d --build
docker ps --filter name=mirage-final-rip
```

> **Known issue with the shipped Dockerfile:** it builds `FROM debian:bookworm-slim`
> (glibc 2.36), but the `final_rip` binary requires **glibc ≥ 2.38** and will fail
> to start (`version 'GLIBC_2.38' not found`) on that base image. If you hit this,
> edit `challenge/Dockerfile` and change the first line to a newer base image before
> rebuilding, e.g.:
> ```
> FROM debian:trixie-slim
> ```
> (or `FROM ubuntu:24.04`) — both were confirmed to run the binary correctly. This
> looks like it will also bite the real event deployment, not just local testing —
> worth patching before the event.

Connect once it's up:

```bash
python3 -c "
import socket
s = socket.create_connection(('127.0.0.1', 1666))
while True:
    data = s.recv(4096)
    if not data: break
    print(data.decode(errors='replace'), end='')
    s.sendall((input() + chr(10)).encode())
"
# or, if you have netcat: nc 127.0.0.1 1666
```

Tear down when done: `docker compose down`.

### 4.2 The Mind Stone (port 1711)

```bash
unzip The_Mind_Stone_DEPLOYMENT.zip -d The_Mind_Stone_DEPLOYMENT
cd The_Mind_Stone_DEPLOYMENT/deployment/challenge

docker build --platform linux/amd64 -t mind-stone-local:latest .
docker run -d --platform linux/amd64 --name mind-stone-local -p 1711:1711 mind-stone-local:latest
docker ps --filter name=mind-stone-local
```

Connect the same way as Final Rip, pointed at port 1711:

```bash
python3 -c "
import socket
s = socket.create_connection(('127.0.0.1', 1711))
while True:
    data = s.recv(4096)
    if not data: break
    print(data.decode(errors='replace'), end='')
    s.sendall((input() + chr(10)).encode())
"
# or: nc 127.0.0.1 1711
```

This one is statically linked, so it isn't sensitive to the host/container glibc
version — the Dockerfile as shipped works fine.

Tear down when done: `docker rm -f mind-stone-local`.

## 5. Forensics challenges (Corrupted by Thanos's Attack, thanos_keks)

These two don't involve running any binary at all — you're given a raw
`thanos_firmware_dump.bin` file to analyze with standard command-line tools. No
Docker or emulation is needed; this works natively on macOS, Linux, or Windows
(WSL/Git Bash).

```bash
unzip Corrupted_by_Thanos_Attack.zip -d Corrupted_by_Thanos_Attack
cd Corrupted_by_Thanos_Attack/participant

file thanos_firmware_dump.bin
xxd thanos_firmware_dump.bin | less
strings thanos_firmware_dump.bin | less
python3   # for anything that needs actual parsing/decoding
```

Repeat the same for `thanos_keks.zip` (its README explicitly warns that some of the
strings you'll find are decoys — don't trust a flag-shaped string just because
`strings` found it).

## 6. Troubleshooting

- **"Cannot connect to the Docker daemon"** — Docker Desktop isn't running; start it
  and wait ~20-30s before retrying.
- **"rosetta error: failed to open elf..." or the container exits immediately** — you
  forgot `--platform linux/amd64` (or `DOCKER_DEFAULT_PLATFORM`) on an Apple Silicon
  Mac; see §1.
- **"Permission denied" running the binary** — you're on native Linux and forgot
  `chmod +x <binary>`.
- **"version `GLIBC_x.xx' not found"** — the container's glibc is older than the
  binary needs; use a newer base image (see §4.1 for Final Rip specifically).
- **"Address already in use" / port conflict** — something else is already bound to
  that port; stop it, or map to a different host port, e.g. `-p 17111:1711` and
  connect to `17111` instead.
- **Stuck container from a previous attempt** — `docker ps -a`, then `docker rm -f
  <name>` to clear it before retrying.
