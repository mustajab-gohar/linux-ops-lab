# Week 3: Network Diagnostics & Packet Analysis

## Scenario
When microservices fail to communicate, logs often only report "Connection Timeout." This lab focuses on dropping down to Layer 3/Layer 4 to prove exactly where network traffic is failing using raw packet capture and structured OSI-model troubleshooting.

## Deliverable 1: `tcpdump` Lifecycle Capture
Captured the complete lifecycle of a web request to an unencrypted endpoint (`http://neverssl.com`) to analyze the TCP handshake and HTTP payload.

**Command Executed:**
```bash
sudo tcpdump -i any -n -A 'port 53 or (host neverssl.com and port 80)' -w output.pcap
```
## Packet Analysis & Observations:

- **DNS Resolution (Happy Eyeballs):** Observed the OS simultaneously requesting `A` (IPv4) and `AAAA` (IPv6) records from the recursive resolver.
```bash
20:14:58.406917 IP 203.0.113.5.44274 > 10.1.144.1.53: 58269+ A? neverssl.com. (30)
E..:..@.@...
...
......5.&.H.............neverssl.com.....
20:14:58.407474 IP 203.0.113.5.44274 > 10.1.144.1.53: 60318+ AAAA? neverssl.com. (30)
E..:..@.@...
...
......5.&.H.............neverssl.com.....
20:14:58.440288 IP 10.1.144.1.53 > 203.0.113.5.44274: 58269 1/0/0 A 34.223.124.45 (46)
E..J.?..@..S
...
....5...6c..............neverssl.com..............<..".|-
20:14:58.455094 IP 10.1.144.1.53 > 203.0.113.5.44274: 60318 1/0/0 AAAA 2600:1f13:37c:1400:ba21:7165:5fc7:736e (58)
E..V.@..@..F
...
```

- **IPv6 Fallback:** The initial `SYN` packet was sent via IPv6. The network returned an `RST, ACK` (Reset), instantly killing the attempt due to lack of local IPv6 routing.
```bash
20:15:05.352471 IP6 2001:db8::1.43128 > 2600:1f13:37c:1400:ba21:7165:5fc7:736e.80: Flags [S], seq 461478720, win 64800, options [mss 1440,sackOK,TS val 1993054845 ecr 0,nop,wscale 8], length 0
`       *..(.@..b\.7..,gA.T..&&....|...!qe_.sn.x.P...@....... O..........
v..}........
20:15:05.354851 IP6 2600:1f13:37c:1400:ba21:7165:5fc7:736e.80 > 2001:db8::1.43128: Flags [R.], seq 0, ack 461478721, win 65535, length 0
`.......&....|...!qe_.sn..b\.7..,gA.T..&.P.x.......AP.......
```

- **TCP Retransmission:** The OS gracefully fell back to IPv4 and sent a new `SYN`. After a slight delay, a TCP Retransmission occurred, guaranteeing delivery before successfully receiving the `SYN, ACK` from the destination.
```bash
20:15:05.356292 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [S], seq 76357929, win 64240, options [mss 1460,sackOK,TS val 1695758766 ecr 0,nop,wscale 8], length 0
E..<.K@.@..U
...".|-.<.P..!).........I.........
e.9.........
20:15:06.385373 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [S], seq 76357929, win 64240, options [mss 1460,sackOK,TS val 1695759795 ecr 0,nop,wscale 8], length 0
E..<.L@.@..T
...".|-.<.P..!).........I.........
e.=.........
20:15:06.569312 IP 34.223.124.45.80 > 203.0.113.5.34364: Flags [S.], seq 74560001, ack 76357930, win 65535, options [mss 1460], length 0
E..,.C..@..f".|-
....P.<.q....!*`....E........
20:15:06.569498 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [.], ack 1, win 64240, length 0
E..(.M@.@..g
...".|-.<.P..!*.q..P....5..
```

- **Application Payload:** Captured the raw `GET / HTTP/1.1` request and the subsequent `200 OK` response payload.
```bash
20:15:06.571649 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [P.], seq 1:77, ack 1, win 64240, length 76: HTTP: GET / HTTP/1.1
E..t.N@.@...
...".|-.<.P..!*.q..P.......GET / HTTP/1.1
Host: neverssl.com
```
```bash
20:15:06.573161 IP 34.223.124.45.80 > 203.0.113.5.34364: Flags [.], ack 77, win 65535, length 0
E..(.D..@..i".|-
....P.<.q....!vP.............
20:15:06.877231 IP 34.223.124.45.80 > 203.0.113.5.34364: Flags [.], seq 1:2881, ack 77, win 65535, length 2880: HTTP: HTTP/1.1 200 OK
E..h.E..@..(".|-
....P.<.q....!vP....u..HTTP/1.1 200 OK
```

- **Graceful Teardown:** Observed the 4-way disconnect (`FIN, ACK` -> `ACK` -> `FIN, ACK` -> `ACK`) successfully freeing the server socket.
```bash
20:15:06.887098 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [F.], seq 77, ack 4262, win 65535, length 0
E..(.Q@.@..c
...".|-.<.P..!v.q..P....5..
20:15:06.888188 IP 34.223.124.45.80 > 203.0.113.5.34364: Flags [.], ack 78, win 65535, length 0
E..(.H..@..e".|-
....P.<.q....!wP.............
20:15:07.183902 IP 34.223.124.45.80 > 203.0.113.5.34364: Flags [F.], seq 4262, ack 78, win 65535, length 0
E..(.I..@..d".|-
....P.<.q....!wP.............
20:15:07.183958 IP 203.0.113.5.34364 > 34.223.124.45.80: Flags [.], ack 4263, win 6464, length 0
E..(..@.@...
...".|-.<.P..!w.q..P..@v...
```
*(Note: The full redacted packet capture is available in the `capture-log.txt` in this directory)* 


## Deliverable 2: "App is Down" Troubleshooting Runbook
A structured, bottom-up approach to diagnosing unreachable servers, bypassing the instinct to "just restart services."

- **Layer 1/2 (Physical/Data Link)**: Verify interface state (`ip link`) and check ARP cache (`arp -a`) to ensure gateway visibility.

- **Layer 3 (Network):** Verify IP assignment (`ip a`) and routing tables (`ip route`). Use `traceroute` to identify exact packet-drop locations across external hops.

- **Layer 4 (Transport):** Verify local port binding (`ss -tulnp | grep 80`). Check host firewall policies (`iptables -L`). Use `nc -vz <ip> 80` from an external client to test the TCP 3-way handshake independently of the application.

- **Layer 7 (Application):** Fetch raw HTTP headers (`curl -Iv <url>`). Check DNS resolution (`dig`). Tail application error logs (`journalctl -xe`).
