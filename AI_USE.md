# AI Use Disclosure — PA1

## 1. Did you use generative AI on this assignment?

- [ ] No. I did not use any generative AI tool for any part of PA1.
- [X] Yes, as described below.

## 2. Statement (the syllabus form)

The author acknowledges the utilization of Claude Sonnet, a language model developed by Anthropic, in the preparation of this assignment. Sonnet was employed in the following manner(s): diagnosing self-check failures and
explaining the causes; reading the packet capture to extract facts; reviewing
and drafting edits to DESIGN.md.

## 3. Log

| # | What I asked (summary) | What I used from the answer | What I changed or rejected | How I checked it was right |
|---|---|---|---|---|
| 1 | Why the self-check failed (S01, S02, S05, S06, C03, seam) and how to fix it | The fixes for S01 (echo the client name in HELLO_OK), S02 (LIST as `name\tsize\n`, sorted by byte value) and the seam (rename `send` to `reply`) | Rejected the earlier AI feedback on PUT (`name \0 digest data`, wrong client slice), because it did not match my code or the spec | Reran `etp_selfcheck.py`: S01, S02 and the seam passed |
| 2 | Fix PUT after S05, S06 and C03 still failed | The 2-byte `NAMELEN` framing (`NAMELEN \| NAME \| DATA`) in both `server.py` and `client.py` | Replaced my NUL-separated parsing; the first diagnosis was dropped as wrong | Checked Section 6.4 and Figure 4 of the spec; the self-check reached 30/30 |
| 3 | Fill in the capture facts and Q1–Q5 from my pcap | The values from the capture (ISNs, MSS, byte and segment counts) and the answer text | Caught that my first capture had the client closing first, against Section 5 item 4; fixed the client to wait for the server's close, then recaptured and updated the facts and Q4 | New pcap shows the server's FIN first and `127.0.0.1:5000` in `time-wait.txt` |
| 4 | Complete `DESIGN.md` and make it concise | The draft sections; the AI also found that my PUT temp file name was a valid ETP name | Fixed the temp name to start with a period (Section 6.4); left a placeholder for my own extra tests in section 7 | Compared against Sections 6.4 and 9 of the spec; a PUT then GET round trip returned identical bytes |

## 4. Where AI-produced material remains

src/client.py
src/etp.py
src/server.py
src/transport_tcp.py
DESIGN.md
README.md
ANALYSIS.md

## 5. One thing the AI got wrong — or how you checked it

The AI feedback I was given on my failing PUT tests was wrong. It said the PUT payload was name \0 digest data and that my client was slicing the payload at a wrong offset. Neither was true: Section 6.4 of the specification defines the payload as NAMELEN (2 bytes) | NAME | DATA, and my code was splitting on a NUL byte. I caught it because the feedback did not match my code (my client never slices the PUT_OK payload) and the self-check output showed the checker reading two extra bytes before the name. I then confirmed the layout in the spec (Figure 4) and changed both the server and the client to use the 2-byte name length.
