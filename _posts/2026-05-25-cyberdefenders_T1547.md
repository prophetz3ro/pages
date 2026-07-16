---
layout: page
title: T1547 Lab
permalink: /cyberdefenders_t1547/
---

### 1. To determine how the fileless malware was executed on the device, we need to determine which trusted Windows process it hijacked. What is the name of the Windows-native binary used to run malicious code?

Ok, so let's start our investigation. We only have one file: `NTUSER.DAT`.

We can open it with Registry Explorer, but where are we supposed to look first?

The first idea that came to my mind was, of course, the startup key:

`HKLM\Software\Microsoft\Windows\CurrentVersion\Run`

I was not mistaken — when you jump to that key, you can immediately see the answer to the first question.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/run_key.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    HKLM\Software\Microsoft\Windows\CurrentVersion\Run
  </figcaption>
</figure>

### 2. Understanding how the malware maintains its presence in the system without a file is crucial for detecting and eradicating the infection. What is the value name of the register string key that contains the malicious script?

If you take a closer look at the screenshot above, you will find it quickly.

The registry value contains heavily obfuscated JavaScript responsible for launching the next stage.

### 3. To fully comprehend the attacker's approach in concealing the malware, it is imperative to determine the encryption technique and the key value utilized for decrypting the second stage. What is the exact key value required for decrypting the second stage of the malware?

When you navigate to the key from question 2, you can observe a strongly obfuscated JavaScript payload.

After a short analysis, it becomes clear that an XOR cipher was used. The key is stored in the `gsHFf3JWfO` variable.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/js_payload.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    JS payload in registry key
  </figcaption>
</figure>

### 4. To fully understand the scope of the fileless malware's payload, we need to find where it is stored. What is the value name of the registry string key that contains the encrypted executable?

Let's continue with the JavaScript payload.

After a short cleanup, the script becomes easy to analyze. The important part is replacing the `eval()` function with `console.log()` so the decrypted payload is printed instead of executed.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/js_execution.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    Cleaned JS payload prepared for analysis
  </figcaption>
</figure>

Now we can safely run the modified script in the Chrome developer console.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/js_execution_result.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    JS payload result
  </figcaption>
</figure>

The highlighted part is, of course, Base64 encoded. We have been given CyberChef, so let's decode it.

The decoded script contains another registry key/value pair.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/key_name.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    Registry key/value used by the next stage
  </figcaption>
</figure>

### 5. Decrypting the malware's executable will allow us to analyze its behavior. What is the key value used to decrypt the executable?

We can explore the decoded script further. You will eventually reach the following section.

These are ASCII characters represented as hexadecimal values.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/executable_key.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    Key encoded as ASCII hex
  </figcaption>
</figure>

### 6. Comparing the hash of the decrypted executable against known threats can confirm its identity and inform our response strategy. What is the MD5 hash of the decrypted executable?

I solved this question in a rather simple way.

I exported the payload value from the artifact's registry and imported it into the lab virtual machine. Then I ran a slightly modified PowerShell script that we decoded previously.

Unfortunately, it was taking a long time to decrypt the payload and compute the hash, so I stopped it and asked ChatGPT to write a Python script which performs the same task.

The Python implementation was much faster. The PowerShell version was slow because it continuously reallocated arrays during XOR operations, while Python handled the byte processing much more efficiently.

In hindsight, the better solution would have been to export the registry value directly as binary data and decrypt it offline with Python. I realized that after obtaining the hash.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/decrypting.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    Decrypting binary
  </figcaption>
</figure>

I am attaching the Python script for reference.

<pre>
import hashlib
import winreg

key = b"[REDACTED]"

with winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\nsem") as k:
    data, _ = winreg.QueryValueEx(k, "zorsuhg")

encrypted = bytes(data)

decrypted = bytes(
    b ^ key[i % len(key)]
    for i, b in enumerate(encrypted)
)

print(hashlib.md5(decrypted).hexdigest())
</pre>

### 7. Identifying the malware family can provide context about the threat actors and their methodologies. What is the name of the malware family associated with this attack based on tria.ge?

Once you have the MD5 hash of the executable, it's relatively easy to look up the malware using online services.

Simply visit:

`https://tria.ge/[MD5_HASH]`

It presents useful information about the executable, including the malware family name.

### 8. Understanding all decryption keys used by the malware can aid in revealing the full scope of the threat. What is the key used to decrypt the second stage of the executable?

Firstly, we need to save the executable as a file so we can examine it with dnSpy.

I modified the decrypting script a bit so the decrypted content is saved as `payload.exe` on disk.

<pre>
import hashlib
import winreg
from pathlib import Path

REG_PATH = r"Software\nsem"
REG_VALUE = "zorsuhg"
XOR_KEY = b"[REDACTED]"

OUT_FILE = Path("payload.exe")

with winreg.OpenKey(winreg.HKEY_CURRENT_USER, REG_PATH) as key:
    data, _ = winreg.QueryValueEx(key, REG_VALUE)

encrypted = bytes(data)

decrypted = bytes(
    byte ^ XOR_KEY[i % len(XOR_KEY)]
    for i, byte in enumerate(encrypted)
)

OUT_FILE.write_bytes(decrypted)

print(f"Saved: {OUT_FILE.resolve()}")
print(f"Size: {len(decrypted)} bytes")
print(f"MD5: {hashlib.md5(decrypted).hexdigest()}")
print(f"SHA256: {hashlib.sha256(decrypted).hexdigest()}")
</pre>

Now we can examine it with dnSpy.

It's a good idea to check the entry point first, which leads us to the `Stock` class. After reviewing its properties and methods, we can find the `Array()` method performing another XOR operation.

We can also see the second-stage key stored in a local variable.

<figure>
  <img
    src="{{ '/assets/images/posts/T1547/sec_stage_key.webp' | relative_url }}"
    alt=""
    width="900">

  <figcaption>
    Array method
  </figcaption>
</figure>

### Conclusion

This lab is a good example of how fileless malware abuses trusted Windows components, PowerShell, registry storage, and in-memory .NET execution to avoid dropping files on disk.

Even though the payload was heavily obfuscated, simple cleanup and step-by-step analysis were enough to fully reconstruct the execution chain and recover the encrypted payload.