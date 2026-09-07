---
layout: post
title: "Cyberdefenders Mr. Robot"
date: 2026-09-03
---

## Introduction

Hello! Today, I am solving a CyberDefenders lab named **MrRobot**. It is a memory-forensics-based lab in which we will use Volatility.

I am using REMnux, which comes with Volatility 3. However, the lab appears to have been designed for Volatility 2, so I downloaded the standalone Volatility 2 executable. This is the easiest option because it does not require manual installation or building from source.

Throughout this write-up, I will explain not only which commands we can use, but also why we use them and the reasoning behind each step.

## Identifying the Volatility profile

Before analysing the memory image, I stored its path in the `$T1` variable:

```bash
T1=/home/remnux/Downloads/temp_extract_dir/target1/Target1-1dd8701f.vmss
```

Volatility 2 requires a profile describing the operating system version, architecture, and relevant kernel structures. I used `imageinfo` to identify suitable profiles:

```bash
./volatility_2.6_lin64_standalone -f $T1 imageinfo
```

![profile]({{ "/assets/images/posts/mrrobot/profile.webp" | relative_url }})

The plugin suggested the following profiles:

```text
Win7SP1x86_23418, Win7SP0x86, Win7SP1x86
```

The results indicate a 32-bit Windows 7 memory image. I selected `Win7SP1x86`, which successfully parsed the system structures and produced consistent results throughout the investigation:

```text
--profile=Win7SP1x86
```

All subsequent Volatility commands therefore use this profile.


## Question 1: Phishing email address

> **Machine:** Target1  
> **Question:** What email address tricked the front desk employee into installing a security update?

First, I moved to the directory containing the Volatility 2 standalone executable:

```bash
cd volatility_2.6_lin64_standalone/
```

I stored the path to the Target1 memory image in the `$T1` variable:

```bash
T1=/home/remnux/Downloads/temp_extract_dir/target1/Target1-1dd8701f.vmss
```

Next, I listed the active processes:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 pslist
```

I noticed that `OUTLOOK.EXE` was running with PID `3196`. Because Outlook's process memory could contain email data, I created a directory for the output:

```bash
mkdir dump
```

I then dumped the process memory:

```bash
./volatility_2.6_lin64_standalone -f "$T1" --profile=Win7SP1x86 memdump -p 3196 -D dump/
```

`memdump` does not extract only the heap. It dumps the resident pages belonging to the process's virtual address space. These pages may contain heaps, thread stacks, executable code, loaded DLLs, and memory-mapped data.

Finally, I extracted printable strings from the dump and searched for email sender fields:

```bash
strings dump/3196.dmp | grep -i 'from' | more
```

The output revealed a suspicious email impersonating AllSafe IT Support:

![email extracted from outlook memory]({{ "/assets/images/posts/mrrobot/email.webp" | relative_url }})

```text
From: The Whit3R0s3 <th3wh1t3r0s3@gmail.com>
```

The message instructed the employee to download a supposed security update named `AnyConnectInstaller.exe`.

**Answer:** `th3wh1t3r0s3@gmail.com`

## Question 2: Delivered filename

> **Machine:** Target1  
> **Question:** What is the filename that was delivered in the email?

The suspicious email identified in the previous question instructed the employee to download a supposed security update from the following URL:

```text
http://180.76.254.120/AnyConnectInstaller.exe
```

The filename can be extracted from the final part of the URL.

**Answer:** `AnyConnectInstaller.exe`

## Question 3: RAT family

> **Machine:** Target1  
> **Question:** What is the name of the RAT family used by the attacker?

I used OSINT techniques and searched for the malicious URL identified in the email:

```text
http://180.76.254.120/AnyConnectInstaller.exe
```

This led me to an [ANY.RUN sandbox report](https://any.run/report/94a4ef65f99c594a8bfbfbc57f369ec2b6a5cf789f91be89976086aaa507cd47/53a0144b-7d73-4474-9d2a-583bfa4fd1fa) for `AnyConnectInstaller.exe`.

The report contains indicators associated with Xtreme RAT, including:

- An embedded resource named `XTREME`
- Registry keys created under `HKEY_CURRENT_USER\Software\Xtreme`
- Persistence through Windows `Run` registry keys
- Code execution inside `svchost.exe`

Based on these indicators, the malware belongs to the Xtreme RAT family.

**Answer:** `XtremeRAT`

## Question 4: Injected process

> **Machine:** Target1  
> **Question:** The malware appears to be leveraging process injection. What is the PID of the process that is injected?

I used the third-party `hollowfind` plugin, which detects indicators of process hollowing.

```bash
../volatility_2.6_lin64_standalone/volatility_2.6_lin64_standalone --plugins=. -f "$T1" --profile=Win7SP1x86 hollowfind
```

The `--plugins=.` option tells Volatility to load additional plugins from the current directory (`.`). In this case, that directory contains the `hollowfind.py` plugin.

### Detecting Process Hollowing

`hollowfind` identified inconsistencies between the process's **PEB (Process Environment Block)** and **VADs (Virtual Address Descriptors)**:

```text
Base Address (VAD): 0x12d0000
Base Address (PEB): 0x13400000
```
![hollowfind]({{ "/assets/images/posts/mrrobot/hollowfind.webp" | relative_url }})


The **PEB** is a user-mode structure containing metadata about the running process, including the base address of its main executable image. **VADs** are kernel-maintained structures describing the process's virtual memory regions.

Normally, the VAD containing the main executable image should correspond with `PEB.ImageBaseAddress`. This mismatch therefore suggests suspicious process memory manipulation.

Additionally, `ldrmodules` showed:

```text
./volatility_2.6_lin64_standalone -f "$T1" --profile=Win7SP1x86 ldrmodules -p 2996
```

![ldrmodules]({{ "/assets/images/posts/mrrobot/ldrmodules.webp" | relative_url }})


The `iexplore.exe` image mapping was identified through VAD analysis but was absent from all three PEB loader lists.

Together, these **cross-view inconsistencies** are strong indicators consistent with **process hollowing**, although they are not proof by themselves.

**Answer:** `2996`

## Question 5: Persistence value

> **Machine:** Target1  
> **Question:** What is the unique value the malware is using to maintain persistence after reboot?

Windows `Run` registry keys are commonly abused by malware to start automatically when a user logs on. To inspect the registry, I first listed the available registry hives:

```bash
./volatility_2.6_lin64_standalone -f "$T1" --profile=Win7SP1x86 hivelist
```

The output showed the `SOFTWARE` hive at virtual offset `0x8b79d008`:

![hivelist]({{ "/assets/images/posts/mrrobot/hivelist.webp" | relative_url }})


I supplied this offset to `printkey` and inspected the Windows `Run` key:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 printkey -o 0x8b79d008 -K 'Microsoft\Windows\CurrentVersion\Run'
```

The `-o` option specifies the virtual offset of the registry hive, while `-K` specifies the key to inspect.

The output contained the following suspicious value:

![hivelist]({{ "/assets/images/posts/mrrobot/printkey.webp" | relative_url }})


Here, `MrRobot` is the registry value name, while the associated data points to the malware executable. This entry causes the malware to start when a user logs on.

**Answer:** `MrRobot`

## Question 6: Malware mutex

> **Machine:** Target1  
> **Question:** Malware often uses a unique value or name to ensure that only one copy runs on the system. What is the unique name the malware is using?

Malware can create a named mutex when it starts. If another instance finds that the mutex already exists, it can terminate instead of running a second copy.

Windows internally refers to mutex objects as `Mutant` objects. I therefore listed the mutant handles belonging to the injected `iexplore.exe` process:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 handles -p 2996 -t mutant
```

Among several legitimate-looking mutexes, one name stood out:

```text
fsociety0.dat
```

![mutant]({{ "/assets/images/posts/mrrobot/mutant.webp" | relative_url }})


Although the name ends in `.dat`, this is not evidence of a file. The `Type` column identifies it as a `Mutant`, meaning a Windows mutex object.

Its unusual name and association with the injected process indicate that it is the malware's custom mutex, likely used to prevent multiple instances from running.

**Answer:** `fsociety0.dat`

## Question 7: Notorious hacker

> **Machine:** Target1  
> **Question:** It appears that a notorious hacker compromised this box before our current attackers. Name the movie he or she is from.

I first inspected the SAM registry hive, but it contained only the `Administrator` and `front-desk` accounts. I therefore searched memory for file objects referencing user-profile directories:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 filescan | grep -i 'users'
```

The output contained the following path:

```text
\Device\HarddiskVolume2\Users\zerocool\AppData\Roaming\Microsoft\Windows\SendTo\Desktop.ini
```

This indicates that a profile directory named `zerocool` existed on the system. The account itself was not present in the SAM, suggesting that it may have been deleted while traces of its profile remained.

`Zero Cool` is the alias used by Dade Murphy, the protagonist of the 1995 movie *Hackers*.

**Answer:** `Hackers`

## Question 8: Administrator NTLM hash

> **Machine:** Target1  
> **Question:** What is the NTLM password hash for the Administrator account?

I used Volatility's `hashdump` plugin to extract the local account password hashes:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 hashdump
```

![hashdump]({{ "/assets/images/posts/mrrobot/hashdump.webp" | relative_url }})


Windows does not store the plaintext passwords of local accounts. It stores protected password hashes in the SAM registry hive.

The relevant data is distributed between two hives:

- `SAM` contains local account information and encrypted LM/NTLM hashes.
- `SYSTEM` contains the material required to reconstruct the system boot key.

Therefore, `hashdump` removes the SAM protection layer and returns the underlying password hashes. It does not recover the plaintext passwords.

The output uses the following format:

```text
username:RID:LM hash:NTLM hash:::
```

The Administrator entry was:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:79402b7671c317877b8b954b3311fa82:::
```

The fields represent:

- `Administrator` — account name
- `500` — RID of the built-in Administrator account
- `aad3b435b51404eeaad3b435b51404ee` — placeholder indicating that no LM hash is stored
- `79402b7671c317877b8b954b3311fa82` — NTLM password hash

**Answer:** `79402b7671c317877b8b954b3311fa82`

## Question 9: Tools transferred by the attacker

> **Machine:** Target1  
> **Question:** The attackers appear to have moved over some tools to the compromised front desk host. How many tools did the attacker move?

Attackers commonly stage tools in temporary or download directories. I therefore used `filescan` and searched for paths containing `\Temp\`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 filescan | grep -i '\\temp\\'
```

`filescan` scans memory for Windows file objects. The results included four suspicious tools under `C:\Windows\Temp`:

![filescan]({{ "/assets/images/posts/mrrobot/filescan.webp" | relative_url }})


```text
C:\Windows\Temp\wce.exe
C:\Windows\Temp\getlsasrvaddr.exe
C:\Windows\Temp\Rar.exe
C:\Windows\Temp\nbtscan.exe
```

Their likely purposes are:

- `wce.exe` — Windows Credential Editor, used to extract or manipulate Windows credentials.
- `getlsasrvaddr.exe` — a utility associated with locating information needed for credential-related operations.
- `nbtscan.exe` — a network-reconnaissance tool used to scan for NetBIOS systems.
- `Rar.exe` — a legitimate command-line archiving utility that can be abused to compress collected data or unpack attacker tooling.

`wce.exe` appeared more than once in the output, but these entries refer to the same filename and should be counted as one tool.

The temporary-directory location does not prove that every file is malicious. However, the combination of their functionality, location, and surrounding evidence indicates that they were likely staged by the attacker.

**Answer:** `4`

## Question 10: Front desk Administrator password

> **Machine:** Target1  
> **Question:** What is the password for the front desk local Administrator account?

I examined the command history using Volatility's `cmdscan` plugin:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 cmdscan
```

The output showed that the attacker executed Windows Credential Editor (`wce.exe`) and redirected its output to `w.tmp`:

```text
wce.exe -w
wce.exe -w > w.tmp
```
Windows Credential Editor (`wce.exe`) is a credential-dumping tool capable of accessing authentication material stored in the memory of `lsass.exe`. The `-w` option attempts to retrieve plaintext passwords associated with active logon sessions. On older Windows systems, authentication packages such as WDigest could retain reusable plaintext credentials in LSASS memory to support single sign-on. WCE reads these credentials directly from memory—it does not obtain them by cracking NTLM hashes.

I located the corresponding file object in the earlier `filescan` output:

```text
0x000000003eca37f8 \Device\HarddiskVolume2\Windows\Temp\w.tmp
```

The first column is the physical offset of the `_FILE_OBJECT` structure. I supplied it to `dumpfiles` using `-Q`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 dumpfiles -Q 0x000000003eca37f8 --dump-dir dump2/
```

Volatility recovered the file as:

```text
file.None.0x85b684b0.dat
```

I displayed its contents:

```bash
cat dump2/file.None.0x85b684b0.dat
```

![password]({{ "/assets/images/posts/mrrobot/password.webp" | relative_url }})


The recovered WCE output contained the local Administrator credentials:

```text
Administrator\front-desk-PC:flagadmin@1234
```

**Answer:** `flagadmin@1234`

## Question 11: `nbtscan.exe` creation timestamp

> **Machine:** Target1  
> **Question:** What is the STD create date timestamp for the `nbtscan.exe` tool?

NTFS stores file metadata in Master File Table (MFT) records. I used Volatility's `mftparser` plugin to recover MFT records from memory and searched for `nbtscan`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 mftparser | grep -A 10 nbtscan
```

![mftparser]({{ "/assets/images/posts/mrrobot/mftparser.webp" | relative_url }})

The `-A 10` option tells `grep` to display the matching line and the following ten lines.

The recovered entry was:

```text
2015-10-09 10:45:12 UTC+0000 2015-10-09 10:45:12 UTC+0000 2015-10-09 10:45:12 UTC+0000 2015-10-09 10:45:12 UTC+0000 Windows\Temp\nbtscan.exe
```

The first timestamp is the file creation time from the NTFS `$STANDARD_INFORMATION` attribute.

**Answer:** `2015-10-09 10:45:12 UTC+0000`

## Question 12: First machine found by `nbtscan`

> **Machine:** Target1  
> **Question:** The attackers stored the output from `nbtscan.exe` in a text file called `nbs.txt`. What is the IP address of the first machine in that file?

I used `filescan` to locate the `nbs.txt` file object in memory:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 filescan | grep -i 'nbs'
```

The file was found at the following physical offset:

```text
0x000000003fdb7808 \Device\HarddiskVolume2\Windows\Temp\nbs.txt
```

I supplied the `_FILE_OBJECT` offset to `dumpfiles`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 dumpfiles -Q 0x000000003fdb7808 --dump-dir dump2/
```

Volatility recovered the file as `file.None.0x83eda598.dat`. I displayed its contents:

```bash
cat dump2/file.None.0x83eda598.dat
```

The file contained:

```text
10.1.1.2     ALLSAFECYBERSEC\AD01           SHARING DC
10.1.1.3     ALLSAFECYBERSEC\EX01           SHARING
10.1.1.20    ALLSAFECYBERSEC\FRONT-DESK-PC  SHARING
10.1.1.21    ALLSAFECYBERSEC\GIDEON-PC       SHARING
```

The first machine listed is `AD01`.

**Answer:** `10.1.1.2`

## Question 13: Malware network connection

> **Machine:** Target1  
> **Question:** What full IP address and port was the attacker's malware using?

I used Volatility's `netscan` plugin to examine network connections present in memory:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 netscan
```

The output showed an established TCP connection associated with `iexplore.exe` and PID `2996`, which was previously identified as the injected process:

```text
TCPv4  10.1.1.20:49205  180.76.254.120:22  ESTABLISHED  2996  iexplore.exe
```

![cmdscan]({{ "/assets/images/posts/mrrobot/cmdscan.webp" | relative_url }})

The fields show:

- `10.1.1.20:49205` — local address and ephemeral source port
- `180.76.254.120:22` — remote address and destination port
- `ESTABLISHED` — the connection was active
- `2996` — PID of the injected `iexplore.exe` process

This connects the previously identified injected process to the attacker-controlled IP address.

**Answer:** `180.76.254.120:22`

## Question 14: Remote administration software

> **Machine:** Target1  
> **Question:** It appears the attacker also installed legitimate remote administration software. What is the name of the running process?

I reviewed the active processes using `pslist`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 pslist
```

The output contained several processes associated with TeamViewer, including:

![TeamViewer]({{ "/assets/images/posts/mrrobot/teamviewer.webp" | relative_url }})

The primary TeamViewer process was `TeamViewer.exe` with PID `2680`.

**Answer:** `TeamViewer.exe`

## Question 15: Built-in remote access

> **Machine:** Target1  
> **Question:** It appears the attackers also used a built-in remote access method. What IP address did they connect to?

I examined the network connections using `netscan`:

```bash
./volatility_2.6_lin64_standalone -f $T1 --profile=Win7SP1x86 netscan
```

The output showed an established connection associated with `mstsc.exe`:

![mstsc]({{ "/assets/images/posts/mrrobot/mstsc.webp" | relative_url }})

`mstsc.exe` is the built-in Microsoft Remote Desktop Connection client, while TCP port `3389` is the standard port used by RDP.

The local system at `10.1.1.20` connected to the remote system at `10.1.1.21`. Based on the earlier `nbtscan` results, this address belonged to `GIDEON-PC`.

**Answer:** `10.1.1.21`