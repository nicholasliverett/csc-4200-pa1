# PA1 — ETP/1 Client and Server

**Student:** Jackson Boyett (`Jackson-Boyett`) Nicholas Liverett (`nicholasliverett`)
**Course:** CSC 4200 — Fall 2026
**Language:** Python
**Submitted commit:** _the SHA you paste into Canvas_

## Build and run

```
make build
make run-server PORT=5000 ROOT=server-files      # in one terminal
./etp-client --host 127.0.0.1 --port 5000 --name YOURNAME list    # in another
```


## What works and what does not

Server tests
  S01  PASS   2 pt  HELLO handshake and HELLO_OK format
  S02  PASS   2 pt  LIST format, sorting and filtering
  S03  PASS   1 pt  GET a small file
  S04  PASS   1 pt  GET a 2 MiB file
  S05  PASS   2 pt  PUT 300 KB, digest, then GET it back
  S06  PASS   1 pt  PUT and GET empty files
  S07  PASS   1 pt  NOT_FOUND is non-fatal
  S08  PASS   1 pt  BAD_NAME blocks traversal; non-fatal
  S09  PASS   2 pt  requests arriving in fragments
  S10  PASS   2 pt  several requests in one write
  S11  PASS   1 pt  unknown TYPE: BAD_TYPE, payload skipped
  S12  PASS   1 pt  bad VERSION is fatal
  S13  PASS   1 pt  oversized LEN is fatal, nothing allocated
  S14  PASS   1 pt  request before HELLO is fatal
  S15  PASS   1 pt  BYE: BYE_OK, then the server closes
  S16  PASS   2 pt  8 clients served while 2 others stall
  S17  PASS   1 pt  survives clients that vanish mid-transfer
  S18  PASS   1 pt  restart on the same port (SO_REUSEADDR)

Client tests (a scripted peer plays the server)
  C01  PASS   1 pt  list: fragmented response printed exactly; sends BYE
  C02  PASS   2 pt  get: 200 KB in fragments written to disk
  C03  PASS   1 pt  put: payload layout and digest printed
  C04  PASS   1 pt  server ERROR: exit 1 and a stderr message
  C05  PASS   1 pt  digest mismatch: exit 3
  C06  PASS   0 pt  no server: exit 2 (not scored)

Evidence and documents
  - capture: 2 TCP connection(s); the GET session looks complete
  - ANALYSIS.md: every etp-facts field is filled in (values are checked at grading)

Repository (scored by the same rules at grading)
  build 3/3  (assumed: --no-build)
  required files 2/2
  commit history 3/3  (7 commits on 4 day(s); full marks: 5 or more after the starter commit, on 3 or more days)
  transport seam 2/2
  Build and repository: 10 / 10

Execution: 30 / 30 points
This is the Execution part of the rubric only. Evidence, design and documentation
are scored by people, using the bands in the PA1 brief.


## Files

| File | What it is |
|---|---|
| `src/server.py` |ETP/1 server: accept loop, one thread per session, request checks and dispatch, file store, logging |
| `src/client.py` | ETP/1 client: list/get/put sessions, prints responses, exit codes 0-3 |
| `src/transport.py` | import line selecting the transport_tcp.py file |
| `src/transport_tcp.py` | TCP transport: the only file using socket (listen, accept, connect, send_all, recv_exact, close) |
| `src/etp.py` | Protocol constants and message encode/read_message framing (8-byte header + payload)|
| `DESIGN.md` | design document |
| `AI_USE.md` | AI-use disclosure and log |
| `evidence/pa1-capture.pcap` | loopback capture of my PUT and GET sessions |
| `evidence/time-wait.txt` | `ss` output showing TIME-WAIT after my session |
| `evidence/ANALYSIS.md` | my capture analysis |

---

## Authorship statement

We are the authors of this submission. Except where this README or `AI_USE.md`
says otherwise, we wrote, or rewrote and understand, every part of it — code,
documents and evidence — and we can explain any line of it to the instructor or
GTA. We are responsible for its correctness, including any part a
generative AI tool helped us produce. Our use of AI tools is disclosed in full in
`AI_USE.md`. The capture and measurements in `evidence/` come from our own programs,
run by us.

**Signed:** Jackson Boyett, Nicholas Liverett  **Date:** 11/12/2026
