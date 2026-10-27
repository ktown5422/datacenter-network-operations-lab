# Phase 9 — Packet analysis and performance testing

[Previous: firewall controls](phase-08-firewall-controls.md) · [Next: monitoring](phase-10-monitoring.md)

**Run on:** the server, monitor, and Mac as labeled. **Start with:** working routed HTTP after Phase 8. **Goal:** Capture four protocols and measure throughput over the private VM-to-VM route.

## Step 1 — Prepare two server terminals

On **`dc-server-01`**, open two console/SSH terminals. Set the correct interface variables in both:

```bash
LAB_APP_IF=enp0s2
LAB_MGMT_IF=enp0s1
mkdir -p ~/dc-lab-notes/captures
ip -br -4 addr
```

Terminal A runs captures; terminal B generates traffic. Each `timeout` bounds a capture to 20 seconds; you can press **Ctrl+C** earlier after the test. Capture only the named lab interface and protocol. `-Z "$(id -un)"` drops capture privileges to your Ubuntu account before writing the file, so the output can be saved in your notes directory.

## Step 2 — Capture ARP on the application segment

On **server terminal A**:

```bash
sudo timeout 20 tcpdump -ni "$LAB_APP_IF" -s 0 -U -Z "$(id -un)" \
  -w "$HOME/dc-lab-notes/captures/arp.pcap" arp
```

On **server terminal B**, clear only the private gateway's cached neighbor entry, then generate traffic:

```bash
sudo ip neigh del 10.10.20.1 dev "$LAB_APP_IF"
ping -c 4 10.10.20.1
ip neigh show dev "$LAB_APP_IF"
```

If the neighbor entry was already absent, the deletion can report “No such file”; continue with the ping. Look for the “who has” request and the reply identifying the gateway MAC. No management neighbor entry is removed.

## Step 3 — Capture routed ICMP

On **server terminal A**:

```bash
sudo timeout 20 tcpdump -ni "$LAB_APP_IF" -s 0 -U -Z "$(id -un)" \
  -w "$HOME/dc-lab-notes/captures/icmp.pcap" icmp
```

On **`dc-monitor-01`**:

```bash
ping -c 4 10.10.20.10
```

Identify echo requests/replies and their IP addresses. On the server's Ethernet segment, the frame's next-hop MAC belongs to the router, while the remote packet's source IP remains `10.10.30.10`. Routing does not replace that source IP with the router's address in this no-NAT lab.

## Step 4 — Capture DNS on the management segment

On **server terminal A**:

```bash
sudo timeout 20 tcpdump -ni "$LAB_MGMT_IF" -s 0 -U -Z "$(id -un)" \
  -w "$HOME/dc-lab-notes/captures/dns.pcap" 'udp port 53 or tcp port 53'
```

On **server terminal B**:

```bash
sudo resolvectl flush-caches
dig ubuntu.com A +time=2 +tries=1
```

DNS leaves via management, where the upstream resolver is reachable. Identify the query name, record type, response code, answer, and query/response pairing. The loopback stub transaction is not visible on this Ethernet capture. If your upstream uses encrypted DNS, a port 53 capture will not show its encrypted transaction; note the actual resolver configuration.

## Step 5 — Capture the HTTP TCP exchange

On **server terminal A**:

```bash
sudo timeout 20 tcpdump -ni "$LAB_APP_IF" -s 0 -U -Z "$(id -un)" \
  -w "$HOME/dc-lab-notes/captures/http.pcap" 'tcp port 80'
```

From **`dc-monitor-01`**:

```bash
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

Locate SYN → SYN/ACK → ACK, the HTTP request/response, and connection close. An unanswered SYN differs from a TCP reset and from an HTTP error response.

## Step 6 — Read and transfer the captures

After each capture has ended, on **`dc-server-01`**:

```bash
ls -lh ~/dc-lab-notes/captures/
tcpdump -nn -r ~/dc-lab-notes/captures/arp.pcap
tcpdump -nn -r ~/dc-lab-notes/captures/http.pcap
```

In **macOS Terminal**, using the server's **management** address:

```bash
LAB_USER=your_ubuntu_username
LAB_SERVER_MGMT_IP=192.168.64.4
mkdir -p ~/dc-lab-captures
scp -i ~/.ssh/id_ed25519_dc_lab \
  "$LAB_USER@$LAB_SERVER_MGMT_IP:dc-lab-notes/captures/*.pcap" ~/dc-lab-captures/
```

Install/open Wireshark from [its official download page](https://www.wireshark.org/download.html) on the Mac. Use **File → Open** for each capture. Apply display filters `arp`, `icmp`, `dns`, and `tcp.port == 80`. For HTTP, select a packet and use **Follow → TCP Stream**. Save short notes describing what each capture proves. Review packet contents before sharing files.

## Step 7 — Permit the throughput test

On **`dc-server-01`**:

```bash
sudo apt install -y iperf3
sudo ufw allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 5201 proto tcp
iperf3 -s -1 -B 10.10.20.10
```

If port 5201 is already in use, inspect `ss -lntp` and stop an automatically enabled iperf3 service before starting this temporary server. Phase 8's router rules already permit this lab path. On **`dc-monitor-01`**, install iperf3 if needed and run:

```bash
ip route get 10.10.20.10
iperf3 -c 10.10.20.10 -B 10.10.30.10 -t 10 --get-server-output
```

## Step 8 — Compare directions and save results

Restart `iperf3 -s -1 -B 10.10.20.10` on the **server** before **each** test. On the **monitor**:

```bash
mkdir -p ~/dc-lab-notes/performance
iperf3 -c 10.10.20.10 -B 10.10.30.10 -t 10 -J \
  > ~/dc-lab-notes/performance/forward.json
```

Restart the one-off server again, then on the **monitor**:

```bash
iperf3 -c 10.10.20.10 -B 10.10.30.10 -t 10 -R -J \
  > ~/dc-lab-notes/performance/reverse.json
```

Record endpoints, direction, duration, sender/receiver throughput, and retransmissions where reported. `-R` makes the server send the test stream. Repeat under similar host load if comparing runs; all VMs share the Mac's CPU and memory, so this is not a physical data center link benchmark.

## Step 9 — Clean up

The one-off server exits when its client finishes; press Ctrl+C if it is still waiting. On **`dc-server-01`**:

```bash
sudo ufw delete allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 5201 proto tcp
sudo ss -lntp '( sport = :5201 )'
```

**Checkpoint:** Four captures have interpretable traffic, both throughput directions have results, and no temporary iperf3 listener or allow rule remains. Phase 11 temporarily reopens that rule for its performance drill.

References: [tcpdump options](https://manpages.ubuntu.com/manpages/resolute/man8/tcpdump.8.html), [iperf3 client/server flags](https://software.es.net/iperf/invoking.html), [Wireshark display filters](https://www.wireshark.org/docs/wsug_html_chunked/ChWorkDisplayFilterSection.html).
