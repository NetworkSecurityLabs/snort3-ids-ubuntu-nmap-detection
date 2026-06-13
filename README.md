# 🛡️ Snort 3 IDS on Ubuntu — Detect nmap Attacks in Real Time

A complete, hands-on setup of **Snort 3 (IDS mode)** on Ubuntu, with custom detection rules that catch seven distinct nmap scan types launched from a Kali Linux attacker VM. Every alert shown here is real — captured live during testing.

📖 **Full article on Dev.to:** [How I Built a Network Intrusion Detection System with Snort 3 on Ubuntu — and Caught Every Scan](https://dev.to/almahmudkhalif/lab-task-13-how-i-built-a-network-intrusion-detection-system-with-snort-3-on-ubuntu-and-caught-3mep)

---

## 📋 Table of Contents

- [Environment](#environment)
- [Attack Types Detected](#attack-types-detected)
- [Step 1 — Install Dependencies](#step-1--update-ubuntu-and-install-dependencies)
- [Step 2 — Install LibDAQ](#step-3--install-libdaq-snorts-data-acquisition-library)
- [Step 3 — Install Tcmalloc](#step-4--install-tcmalloc-memory-optimization)
- [Step 4 — Install Snort 3](#step-5--install-snort-3)
- [Step 5 — Configure Network Interface](#step-6--configure-the-network-interface)
- [Step 6 — Configure snort.lua](#step-7--create-the-rules-directory-and-configure-snortlua)
- [Step 7 — Write Detection Rules](#step-8--write-the-detection-rules)
- [Step 8 — Validate and Run](#step-9--validate-the-configuration)
- [Step 9 — Simulate Attacks from Kali](#step-11--simulate-attacks-from-kali-linux)
- [Live Alert Output](#live-alert-output)
- [Common Mistakes](#common-mistakes)
- [Connect With Me](#-connect-with-me)

---

## Environment

| Machine | OS | Role | IP |
|---|---|---|---|
| Defender | Ubuntu 22.04 | Snort 3 IDS | 192.168.1.104 |
| Attacker | Kali Linux 2026.1 | nmap scan source | 192.168.1.106 |

Both VMs use **Bridged Adapter** in VirtualBox so they share the same subnet.

---

## Attack Types Detected

| # | Scan Type | nmap Flag | Snort Rule Flag |
|---|---|---|---|
| 1 | Ping Sweep | `-sn` | ICMP + dsize:0 |
| 2 | XMAS Scan | `-sX` | flags:FPU |
| 3 | FIN Scan | `-sF` | flags:F |
| 4 | NULL Scan | `-sN` | flags:0 |
| 5 | SYN Scan | `-sS` | flags:S |
| 6 | TCP Connect Scan | `-sT` | tcp (no flag filter) |
| 7 | UDP Scan | `-sU` | udp protocol |

---

## Step 1 — Update Ubuntu and Install Dependencies

```bash
sudo apt update

sudo apt install -y build-essential \
  libpcap-dev libpcre2-dev libnet1-dev zlib1g-dev luajit hwloc \
  libdumbnet-dev bison flex liblzma-dev openssl libssl-dev \
  pkg-config libhwloc-dev cmake cpputest libsqlite3-dev uuid-dev \
  libcmocka-dev libnetfilter-queue-dev libmnl-dev autotools-dev \
  libluajit-5.1-dev libunwind-dev git wget ethtool
```

![Dependencies install](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p84es0aeaefdsbpre23y.png)

---

## Step 2 — Create a Working Directory

```bash
mkdir snort-source-files
cd snort-source-files
```

---

## Step 3 — Install LibDAQ (Snort's Data Acquisition Library)

LibDAQ is Snort's packet capture abstraction layer.

```bash
git clone https://github.com/snort3/libdaq.git
cd libdaq
sudo ./bootstrap
sudo ./configure
sudo make
sudo make install
cd ..
```

![git clone libdaq](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h69abnxvvus4ykr64ryl.png)

![bootstrap libdaq](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/52i39vd1dy58hyvrehv1.png)

![configure libdaq](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fpjkug8y9s095kbvjktt.png)

![make libdaq](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4ipqrxvw78giudbuinty.png)

![make install libdaq](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5u9af4hfcv0zfmnxt80s.png)

---

## Step 4 — Install Tcmalloc (Memory Optimization)

Google's memory allocator — reduces fragmentation and speeds up Snort under load.

```bash
wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.10/gperftools-2.10.tar.gz
tar xzf gperftools-2.10.tar.gz
cd gperftools-2.10
sudo ./configure
sudo make
sudo make install
cd ..
```

![wget gperftools](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mjq2cjmp3w788jkpgix4.png)

![configure gperftools](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8tf0dtvak7e452tjn5p2.png)

![make gperftools](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/orfjqqx2icbhlz6go11f.png)

![make install gperftools](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/15qw2c9kuj3xgzerai8e.png)

---

## Step 5 — Install Snort 3

```bash
git clone https://github.com/snort3/snort3.git
cd snort3
sudo ./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc
cd build
sudo make
sudo make install
sudo ldconfig
sudo ln -s /usr/local/bin/snort /usr/sbin/snort
snort -V
```

Expected version output:

```
-*> Snort++ <*-
   Version 3.12.2.0
   By Martin Roesch & The Snort Team
```

![git clone snort3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tvq16ohu2phegjg7wap8.png)

![configure_cmake snort3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5ztv45084rpwcu99w7j4.png)

![make snort3 progress](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ys2pbv770waseozvu62g.png)

![make install snort3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pgl8qi7zx0i8uun0ctu3.png)

![snort -V version confirmed](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/59sryuyxngqfvwa55nfq.png)

---

## Step 6 — Configure the Network Interface

```bash
# Find your interface name
ip a

# Set promiscuous mode
sudo ip link set dev enp0s3 promisc on
sudo ethtool -K enp0s3 gro off lro off
```

Create a systemd service to persist across reboots:

```bash
sudo nano /etc/systemd/system/snort3-nic.service
```

```ini
[Unit]
Description=Set Snort3 NIC in promiscuous mode

[Service]
Type=oneshot
ExecStart=/sbin/ip link set dev enp0s3 promisc on
ExecStart=/sbin/ethtool -K enp0s3 gro off lro off

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now snort3-nic.service
sudo systemctl status snort3-nic.service
```

![ip a showing enp0s3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jyu3m5t9wpyympvgaogi.png)

![install ethtool](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t9jv1c0qbwof5exgnzfh.png)

![snort3-nic.service file content](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ovvnp1xld352nulm8paa.png)

![systemctl status snort3-nic.service active](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kd76deyrfdp2wgnmw1jm.png)

---

## Step 7 — Create the Rules Directory and Configure snort.lua

```bash
sudo mkdir -p /usr/local/etc/rules/local-rules
cd /usr/local/etc/snort
sudo nano snort.lua
```

Set your network in snort.lua:

```lua
HOME_NET = '192.168.0.0/24'
EXTERNAL_NET = '!$HOME_NET'

include 'snort_defaults.lua'
```

Add the IPS block (scroll to detection section):

```lua
ips =
{
    include = '/usr/local/etc/rules/local-rules/local.rules',
    variables =
    {
        nets =
        {
            HOME_NET = HOME_NET,
            EXTERNAL_NET = EXTERNAL_NET
        }
    }
}
```

![mkdir rules directory](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iwgw5xt8pyqoipzj2svh.png)

![snort.lua HOME_NET config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8e8zpb72348kusnnrkw5.png)

![snort.lua ips block](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hhciank53801dg1wj7j6.png)

![Kali and Ubuntu network settings bridged](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4e8qxeazvfoui2xsz41z.png)

![ip a on Ubuntu showing 192.168.1.104](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7txj1wilob0op373p2hu.png)

![ip a on Kali showing 192.168.1.106](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vnsncj6lmeg3ufn5sece.png)

![snort.lua updated HOME_NET 192.168.0.0/24](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t8lxbs2wmz9nrceuney7.png)

---

## Step 8 — Write the Detection Rules

```bash
sudo nano /usr/local/etc/rules/local-rules/local.rules
```

```text
alert icmp any any -> $HOME_NET any (msg:"NMAP Ping Sweep Scan"; dsize:0; sid:1000001; rev:1;)

alert tcp any any -> $HOME_NET 22 (msg:"NMAP XMAS Scan"; flags:FPU; sid:1000002; rev:1;)

alert tcp any any -> $HOME_NET 22 (msg:"NMAP FIN Scan"; flags:F; sid:1000003; rev:1;)

alert tcp any any -> $HOME_NET 22 (msg:"NMAP NULL Scan"; flags:0; sid:1000004; rev:1;)

alert tcp any any -> $HOME_NET 22 (msg:"NMAP SYN Scan"; flags:S; sid:1000005; rev:1;)

alert tcp any any -> $HOME_NET 22 (msg:"NMAP TCP Connect Scan"; sid:1000006; rev:1;)

alert udp any any -> $HOME_NET any (msg:"NMAP UDP Scan"; sid:1000007; rev:1;)
```

![local.rules in nano editor](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aggso5pkoi4wroefoifd.png)

![snort config test validation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1gcwxjocbzc1c948mhpb.png)

---

## Step 9 — Validate the Configuration

```bash
snort -c /usr/local/etc/snort/snort.lua -T
```

Expected:

```
Snort successfully validated the configuration (with 0 warnings).
o")~   Snort exiting
```

![validation output part 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yfegvrjocuaza8j7b7q8.png)

![validation output part 2 - 0 warnings](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9os2ytaixxx0grhynox2.png)

---

## Step 10 — Start Snort in Alert Mode

```bash
sudo snort -c /usr/local/etc/snort/snort.lua -i enp0s3 -A alert_fast
```

![snort starting - commencing packet processing](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6jrb91gqgo6fo2mp07vc.png)

![snort running on enp0s3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u07ik8str9kkynvxp9m0.png)

---

## Step 11 — Simulate Attacks from Kali Linux

```bash
# 1. Ping Sweep
nmap -sn 192.168.1.104

# 2. XMAS Scan
sudo nmap -sX 192.168.1.104

# 3. FIN Scan
sudo nmap -sF 192.168.1.104

# 4. NULL Scan
sudo nmap -sN 192.168.1.104

# 5. SYN Scan
sudo nmap -sS 192.168.1.104

# 6. TCP Connect Scan
nmap -sT 192.168.1.104

# 7. UDP Scan
sudo nmap -sU 192.168.1.104
```

![Ping sweep + Snort alerting](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/prrov7gnk9blvkty9l2z.png)

![XMAS scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gxogrposmxqg7v2xee21.png)

![FIN scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h4jizyjp0oq2qubo6izi.png)

![NULL scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p0xmrazt8erp55qxtcml.png)

![SYN scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w9zxerwj0o8j4ydg9ewh.png)

![TCP Connect scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hk5lk6ne3p2bzmibwito.png)

![UDP scan triggered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m0vsz21hhlihsk7efp18.png)

---

## Live Alert Output

When all scans run, Snort produces alert lines like this in real time:

```
05/14-18:13:07.004392 [**] [1:1000001:1] "NMAP Ping Sweep" [**] [Priority: 0] {ICMP} 192.168.1.106 -> 192.168.1.104
05/14-18:15:21.301976 [**] [1:1000002:1] "NMAP XMAS Scan" [**] [Priority: 0] {TCP} 192.168.1.106:34107 -> 192.168.1.104:22
05/14-18:16:03.900911 [**] [1:1000003:1] "NMAP FIN Scan" [**] [Priority: 0] {TCP} 192.168.1.106:41928 -> 192.168.1.104:22
05/14-18:16:53.770290 [**] [1:1000004:1] "NMAP NULL Scan" [**] [Priority: 0] {TCP} 192.168.1.106:52411 -> 192.168.1.104:22
05/14-18:17:26.976133 [**] [1:1000005:1] "NMAP SYN Scan" [**] [Priority: 0] {TCP} 192.168.1.106:40209 -> 192.168.1.104:22
05/14-18:15:21.301976 [**] [1:1000006:1] "NMAP TCP Connect Scan" [**] [Priority: 0] {TCP} 192.168.1.106:34107 -> 192.168.1.104:22
05/14-18:12:48.402112 [**] [1:1000007:1] "NMAP UDP Scan" [**] [Priority: 0] {UDP} 192.168.1.1:1900 -> 239.255.255.250:1900
```

All 7 rules firing — IDS is working.

---

## Common Mistakes

| Mistake | What Goes Wrong | Fix |
|---|---|---|
| Using Snort 2 instead of Snort 3 | Config file format is completely different | Clone from `github.com/snort3/snort3` |
| Wrong `HOME_NET` subnet | Rules never fire | Check with `ip a` and match exactly |
| Not setting promiscuous mode | Snort misses most traffic | `sudo ip link set dev enp0s3 promisc on` |
| Interface name mismatch | Snort starts but captures nothing | Confirm name with `ip a` |
| Rules path typo in snort.lua | Snort validates but no alerts fire | Double-check the full path |
| Skipping `sudo ldconfig` | Binary cannot find shared libraries | Always run after `make install` |
| `flags:0` for NULL scan version mismatch | Rule loads but never matches | Test with `snort -c snort.lua -T` |

---

## 🌐 Connect With Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)
[![Dev.to](https://img.shields.io/badge/Dev.to-Articles-black?logo=devdotto)](https://dev.to/almahmudkhalif/)
