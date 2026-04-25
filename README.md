# DPI Engine — Deep Packet Inspection System

Real-time network packet analyzer that captures live traffic, performs 
Layer 7 protocol classification, and enforces application-level blocking rules.

Built in C++ with a multi-threaded pipeline architecture.

---

## What It Does

- Captures packets from `.pcap` files and classifies them by application 
  (YouTube, Facebook, DNS, HTTPS, etc.) using TLS SNI extraction
- Blocks traffic by IP, application type, or domain pattern
- Outputs filtered traffic to a new `.pcap` file with a processing report
- Scales via a configurable thread pool (Load Balancers × Fast Path workers)

---

## Architecture

[PCAP Reader]
│
▼ hash(5-tuple) % N
[Load Balancer Threads]      ← distribute flows consistently
│
▼ hash(5-tuple) % M
[Fast Path Worker Threads]   ← each owns its own flow table (no lock contention)
│
▼
[Output Queue → Writer Thread]

Key design decision: consistent hashing on the 5-tuple ensures all packets 
from the same TCP connection always reach the same Fast Path worker.
This eliminates the need for shared locks on the flow table.

---

## How SNI Extraction Works

HTTPS traffic is encrypted — but the TLS Client Hello sends the target 
domain in plaintext before encryption begins. The engine parses the 
TLS handshake byte-by-byte to extract the Server Name Indication (SNI) 
field, mapping it to an application type.

Once a flow is identified (e.g., YouTube), all subsequent packets in 
that 5-tuple are classified and blocked without re-inspection.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Packet capture | C++17, custom PCAP parser (no libpcap dependency) |
| Protocol parsing | Manual byte-level parsing of Ethernet/IP/TCP headers |
| SNI extraction | Custom TLS Client Hello parser |
| Concurrency | `std::thread`, `std::mutex`, `std::condition_variable` |
| Thread-safe queue | Custom bounded queue with producer-consumer signaling |
| Build system | CMake |

---

## Build & Run

```bash
# Clone
git clone https://github.com/Nandini905/dpi-engine
cd dpi-engine

# Build (requires g++ with C++17, Linux/macOS)
mkdir build && cd build
cmake .. && make

# Run — basic
./dpi_engine ../test_dpi.pcap output.pcap

# Run — with blocking rules
./dpi_engine ../test_dpi.pcap output.pcap \
  --block-app YouTube \
  --block-app TikTok \
  --block-domain facebook \
  --block-ip 192.168.1.50

# Configure thread pool
./dpi_engine input.pcap output.pcap --lbs 2 --fps 4
# Creates 2 LB threads × 4 FP threads = 8 processing threads

# Generate test traffic
python3 generate_test_pcap.py
```

---

## Sample Output
╔══════════════════════════════════════════════════════╗
║         DPI ENGINE v2.0 (Multi-threaded)             ║
╠══════════════════════════════════════════════════════╣
║  Total Packets : 77    Forwarded : 69    Dropped : 8 ║
╠══════════════════════════════════════════════════════╣
║  APPLICATION BREAKDOWN                               ║
║  HTTPS    39   50.6%  ##########                     ║
║  YouTube   4    5.2%  # (BLOCKED)                    ║
║  Facebook  3    3.9%                                 ║
╠══════════════════════════════════════════════════════╣
║  DETECTED SNIs                                       ║
║  www.youtube.com → YouTube (BLOCKED)                 ║
║  www.facebook.com → Facebook                         ║
║  github.com → GitHub                                 ║
╚══════════════════════════════════════════════════════╝

---

## Project Structure
dpi-engine/
├── include/
│   ├── pcap_reader.h          PCAP file I/O
│   ├── packet_parser.h        Ethernet/IP/TCP header parsing
│   ├── sni_extractor.h        TLS Client Hello + HTTP Host extraction
│   ├── types.h                FiveTuple, AppType, Flow structs
│   ├── rule_manager.h         IP/app/domain blocking rules
│   ├── connection_tracker.h   Stateful flow tracking
│   ├── load_balancer.h        LB thread — consistent hash dispatch
│   ├── fast_path.h            FP thread — DPI + rule enforcement
│   └── thread_safe_queue.h    Bounded queue with condition variables
├── src/
│   ├── main_working.cpp       Single-threaded version
│   └── dpi_mt.cpp             Multi-threaded version
├── generate_test_pcap.py      Generates test traffic
├── test_dpi.pcap              Sample capture
└── CMakeLists.txt

---

## What I Learned

- TCP/IP packet structure at the byte level — parsing without libraries
- TLS handshake internals — how SNI leaks domain names before encryption
- Lock-free flow table design using consistent hashing
- Producer-consumer thread coordination with bounded queues
- Trade-offs between single-threaded simplicity and multi-threaded throughput

---

## Future Work

- [ ] Live packet capture via raw sockets (currently PCAP files only)
- [ ] QUIC/HTTP3 support (SNI in UDP Initial packets)
- [ ] Web dashboard for real-time traffic visualization
- [ ] ML-based anomaly detection for unknown protocols
