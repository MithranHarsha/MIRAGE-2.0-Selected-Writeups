# MIRAGE 2.0 — Running the Challenges Locally

This README explains how to unzip and run every challenge in this project on your
own laptop (macOS, Windows, or Linux) so you can practice before/without a live
event server. It only covers **deployment** — how to get each challenge running and
how to connect to it. Solving them is on you.

Challenge zips live under `Challenges/`, split into two category folders:
`Challenges/Binary_Exploitation/` and `Challenges/Reverse_Engineering/`.

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

All zip paths below are relative to `Challenges/`.

**`Binary_Exploitation/`**

| Challenge                                   | Zip                                                  | Type                                     |
| ------------------------------------------- | ---------------------------------------------------- | ---------------------------------------- |
| Overload Containment Field                  | `overload-containment-field.zip`                     | Offline binary (pwn)                     |
| Nova Corps Scramble                         | `nova-corps-scramble.zip`                            | Offline binary (logic)                   |
| Corvus Glaive's Edge                        | `corvus-glaives-edge.zip`                            | Offline binary (pwn)                     |
| Worldminds Archive                          | `worldminds-archive.zip`                             | Offline binary (pwn, ships its own libc) |
| Cull Obsidian's Brute Force                 | `MIRAGE_Binary_Ch7_Cull_Obsidian_FIXED_SOLVABLE.zip` | Network service, port 1667               |
| Ebony Maw's Persuasion Protocol             | `ebony-maw-challenge.zip`                            | Network service, port 1337               |
| Nova Prime's Last Stand                     | `Nova_Primes_Last_Stand_PARTICIPANT.zip`             | Network service, port 1669               |
| The Final Rip                               | `The_Final_Rip_DEPLOYMENT.zip`                       | Network service, port 1666               |
| Corrupted by Thanos's Attack (trap variant) | `thanos_keks.zip`                                    | Forensics (no execution)                 |

**`Reverse_Engineering/`**

| Challenge                    | Zip                                    | Type                       |
| ---------------------------- | -------------------------------------- | -------------------------- |
| Corrupted by Thanos's Attack | `Corrupted_by_Thanos_Attack.zip`       | Forensics (no execution)   |
| Scan Vision's Core           | `Scan_Visions_Core_PARTICIPANT_v2.zip` | Offline binary (RE)        |
| Shuri's Last Attempt         | `Shuris_Last_Attempt.zip`              | Network service, port 1710 |
| The Mind Stone               | `The_Mind_Stone_DEPLOYMENT.zip`        | Network service, port 1711 |

## 3. Standalone offline binaries

Overload Containment Field, Nova Corps Scramble, Corvus Glaive's Edge, and Scan
Vision's Core are simple stdin/stdout programs — no server, no networking. Unzip,
then run inside a throwaway Linux container:

```bash
# Example shown for Overload Containment Field — repeat the same pattern for the
# others, swapping the zip name / extracted folder / binary name.
cd Challenges/Binary_Exploitation
unzip overload-containment-field.zip -d overload-containment-field
cd overload-containment-field/P3-overload-containment-field

docker run --rm -it --platform linux/amd64 \
  -v "$PWD":/ch -w /ch debian:bookworm-slim \
  bash -c "chmod +x containment_field && ./containment_field"
```

Binary names / extracted sub-paths for the other three:

- **Nova Corps Scramble**: `nova-corps-scramble/P4-nova-corps-scramble/nova_corps_scramble`
- **Corvus Glaive's Edge**: `corvus-glaives-edge/P5-corvus-glaives-edge/corvus_glaives_edge`
- **Scan Vision's Core** (in `Challenges/Reverse_Engineering/`): `Scan_Visions_Core_PARTICIPANT_v2/participant/vision_core`

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

### 3.1 Worldminds Archive (special case: needs its own libc)

This one ships a specific `libc.so.6` alongside the binary — it's built for **Ubuntu
22.04's glibc 2.35**, so it needs a matching userland or it will crash with things
like "stack smashing detected". The simplest fix is to run it inside an actual
Ubuntu 22.04 container and point `LD_LIBRARY_PATH` at the challenge folder (which
contains the matching `libc.so.6`) instead of relying on the container's own libc:

```bash
cd Challenges/Binary_Exploitation
unzip worldminds-archive.zip -d worldminds-archive
cd worldminds-archive/P6-worldminds-archive

docker run --rm -it --platform linux/amd64 \
  -v "$PWD":/ch -w /ch ubuntu:22.04 \
  bash -c "chmod +x worldmind_archive && LD_LIBRARY_PATH=/ch ./worldmind_archive"
```

## 4. Network-service challenges

Six challenges ship a `Dockerfile` (Final Rip additionally ships a
`docker-compose.yml`) that stand up a TCP service, the same way they'd run on the
real event VM. You build/run them locally and then connect over the network — just
to `localhost` instead of a remote IP.

A generic connector, reusable for any of these once the container is up (swap the
port number):

```bash
python3 -c "
import socket
s = socket.create_connection(('127.0.0.1', <PORT>))
while True:
    data = s.recv(4096)
    if not data: break
    print(data.decode(errors='replace'), end='')
    s.sendall((input() + chr(10)).encode())
"
# or, if you have netcat: nc 127.0.0.1 <PORT>
```

### 4.1 Cull Obsidian's Brute Force (port 1667)

This one ships its own `START.sh`/`STOP.sh`, but they hardcode `sudo docker`, which
fails on Docker Desktop (macOS/Windows) since the daemon isn't reachable as root
there — only use the scripts as-is on Linux with a system Docker install. Everywhere
else, run the same steps manually instead:

```bash
cd Challenges/Binary_Exploitation
unzip MIRAGE_Binary_Ch7_Cull_Obsidian_FIXED_SOLVABLE.zip -d cull-obsidian
cd cull-obsidian

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build -t mirage-ch7-cull-obsidian ./challenge
docker run -d --platform linux/amd64 --name mirage-ch7-cull-obsidian \
  --restart unless-stopped -p 1667:1667 mirage-ch7-cull-obsidian
docker ps --filter name=mirage-ch7-cull-obsidian
# tear down: docker rm -f mirage-ch7-cull-obsidian
```

### 4.2 Ebony Maw's Persuasion Protocol (port 1337)

```bash
cd Challenges/Binary_Exploitation
unzip ebony-maw-challenge.zip -d ebony-maw-challenge
cd ebony-maw-challenge/ebony-maw

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build -t ebony-maw:latest .
docker run -d --platform linux/amd64 --name ebony-maw -p 1337:1337 ebony-maw:latest
docker ps --filter name=ebony-maw
# tear down: docker rm -f ebony-maw
```

### 4.3 Nova Prime's Last Stand (port 1669)

Ships its own deploy script:

```bash
cd Challenges/Binary_Exploitation
unzip Nova_Primes_Last_Stand_PARTICIPANT.zip -d nova-primes-last-stand
cd nova-primes-last-stand

chmod +x start.sh
DOCKER_DEFAULT_PLATFORM=linux/amd64 ./start.sh
# to use a different host port: ./start.sh 1669
# tear down: docker rm -f nova-prime-last-stand
```

### 4.4 The Final Rip (port 1666)

```bash
cd Challenges/Binary_Exploitation
unzip The_Final_Rip_DEPLOYMENT.zip -d The_Final_Rip_DEPLOYMENT
cd The_Final_Rip_DEPLOYMENT/The_Final_Rip_DEPLOYMENT

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose up -d --build
docker ps --filter name=mirage-final-rip
# tear down: docker compose down
```

### 4.5 Shuri's Last Attempt (port 1710)

```bash
cd Challenges/Reverse_Engineering
unzip Shuris_Last_Attempt.zip -d Shuris_Last_Attempt
cd Shuris_Last_Attempt/deployment_verified/challenge

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build -t shuri-last-attempt:latest .
docker run -d --platform linux/amd64 --name shuri-last-attempt -p 1710:1710 shuri-last-attempt:latest
docker ps --filter name=shuri-last-attempt
# tear down: docker rm -f shuri-last-attempt
```

### 4.6 The Mind Stone (port 1711)

```bash
cd Challenges/Reverse_Engineering
unzip The_Mind_Stone_DEPLOYMENT.zip -d The_Mind_Stone_DEPLOYMENT
cd The_Mind_Stone_DEPLOYMENT/deployment/challenge

DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build -t mind-stone-final:latest .
docker run -d --platform linux/amd64 --name mind-stone-final -p 1711:1711 mind-stone-final:latest
docker ps --filter name=mind-stone-final
# tear down: docker rm -f mind-stone-final
```

This one is statically linked, so it isn't sensitive to the host/container glibc
version.

## 5. Forensics challenges

`Corrupted_by_Thanos_Attack.zip` (in `Reverse_Engineering/`) and `thanos_keks.zip`
(in `Binary_Exploitation/`) don't involve running any binary at all — you're given a
raw `thanos_firmware_dump.bin` file to analyze with standard command-line tools. No
Docker or emulation is needed; this works natively on macOS, Linux, or Windows
(WSL/Git Bash).

```bash
cd Challenges/Reverse_Engineering
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
  binary needs; try a newer base image (e.g. `debian:trixie-slim` or
  `ubuntu:24.04`) in place of the one in that challenge's Dockerfile.
- **"Address already in use" / port conflict** — something else is already bound to
  that port; stop it, or map to a different host port, e.g. `-p 17111:1711` and
  connect to `17111` instead.
- **Stuck container from a previous attempt** — `docker ps -a`, then `docker rm -f
  <name>` to clear it before retrying.
