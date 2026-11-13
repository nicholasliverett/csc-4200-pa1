# PA1 Design — ETP/1 Client and Server

## 1. Architecture

client.py ──┐                      ┌── server.py
            ├──► etp.py            │
            └──► transport.py ◄────┘
                      │  (one import line)
                      ▼
                 transport_tcp.py ──► socket (the only file that uses it)

## 2. The transport seam

transport_tcp.py is the transport. Its six operations:

listen: SO_REUSEADDR, bind, listen.
accept: returns (conn, peer).
connect: socket.create_connection.
send_all: sendall.
recv_exact: loops on recv until n bytes arrive; raises ConnectionClosed(clean=...) if the stream ends.
close: shutdown(SHUT_WR), drain for about a second, then close.

No other file uses the socket API. To switch to the PA2 transport we wrote a second module with the same six functions and change the import in transport.py. The ETP, server and client code stay the same.

## 3. Framing

read_message calls recv_exact(conn, 8) for the header, then recv_exact(conn, LENGTH) for the payload. recv_exact keeps reading until it has exactly n bytes, so split headers are reassembled and extra bytes stay in the socket for the next call, which handles messages packed together. VERSION and LENGTH are checked before the payload is read: a bad version means the payload can't be trusted, and an oversized LENGTH must never cause a large read or allocation. The server replies BAD_VERSION or TOO_LARGE, both fatal, in the Section 8.1 order.

## 4. Concurrency

One thread per connection. A client stalled mid-header blocks only its own thread; the accept loop and other sessions keep running. The only shared state is the root directory and a lock around log output. PUT writes to a temporary file whose name starts with a period (so it is never listed or fetched) and renames it into place with os.replace. A concurrent GET sees either the old file or the whole new one, and the last of two PUTs wins.

## 5. Errors and disconnects

Handlers raise ProtocolError(code); handle_session sends an ERROR with that code. If the code is in FATAL_ERRORS (BAD_VERSION, TOO_LARGE, NOT_READY) the session ends and transport.close closes gracefully. Otherwise the loop continues, since the whole payload has already been read. An unexpected handler exception becomes a non-fatal SERVER_ERROR.

If a client disconnects during a large GET, sendall raises an OSError. It is caught at session level, logged to stderr, and the connection is closed; other sessions are unaffected. Python raises an exception instead of killing the process with SIGPIPE, so no signal handling was needed.

Client exit codes: 0 success; 1 server ERROR; 2 any other failure (connect error, unreadable file, early close, bad response); 3 PUT digest mismatch.

## 6. Threat model

Anyone who can reach the port can attack the server, and anyone on the path can read or alter the traffic, because ETP/1 has no authentication or encryption. The server resists path traversal: every name is checked against the strict Section 7 rule before it is joined to the root, so ../secret gets BAD_NAME. It does not resist resource exhaustion: one PUT can hold 16 MiB in memory, connections and threads are unlimited, and there is no idle timeout, so many stalled clients can tie up resources.

## 7. Testing

Beyond etp_selfcheck.py, we captured a GET session in Wireshark and checked the bytes, segmentation and teardown against the specification, and checked TIME-WAIT with ss. We also compared transferred files with the originals.

The capture found a bug the self-check missed. It showed the client's FIN before the server's, with the client in TIME-WAIT, which contradicts Section 5 item 4 (the server closes after BYE_OK; the client waits for that close). The client closed immediately after BYE_OK, so whichever side closed first won a race. The fix: the client now reads until end-of-stream after BYE_OK before closing. The recapture shows the server's FIN first and 127.0.0.1:5000 in TIME-WAIT.
