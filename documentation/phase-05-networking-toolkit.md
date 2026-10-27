# Phase 5 — Linux networking toolkit

[Previous: DNS and DHCP](phase-04c-dns-dhcp.md) · [Next: remote SSH](phase-06-remote-ssh.md)

**Run on:** `dc-network-01` inside Ubuntu. **Start with:** working DNS and internet access.

## Step 1 — Install the tools

```bash
sudo apt update
sudo apt install -y tcpdump traceroute iperf3 curl htop dnsutils netcat-openbsd iproute2
```

If the package installer asks whether to start iperf3 as a daemon, choose **No**; the walkthrough starts a temporary foreground server when needed. If a daemon was enabled automatically, inspect `systemctl status iperf3 --no-pager` and stop it with `sudo systemctl disable --now iperf3` before the foreground exercises.

```bash
tcpdump --version
traceroute --version
iperf3 --version
curl --version
htop --version
```

Record package versions rather than assuming every Ubuntu installation has identical output.

## Step 2 — Trace a path

```bash
ip route get 1.1.1.1
traceroute -n -m 12 -w 1 1.1.1.1
```

`ip route get` shows the local route decision. `traceroute` probes successive hops. Asterisks mean a probe did not receive the expected reply; they do not, by themselves, prove forwarding failed. Compare the final destination's reachability with your Phase 4A results.

## Step 3 — Inspect an HTTP transaction

```bash
curl -I --connect-timeout 5 --max-time 15 https://ubuntu.com
curl -o /dev/null -sS --connect-timeout 5 --max-time 15 \
  -w 'HTTP %{http_code}; connect %{time_connect}s; total %{time_total}s\n' https://ubuntu.com
```

Explain the HTTP status, headers, and timings. A redirect is an application response; it is not a network failure. Do not use `curl -k` to hide certificate errors.

## Step 4 — Inspect listeners and resource use

```bash
sudo ss -lntup
free -h
uptime
htop
```

In `ss`, locate TCP port 22 and identify the listening process. In `htop`, find CPU and memory usage, observe a process, then press `q` to exit. Low free memory alone does not prove memory pressure; compare the available-memory figure.

## Step 5 — Take a short ICMP capture

Open two Ubuntu terminals. Set the correct management interface in terminal A:

```bash
LAB_MGMT_IF=enp0s1
sudo timeout 15 tcpdump -ni "$LAB_MGMT_IF" -c 8 icmp
```

In terminal B, ping your actual gateway:

```bash
ping -c 4 192.168.64.1
```

Look for echo requests and replies. The capture ends after eight matching packets or the time limit; a timeout exit code is expected if fewer packets match. Run captures only on your lab interfaces.

## Step 6 — Measure a local test connection

In terminal A:

```bash
iperf3 -s -1 -B 127.0.0.1
```

In terminal B:

```bash
iperf3 -c 127.0.0.1 -t 5
```

This checks that the tool works over loopback. It does **not** measure the virtual network or physical network. Phase 9 repeats the test between VMs.

**Checkpoint:** Explain each tool's purpose and limits: `tcpdump` captures packets; `traceroute` probes hops; `iperf3` measures test-stream throughput; `curl` tests application requests; `htop` shows process/resource activity. The temporary iperf3 server exits after one client.

References: [tcpdump manual](https://manpages.ubuntu.com/manpages/resolute/man8/tcpdump.8.html), [iperf3 manual](https://software.es.net/iperf/invoking.html).
