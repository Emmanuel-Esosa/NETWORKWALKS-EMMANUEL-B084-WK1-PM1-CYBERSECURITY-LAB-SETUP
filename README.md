# Cybersecurity Lab Environment Setup on macOS

## Project Overview

For this task I set up a virtual cybersecurity lab on my Mac using VirtualBox, with Kali Linux as the attacker machine. Kali sits on its own private NAT Network (10.0.0.0/24) with a fixed IP address and full internet access, so other target machines can be added to the same network later for authorised testing.

I did this on macOS rather than Windows, so some steps differ from the course guide. I've noted those differences below.

> ⚠️ **Ethical use:** This lab is for learning only. I will only use it to test machines I own or have explicit permission to test.

## Project Objectives ## 
The main objectives of this lab are to:
- Build an isolated lab for practising cybersecurity tools safely.
- Create a private NAT Network for future target machines to connect to.
- Assign Kali a fixed IP address with full internet access.
- Enable file sharing and clipboard between the Mac and Kali.
- Save a clean snapshot as a recovery point.
- Document the setup and the problems solved along the way.
## Lab Requirements and Results

| Requirement | Result |
|:--|:--|
| VirtualBox as the base | ✅ VirtualBox on macOS |
| Kali Linux as the attacker machine | ✅ Kali Linux VM |
| NAT Network in 10.0.0.0/24 | ✅ `NatNetwork`, 10.0.0.0/24, DHCP enabled |
| Clipboard and drag & drop enabled | ✅ Both set to Bidirectional |
| Host Downloads folder shared | ✅ Mounted in Kali at `/media/sf_Downloads` |
| Kali IP 10.0.0.2/24 | ✅ Static IP configured |
| Full internet access | ✅ Gateway, external IP and DNS tests passed |
| Snapshot of the VM | ✅ `Week 1 Lab Setup` |

## Host System

| Item | Detail |
|:--|:--|
| Host OS | macOS (2 CPU cores) |
| Hypervisor | Oracle VirtualBox |
| Guest OS | Kali Linux (64-bit) |
| VM resources | 4 GB RAM, 2 CPUs |

> **Note on 7-Zip (Step 1):** 7-Zip is a Windows tool. macOS can extract archives natively, so this step wasn't needed. VirtualBox and Kali Linux (Steps 2 and 4) were already installed from earlier lab work.

## Step 0: Protecting My Existing Lab

My Kali VM was already part of a Wazuh SIEM lab I built earlier. Moving Kali to a new network would break its link to the Wazuh server, so before changing anything I shut down both VMs and took a snapshot of each (`wazuh lab done V`). This lets me return to the working Wazuh lab at any time.

## Step 3: Create the NAT Network

In VirtualBox: **File > Tools > Network Manager > NAT Networks > Create**

| Setting | Value |
|:--|:--|
| Name | NatNetwork |
| IPv4 Prefix | 10.0.0.0/24 |
| DHCP | Enabled |

VirtualBox gave the new network a default prefix of 10.0.2.0/24, so I changed it to 10.0.0.0/24 and clicked **Apply**.

<img width="1437" height="827" alt="1-nat-network" src="https://github.com/user-attachments/assets/59254789-56ed-43fb-b1b4-135f2ea23a59" />


## VM Settings for Kali

With Kali powered off, I opened **Settings** and configured the following.

### Network (Adapter 1)

| Setting | Value |
|:--|:--|
| Attached to | NAT Network |
| Name | NatNetwork |
| Adapter Type | Intel PRO/1000 MT Desktop |
| Promiscuous Mode | Allow All |

Promiscuous Mode is set to **Allow All** so Kali can see all traffic on the lab network for future sniffing exercises.

<img width="1439" height="829" alt="2-kali-network" src="https://github.com/user-attachments/assets/10e1997f-b5e6-4591-8564-9e3decff5af5" />


### Clipboard and Drag & Drop

**General > Features:** Shared Clipboard and Drag-and-Drop both set to **Bidirectional**.

<img width="1440" height="832" alt="3-clipboard" src="https://github.com/user-attachments/assets/1b4e97d4-4792-4320-a72c-2bfa67555143" />


### Shared Folder

**Shared Folders > Add:**

| Setting | Value |
|:--|:--|
| Folder Path | /Users/user/Downloads (macOS Downloads) |
| Folder Name | Downloads |
| Auto-mount | Yes |
| Make Global | Yes |
| Access | Full |

<img width="1440" height="829" alt="4-shared-folder" src="https://github.com/user-attachments/assets/4e7996e0-11c1-4c82-a63a-4dc3aad76d54" />


## Step 5: Configure Kali's IP Address

After booting Kali, I checked the network state:

```bash
ip a
nmcli device status
```

eth0 was stuck on *"connecting (getting IP configuration)"* with no IPv4 address, a known issue with VirtualBox 7 and newer Kali releases. I set the static IP with `nmcli`, including the `dad-timeout` fix from the course guide:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 10.0.0.2/24 ipv4.gateway 10.0.0.1 ipv4.dns 8.8.8.8 ipv4.dad-timeout 0
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

| Setting | Value |
|:--|:--|
| IP Address | 10.0.0.2/24 |
| Gateway | 10.0.0.1 |
| DNS | 8.8.8.8 |

`ip a` then showed `inet 10.0.0.2/24` on eth0.

<img width="1435" height="829" alt="5-ip-address" src="https://github.com/user-attachments/assets/0120d8d3-6114-4ff0-aaba-08663396d660" />


## Connectivity Tests

```bash
ping -c 3 10.0.0.1      # NAT Network gateway
ping -c 3 8.8.8.8       # external IP (internet)
ping -c 3 google.com    # DNS resolution
```

All three returned **3 packets received, 0% packet loss**.

| Test | Command | Result |
|:--|:--|:--|
| IP address | `ip a` | `10.0.0.2/24` on eth0 ✅ |
| Gateway | `ping -c 3 10.0.0.1` | 3/3 replies ✅ |
| Internet | `ping -c 3 8.8.8.8` | 3/3 replies ✅ |
| DNS | `ping -c 3 google.com` | Resolved and 3/3 replies ✅ |
| Shared folder | `mount \| grep vboxsf` | Downloads mounted at `/media/sf_Downloads` ✅ |


The gateway ping (10.0.0.1) worked first time, as shown in the Step 5 screenshot above. My first pings to 8.8.8.8 and google.com failed because my Mac had lost internet (see Problem 3). After connecting the Mac to a hotspot, both passed.

<img width="1430" height="751" alt="6-ping-tests" src="https://github.com/user-attachments/assets/e0bed1d9-087e-4737-bb94-5546533755da" />


### Shared Folder Test

```bash
mount | grep vboxsf
```

Output: `Downloads on /media/sf_Downloads type vboxsf (rw,nodev,relatime)`, confirming the Mac's Downloads folder is mounted inside Kali.

<img width="1431" height="71" alt="7-mount" src="https://github.com/user-attachments/assets/c83159d0-30e2-4d7f-9ccb-4af5b73237f3" />


## Step 6: Snapshot

After shutting Kali down (`sudo poweroff`), I took a snapshot named **Week 1 Lab Setup**. This is my clean baseline for future exercises, sitting after the `wazuh lab done V` snapshot.

<img width="1438" height="829" alt="8-snapshot" src="https://github.com/user-attachments/assets/8a39a197-5e2a-488a-a60f-48a6fb8c0f96" />


## Problems I Faced and How I Solved Them

**1. "Invalid settings detected" in VM settings**
VirtualBox warned that more virtual CPUs (3) were assigned than my Mac has (2). I reduced the VM to 2 CPUs and the warning cleared. Kali also runs more smoothly now.

**2. Kali couldn't get an IP address**
eth0 stayed on *"getting IP configuration"*. Setting a static IP with `ipv4.dad-timeout 0` and restarting the connection fixed it.

**3. Gateway worked, but no internet**
`ping 10.0.0.1` succeeded, but `ping 8.8.8.8` returned *"Destination Net Unreachable"* from 10.0.0.1, and `google.com` failed to resolve. The cause wasn't Kali at all: there was a power outage, my router went off, and my Mac had no internet. A VM on a NAT Network can only reach the internet through its host. I connected the Mac to my phone's hotspot and both tests passed.

**Lesson:** when the gateway responds but the internet doesn't, check the host machine's connection before changing anything in the VM.

## What I Learned

- How a VirtualBox NAT Network lets VMs talk to each other while sharing the host's internet connection
- Setting a static IP on Kali with `nmcli`, and why `dad-timeout` matters on newer VirtualBox/Kali versions
- Reading ping errors to find *where* a connection breaks (VM, gateway, or host)
- Using snapshots to protect existing work before making changes
- Matching VM resources to the host's real hardware
- Adapting Windows-based instructions to macOS


## Author

**Emmanuel-Osagie Esosa Ifeoluwa**
Cybersecurity Intern, Batch B084
Networkwalks Academy

LinkedIn:https://www.linkedin.com/in/esosa-ifeoluwa-emmanuel-4b2364349

## Project Information

- **Program name:** Networkwalks Cybersecurity Internship
- **Week:** 01
- **Project:** Cybersecurity & Pentesting Lab Setup
- **Repository:** GitHub

Kali Linux lab on a VirtualBox NAT Network, built on macOS.
