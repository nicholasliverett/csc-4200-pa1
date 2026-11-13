# PA1 Capture Analysis

_Your capture, `evidence/pa1-capture.pcap`, holds two ETP sessions from your own
programs: a PUT, then a GET of the same file. This analysis is about the **GET
session** unless a question says otherwise. Use Wireshark (or tshark) to read the
capture. The numbers must come from your capture: they are checked against it._

## How I captured it

_Paste the exact commands you ran (capture, server, client, ss), and the size of
the file you transferred._

```
   mkdir -p evidence scratch
   head -c 262144 /dev/urandom > scratch/photo.bin   # 256 KiB
   ./etp-server --port 5000 --root server-files    # First Terminal

   sudo tshark -i lo -s 200 -f "tcp port 5000" -w evidence/pa1-capture.pcap # Second Terminal


   C="./etp-client --host 127.0.0.1 --port 5000 --name Jackson"   # Third Terminal
   $C put scratch/photo.bin photo.bin
   $C get photo.bin scratch/copy.bin
   cmp scratch/photo.bin scratch/copy.bin && echo identical
   ss -tan state time-wait '( sport = :5000 or dport = :5000 )' >
   evidence/time-wait.txt
```

## Facts

_Fill in every value from **your** capture, for the **GET** session. Integers in
decimal. Sequence numbers are the **absolute** (raw) values — in Wireshark, turn
off "relative sequence numbers" (Edit > Preferences > Protocols > TCP) or read
the "Sequence Number (raw)" field. With tshark, add
`-o tcp.relative_sequence_numbers:FALSE`. Do not change the key names or the
`etp-facts` label: a script reads this block._

```etp-facts
server_port: 5000
client_port: 48098
client_isn:  3067228580               # raw sequence number in the client's SYN
server_isn:  1848765859          # raw sequence number in the server's SYN/ACK
client_mss:  65495            # MSS option value in the client's SYN, in bytes
server_to_client_payload_bytes: 262175       # all TCP payload bytes the server sent in this session (HELLO_OK + GET_OK + BYE_OK)
server_to_client_data_segments: 9     # server-to-client frames with payload, across all three responses, counted as captured
first_fin_from: server                 # server or client
```

## Q1. The handshake

_Give the frame number, flags, and raw sequence and acknowledgment numbers of the
three handshake segments of the GET session. Explain why the acknowledgment number
in the third segment is what it is. What MSS did the client advertise, and what on
your machine determines that value? 

| Frame | Flags | Seq (raw) | Ack (raw) |
|---|---|---|---|
| 25 | SYN | 3067228580 | 0 |
| 26 | SYN, ACK | 1848765859 | 3067228581 |
| 27 | ACK | 3067228581 | 1848765860 |

The third segment’s ack is 1848765860 because the server’s SYN (seq 1848765859) consumed one sequence number. The MSS is still 65495, derived from the loopback MTU of 65536 minus 40 bytes of headers.

## Q2. Your protocol on the wire

_Find the segment that carries your client's GET request. Give its frame number,
paste the first 8 payload bytes in hex, and decode each header field (VERSION,
TYPE, STATUS, RESERVED, LENGTH). Do the same for the header of the server's
GET_OK. Does GET_OK's LENGTH equal the size of your file? Show the arithmetic that
connects `server_to_client_payload_bytes` to the three responses the server sent in
this session.

The GET request is in frame 32. Its first 8 payload bytes are 01 03 00 00 00 00 00 09, followed by the name photo.bin:

VERSION = 1
TYPE = 0x03 (GET)
STATUS = 0
RESERVED = 0
LENGTH = 9 (the length of photo.bin)

The GET_OK header is in frame 33: 01 83 00 00 00 04 00 00:

VERSION = 1
TYPE = 0x83 (GET_OK, i.e. GET | 0x80)
STATUS = 0
RESERVED = 0
LENGTH = 0x00040000 = 262,144

LENGTH equals the size of the file, 262,144 bytes (256 KiB).

server_to_client_payload_bytes is the sum of the three responses:

HELLO_OK: 8 header + 7 payload (Jackson) = 15
GET_OK: 8 header + 262,144 data = 262,152
BYE_OK: 8 header + 0 = 8
Total: 15 + 262,152 + 8 = 262,175

## Q3. A byte stream, not messages

_How many segments carried the **GET_OK response** (not counting the segments
that carried HELLO_OK and BYE_OK), and what were their payload sizes? Were they
all the same size? Were they all the MSS? Did any one segment carry the
end of one ETP message and the start of the next, or did one ETP message span
several segments? Explain what this means for code that reads an ETP message, and
point to the place in your own code that deals with it.

| Frames | Payload per frame |
|---|---|
| 33, 34 | 32,768 |
| 36 | 61,440 |
| 38, 41 | 47,616 |
| 39 | 22,016 |
| 42 | 17,928 |

These add up to 262,152 bytes, exactly the GET_OK message. The sizes are not all the same, and none of them reached the MSS of 65,495. They are best understood as captured frames rather than wire-sized segments, because loopback segmentation offload produces large, uneven frames.

No segment carried the end of one ETP message and the start of the next: HELLO_OK and BYE_OK each travel alone, and frame 33 begins with the GET_OK header. This is because the protocol is strictly request/response. Here one ETP message spanned seven segments, but TCP gives no such guarantee in general. A message can be split at any byte, and two messages can arrive glued together.

So code that reads an ETP message must keep reading until it has exactly the number of bytes it needs, and never assume one recv equals one message. My code does this in recv_exact in transport_tcp.py, which loops on recv until n bytes have been collected. read_message in etp.py calls it twice: once for the 8-byte header, then once for LENGTH bytes of payload.

## Q4. Teardown and TIME-WAIT

_Which side sent the first FIN in the GET session, and why — cite the rule in the
specification. Paste the relevant line(s) from `evidence/time-wait.txt`. Which end
of the connection (address and port) is in TIME-WAIT, why that end, and roughly how
long Linux keeps it there? Connect this to the SO_REUSEADDR requirement.

The server sent the first FIN in the GET session (frame 46; the client's FIN follows in frame 47). The specification requires this in Section 5, item 4 ("Closing"): after sending BYE_OK the server closes the connection, and the client waits until it sees the server's close (a read returns end-of-stream) and only then closes its own side. Section 10 gives the matching client rule.

The relevant line from time-wait.txt:

`0      0          127.0.0.1:5000    127.0.0.1:48098`

The end in TIME-WAIT is the server's, 127.0.0.1:5000. TIME-WAIT belongs to the side that performs the active close (sends the first FIN). After acknowledging the peer's FIN it must keep the connection's address and port pair reserved, so that delayed segments from the old connection cannot be mistaken for a new one and so it can re-acknowledge if its final ACK is lost. Linux keeps the entry for about a minute (60 seconds).

This is why the specification requires SO_REUSEADDR. Because the server is the active closer, its listening port 5000 is held in TIME-WAIT after every BYE. A restarted server's bind() would fail with "address already in use" until the entries expire. Setting SO_REUSEADDR on the listening socket before bind() lets it restart immediately, which is what listen() in transport_tcp.py does and what test S18 checks.

## Q5. What an observer learns

_Suppose someone else obtained this capture file. List specifically what they
learn about your session (names, sizes, timing, anything else). You captured with a
200-byte snap length: what does that hide, and what does it not hide? Which kind of
protection would stop a real on-path observer from learning these things, and what
would it still not hide?

Everything is plaintext, so an observer learns:

Who: the client and server addresses and ports, and the client name Jackson (in HELLO and HELLO_OK).
What: the operations used (HELLO, GET, BYE), the filename photo.bin, and that the request succeeded (GET_OK, STATUS 0).
How big and when: the file size (262,144) from the LENGTH field and the sequence numbers, plus the timing, number of segments and duration of the session.

The 200-byte snap length keeps only the first 200 bytes of each frame, which is 134 payload bytes after the 66 bytes of Ethernet, IP and TCP headers. It hides most of the file content, because each 32–47 KB segment is cut down to 134 bytes. It does not hide the headers, so sizes, ports, and the sequence and ack numbers all survive. It also doesn’t hide the short messages, because HELLO, the GET request and BYE all fit within 134 bytes. The first 134 bytes of each data segment of the file are also still readable.

Encrypting the stream, for example TLS beneath the ETP layer at the transport seam, would hide the name, the filename, the status and the file contents. It would still not hide who is talking to whom (IP addresses and ports), when and for how long, or the approximate volume of data. Packet sizes and timing can also still reveal what was transferred.