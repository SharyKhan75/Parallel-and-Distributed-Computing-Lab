# Distributed Task Offloading & Remote Rendering System

**Course:** CSC-334 Parallel and Distributed Computing
**Author:** _your name / roll number_

A client–server system that offloads video transcoding from a client laptop to a remote worker node over a local network. The client picks a video and sets the quality in a desktop GUI. The worker encodes it with FFmpeg and streams live progress back. The client then downloads the result, checks its integrity, and compares the time with rendering locally.

> **Hardware note:** Neither of my laptops has an NVIDIA GPU, so the worker ran in **CPU mode** (`libx264`). The NVENC path (`h264_nvenc`) is implemented and is selected automatically on a machine where FFmpeg has NVENC. It could not be tested here. See [Results](#results).

---

## Architecture

```
 CLIENT LAPTOP                                        WORKER NODE
 CustomTkinter GUI -> RenderClient --TCP :5001-->  server.py -> job queue -> FFmpeg
 progress bar / log  <-- live progress (2 Hz) <--                  |
 verified output     <-- result file + SHA-256 <------------------+
```

- **Protocol:** TCP, length-prefixed JSON messages, raw bytes for files (documented at the top of `protocol.py`).
- **Handshake:** the server sends a random nonce. The client replies with HMAC-SHA256 of the shared token, so the token never travels over the network. A protocol version check follows, then a latency probe (min/avg/max RTT).
- **Worker:** a headless daemon with a job queue. It runs FFmpeg with `-hwaccel cuda -c:v h264_nvenc` when NVENC is available, and falls back to `libx264` otherwise. Progress is parsed from `ffmpeg -progress`.
- **Robustness:** socket timeouts, SHA-256 checks on both upload and download with automatic retry, and reconnect with back-off. After a dropped connection the client re-attaches to the still-running job instead of restarting it.

## Repository structure

```
client/   gui.py (CustomTkinter app), net_client.py, cli.py, benchmark.py,
          make_test_videos.py, local_render.py, protocol.py
server/   server.py (render daemon), protocol.py
docs/     REPORT.md, screenshots/
results/  benchmark output (CSV, table, chart)
```

## Requirements

| Node | Needs |
|---|---|
| Both | Python 3.9+, FFmpeg (`winget install ffmpeg` on Windows) |
| Client | `pip install -r client/requirements.txt` (customtkinter, psutil, matplotlib) |
| Server | nothing extra (Python standard library only) |

## Network configuration

Both laptops must be on the same network: the same Wi-Fi, a phone hotspot, or a direct LAN cable.

**Option A: same Wi-Fi.** On the server run `ipconfig` and note its IPv4 address (mine was `192.168.100.22`).

**Option B: direct cable with static IPs** (admin Command Prompt; replace `Ethernet` with your adapter name):
```
netsh interface ip set address "Ethernet" static 192.168.1.1 255.255.255.0    (server)
netsh interface ip set address "Ethernet" static 192.168.1.2 255.255.255.0    (client)
```

**Firewall (server, admin Command Prompt):**
```
netsh advfirewall firewall add rule name="GPU Offload" dir=in action=allow protocol=TCP localport=5001
```
Set the network profile to **Private**. To verify from the client, run in PowerShell: `Test-NetConnection <server IP> -Port 5001`. `TcpTestSucceeded` should be `True`. Plain `ping` may time out because Windows blocks ICMP by default, which does not affect the application.

## How to run

**1. Start the server (worker laptop):**
```
cd server
python server.py --cpu --token mysecret
```
Drop `--cpu` on a machine with an NVIDIA GPU and an NVENC-enabled FFmpeg. The log should then say `mode: nvenc`.

**2. Start the client (other laptop):**
```
cd client
pip install -r requirements.txt
python gui.py
```
**3. In the app:**
1. Enter the server IP and the token, then click **Test Connection**. It shows the handshake result, latency and worker mode.
2. Click **Browse** and choose a video, then set the resolution, bitrate and preset.
3. Click **Render on Remote GPU** and watch the upload, queue, encode and download in the progress bar and log.
4. Click **Render Locally (compare)** to time the same job on the client and print the speedup.

**Headless alternative:**
```
python cli.py check  --host <server IP> --token mysecret
python cli.py render in.mp4 out.mp4 --host <server IP> --token mysecret --height 720 --bitrate 3000
```

## Benchmarking

```
cd client
python make_test_videos.py
python benchmark.py --host <server IP> --token mysecret testdata/*.mp4 --heights 720 --repeat 3
```
This writes `results/benchmark.csv`, `results/benchmark.md` and `results/speedup.png`. It reports the local time, remote time, speedup, upload and download times, network overhead, link throughput, and CPU or GPU utilisation.

## Results

Test environment: _client laptop model/CPU_, _server laptop model/CPU_, connected via _Wi-Fi / cable_. Both machines have integrated graphics only. The server encoded on CPU (`libx264`).

**Single test (my recording, 720p, 8000 kbps):**

| Metric | Value |
|---|---|
| Input size | 10 MB |
| Local render time | 5.1 s |
| Remote render time (upload + encode + download) | 13.7 s |
| Speedup (local / remote) | **0.37x** |
| Output size (remote / local) | 15.5 MB / 15.6 MB |

_Add here: the upload / encode / download breakdown from the log, and the benchmark table from `results/benchmark.md` with the chart `results/speedup.png`._

### Analysis

- **Remote was slower (0.37x).** Without an NVIDIA GPU, the worker has no hardware advantage. Both machines encode on CPU, so the remote path pays the full transfer cost on top of an equal or slower encode.
- **Network overhead dominates short jobs.** About 25 MB moved over the network (10 MB up, 15.5 MB down) for a clip that takes about 5 s to encode locally. Offloading only pays off when the compute time is much larger than the transfer time, as with long, high-resolution videos and a real GPU.
- **Output was larger than the input** because the target bitrate (8000 kbps) exceeds the source's bitrate. Lowering it shrinks the output and the download time.
- **Integrity:** every transfer was verified with SHA-256, and the output files played correctly.
- **Expected with a GPU:** NVENC encodes far faster than CPU encoding, so the speedup should rise above 1x as resolution and duration grow. This was not measured.

## Limitations

- The NVENC path was not tested (no NVIDIA GPU available).
- The link is authenticated but not encrypted. Use it only on a trusted network.
- Whole files are transferred. Pipelining the upload with the encode would reduce latency.
- One job is encoded at a time by default (`--workers N` allows more).

## Screenshots

_Add these to `docs/screenshots/` and link them here:_

![Connection test](docs/screenshots/connection.png)
![Render in progress](docs/screenshots/progress.png)
![Completed render log](docs/screenshots/done.png)
![Server console](docs/screenshots/server.png)
![Benchmark chart](results/speedup.png)
