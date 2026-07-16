---
layout: page
title: KrakenKeylogger
permalink: /cyberdefenders_krakenkeylogger/
---

Today I am going to solve the KrakenKeylogger lab on the CyberDefenders site.

### 1. What is the web messaging app the employee used to talk to the attacker?

At the beginning, I searched for any application installed in Program Files. Later, I searched different directories as well, but I was not able to find anything. Then I discovered `wpndatabase.db`.

`wpndatabase.db` is a local database file in Windows 10 and Windows 11 that stores system and application notifications. It belongs to the Windows Push Notification (WPN) service.

File location:

```text
Users/OMEN/AppData/Local/Microsoft/Windows/Notifications/wpndatabase.db
```

You can open it with SQLiteBrowser.

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/wpndatabase_db.webp' | relative_url }}"
    alt="WPN database"
    width="900">

  <figcaption>
    SQLite database containing notification artifacts.
  </figcaption>
</figure>

When you search through all records, you can find:

<pre>
&lt;toast launch=&quot;0|0|Default|0|https://web.[REDACTED].org/|p#https://web.[REDACTED].org/#&quot; displayTimestamp=&quot;2023-07-11T16:57:15Z&quot;&gt;
 &lt;visual&gt;
  &lt;binding template=&quot;ToastGeneric&quot;&gt;
   &lt;text&gt;Nawaf&lt;/text&gt;
   &lt;text&gt;[REDACTED]lt;/text&gt;
   &lt;text placement=&quot;attribution&quot;&gt;web.[REDACTED].org&lt;/text&gt;
   &lt;image placement=&quot;appLogoOverride&quot; src=&quot;C:\Users\OMEN\AppData\Local\Google\Chrome\User Data\Notification Resources\cd18935b-57e3-4838-b5e3-ef360362f771.tmp&quot; hint-crop=&quot;none&quot;/&gt;
  &lt;/binding&gt;
 &lt;/visual&gt;
 &lt;actions&gt;
  &lt;action content=&quot;[REDACTED]#&quot;/&gt;
 &lt;/actions&gt;
&lt;/toast&gt;
</pre>

It becomes clear which application was used.

### 2. What is the password for the protected ZIP file sent by the attacker to the employee?

A closer look at the toast message reveals the password.

### 3. What domain did the attacker use to download the second stage of the malware?

The toast message suggests that a ZIP archive was downloaded. Let's find it:

```bash
find . -name '*.zip'
```

Result:

```text
./Users/OMEN/Downloads/project templet test.zip
```

When you go to that folder, you realize it was already unzipped. Inspect the `template.lnk` file.

It was tricky for me because usually you would use LECmd on Windows. However, on Linux you need to install the .NET environment and LECmd. I did that, but for some reason some data was truncated.

I searched further and realized ExifTool can be used to inspect the file contents.

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/template_data.webp' | relative_url }}"
    alt="Template data"
    width="900">

  <figcaption>
    Obfuscated PowerShell script inside template.lnk
  </figcaption>
</figure>

I used ChatGPT to recreate the string deobfuscation logic in Python and decode the URL.

```python
def decode_string(s: str) -> str:
    """
    Replicates the PowerShell string deobfuscation logic.
    """

    s = s[::-1]

    decoded = "".join(
        s[i:i+2][::-1]
        for i in range(0, len(s), 2)
    )

    return decoded


if __name__ == "__main__":

    encoded = 'aht1.sen/hi/coucys.erstmaofershma//s:tpht'

    decoded = decode_string(encoded)

    print(decoded)
```

The script prints the URL used by the attacker.

### 4. What is the name of the command that the attacker injected using one of the installed LOLAPPS on the machine to achieve persistence?

I opened the following site to read more about LOLAPPS:

https://lolapps-project.github.io/

Now we need to determine which application was used in this case.

I used Regripper on Kali Linux to parse the `SOFTWARE` hive and understand what is executed during system startup.

Please note that it is usually a good idea to merge transaction logs to get the full picture.

```bash
regripper -r SOFTWARE -p run
```

It becomes clear that the application used in the attack was Greenshot.

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/regripper.webp' | relative_url }}"
    alt="Regripper output"
    width="900">

  <figcaption>
    Regripper used to inspect Windows registry startup entries
  </figcaption>
</figure>

Based on the previously referenced site, we can read more about Greenshot persistence techniques.

> Any Greenshot plugin can contain persistence...

Let's find the `.ini` file:

```bash
find . -name 'Greenshot.ini'
```

Location:

```text
./Users/OMEN/AppData/Roaming/Greenshot/Greenshot.ini
```

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/greenshot.webp' | relative_url }}"
    alt="Greenshot.ini"
    width="900">

  <figcaption>
    Greenshot.ini configuration file
  </figcaption>
</figure>

### 5. What is the complete path of the malicious file that the attacker used to achieve persistence?

Based on the previous question and my research, every time Windows starts, Greenshot starts as well. Therefore, all commands configured in `Greenshot.ini` are potential persistence mechanisms.

### 6. What is the name of the application the attacker utilized for data exfiltration?

Based on MITRE ATT&CK technique T1219, an adversary may use legitimate remote access software to establish command and control communication.

At this point, our task becomes determining whether any remote access software is installed.

I did not find anything interesting in Program Files, so I used a more basic approach and manually listed directories. Eventually I found:

```text
Users/OMEN/AppData/Roaming/[REDACTED]
```

`AppData/Roaming` stores user-specific application data that follows the user profile.

### 7. What is the IP address of the attacker?

Inside that directory we can observe the `ad.trace` file, which stores logs.

After researching how IP-related entries are logged, I searched for strings such as `from`, which revealed the attacker's IP address.

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/ip.webp' | relative_url }}"
    alt="Attacker IP"
    width="900">

  <figcaption>
    Attacker IP address logged inside ad.trace
  </figcaption>
</figure>

### Conclusion

This lab demonstrates how attackers can combine social engineering, LOLAPPS, PowerShell obfuscation, and legitimate remote access tools to compromise a system and maintain persistence.

It is also a good reminder that valuable forensic artifacts are often hidden in less obvious locations such as notification databases, .lnk metadata, application logs, and configuration files.