Step 1: Access the Windows machine.

[Content Image](https://assets.ine.com/lab/learningpath/fa8f67890732a493f012184e373948447b345a5ecec6d89dd9c53b7bffedf15a.png)

Step 2: Start Wireshark and open the mal01.pcapng file located on the Desktop.

[Content Image](https://assets.ine.com/lab/learningpath/0602826feda6e0129f97a20e9ae06c37d1dca011f32835c12834acbe57adf223.png)

Step 3: Filter for HTTP traffic to investigate potential C2 activity, as HTTP is a common channel used by malware for command and control communications.

Filter:
```
http
```
We immediately notice a suspicious GET request to /checkmate.exe from 10.0.0.156 to 10.0.0.155. Downloading an executable file over HTTP is unusual.

[Content Image](https://assets.ine.com/lab/learningpath/bf0922b486b413eda876b9a59fef7eae02204350ea51dd378e519c660ce25d5b.png)

Right-click on the frame and go to Follow > HTTP Stream.

[Content Image](https://assets.ine.com/lab/learningpath/77abc7a34d6afe38b021178a313c766c84394414b29cbfaead27c5ff8b494b05.png)


[Content Image](https://assets.ine.com/lab/learningpath/bdbc5083ebe1ed777b7ff4592d467f2fb7eed3bb80ef1e723074ea7c9bbc1af7.png)

Based on this HTTP stream, we can identify several key indicators:
1. PowerShell as User-Agent: The request is coming from WindowsPowerShell/5.1.17763.1432, which means this executable is being downloaded via a PowerShell command (likely Invoke-WebRequest or wget). This is a common technique in attack scenarios.
2. Python SimpleHTTP Server: The server hosting the executable is SimpleHTTP/0.6 Python/3.12.8, which is a lightweight, temporary web server often used for quick file transfers rather than legitimate software distribution.
3. Executable File Type: The content-type application/x-msdos-program confirms this is a Windows executable (.exe file).
4. Internal Network Communication: The communication is between two internal IPs (10.0.0.156 requesting from 10.0.0.155:8000), suggesting lateral movement or internal file distribution.
5. Non-standard Port: Port 8000 is being used instead of standard HTTP port 80, which is typical for ad-hoc Python SimpleHTTP servers.
Step 4: Let's extract the checkmate.exe executable, calculate its hash, and check it against a threat intelligence database like VirusTotal to confirm whether it is malicious.

Go to File > Export Objects > HTTP.

[Content Image](https://assets.ine.com/lab/learningpath/556cb6ec1e657bda5506a374a9a2ea1cffb6802422ddd1954dbd0fa54e361a3d.png)

Select the checkmate.exe file and click on Save. Save it to the desktop.

[Content Image](https://assets.ine.com/lab/learningpath/c264be4ef19815f6a6fb93f329d18205a997e3e409df727800d8807a6b98a7d7.png)

Next, open PowerShell and caluclate the MD5 hash of the executable.

Commands:
```
cd Desktop

Get-FileHash -Algorithm MD5 checkmate.exe
```

[Content Image](https://assets.ine.com/lab/learningpath/6f9218fc4b92326c19531e2d41449cae8396bf35ca7841acf3298f0aeb3706d2.png)

Copy the MD5 hash and search for it on VirusTotal at https://www.virustotal.com.

[Content Image](https://assets.ine.com/lab/learningpath/3819c1f5e8a03b9d877755051db5cbb983acab4da43f35c105705d2f1732618b.png)

The hash returns no results on VirusTotal, indicating this may be a previously unknown or custom-built executable.

Step 5: Back in Wireshark, we can observe that after the initial request to download the executable, there are multiple HTTP POST requests and 200 OK response exchanges between 10.0.0.156 and 10.0.0.155.

[Content Image](https://assets.ine.com/lab/learningpath/21085bc6b4a5e4f4ed9b13865dd0cc87b95ca991b0ff70888e3510177a04586b.png)

Right-click on these requests and follow the HTTP stream. Notice that they either contain encrypted content, unusual data, or empty responses.

[Content Image](https://assets.ine.com/lab/learningpath/e19a170d17459221b35bcca9f6814fd7e7c253e4a2f8599abb90403b54857389.png)

Let's visualize this traffic pattern. First, apply the following filter to narrow down the HTTP traffic involving only 10.0.0.156 and 10.0.0.155.

Filter:
```
http && ip.addr==10.0.0.155 && ip.addr==10.0.0.156
```

[Content Image](https://assets.ine.com/lab/learningpath/2e559f696a8c1924b4ab51d028c2864fe3f809b31bfc441da10fbed33488f844.png)

Next, go to Statistics > I/O Graphs.

[Content Image](https://assets.ine.com/lab/learningpath/674c416cf1cfc3fe801aad0999ea0db5413ce050ba7540753f8ed94e3552b30e.png)

Make sure the graph only visualizes the filtered packets. Uncheck the remaining boxes as shown below:

[Content Image](https://assets.ine.com/lab/learningpath/18e1197dcbe1f552d792776f5c09332aae1cd85e1c5017862484384051bb15d1.png)

We observe beaconing activity occurring approximately every 2 seconds, which is indicative of potential C2 activity. Beaconing is when a compromised system periodically sends automated requests to a C2 server to check for new commands or maintain the connection.

Now the key questions arise: What kind of C2 are we dealing with? How do we identify the C2 framework? Is it a well-known framework or something custom?

The solution is to gather evidences that may point us to a known C2 framework.

Step 6: For example, examine the first POST request from the victim machine after the executable is downloaded and possibly executed.

[Content Image](https://assets.ine.com/lab/learningpath/b9651702342f351bdc2e026b24090df688b21fd51b06a022d3e40db331935c00.png)

Upon closer inspection, the first few bytes read dead beef. This is a well know magic byte value associated with Havoc C2.

This can be confirmed by checking the Defines.h file in the Havoc C2 github repository.

[Content Image](https://assets.ine.com/lab/learningpath/df215581d7bdb6d8f4a91da009c4b4fb3a01947ea54070020f6cbeaabcb26cf1.png)

Link: https://github.com/HavocFramework/Havoc/blob/main/payloads/Demon/include/common/Defines.h

Step 7: Now that we have identified the framework as Havoc C2, we can apply known reverse-engineering and traffic analysis techniques to recover the AES key and IV used for encryption and decryption. While threat intelligence write-ups and automated scripts are available online for reference, the following section provides a concise, practical cheat sheet for analyzing Havoc C2 traffic.

Understanding AES Key and IV:
- AES Key: The Advanced Encryption Standard (AES) key is the secret cryptographic key used to encrypt and decrypt the C2 communication between the implant and the server. In Havoc C2, this key is embedded in both the implant and server to secure the traffic.
- IV (Initialization Vector): The IV is a random value used alongside the AES key to ensure that identical plaintext messages produce different ciphertext outputs. This prevents pattern recognition in encrypted traffic.
- Why they matter: Without the AES key and IV, the encrypted POST request data appears as random bytes. Extracting these values allows us to decrypt the C2 traffic and understand what commands are being sent and what data is being exfiltrated.
Select the first POST request, copy the packet bytes as a Hex Dump as shown below and paste it on notepad:

[Content Image](https://assets.ine.com/lab/learningpath/5923b0c316957a940c8fd3166db14671c978d55b8597895c069c3560061475b1.png)

```
0000   02 55 6a 9e 55 49 02 a5 f2 86 c8 b5 08 00 45 00
0010   01 11 36 b0 40 00 80 06 00 00 0a 00 00 9c 0a 00
0020   00 9b c4 bf 00 50 f1 e2 41 43 bf 83 ff ba 50 18
0030   20 14 16 3a 00 00 00 00 00 e5 de ad be ef 08 75
0040   c2 54 00 00 00 63 00 00 00 00 08 da 26 84 0e c4
0050   d8 c2 3e 32 5e ea e6 ea e6 48 f6 5a 2c d0 48 50
0060   6e 64 32 dc d2 c4 76 86 d6 8a 9a f8 84 b0 68 dc
0070   38 d0 2c a6 b2 ca 2c 8e 96 82 8f d3 51 ad 66 c7
0080   7c 4c 95 3f a4 a0 44 a1 bd 93 16 ff 56 13 32 13
0090   6a 92 7c 7c d5 db 7f 68 5c c9 b7 24 5c 2d b4 c3
00a0   ba 16 2b 5d 37 78 3e 8b fc 1a a2 eb f0 03 05 e8
00b0   81 05 c0 ac 9f 69 91 5d d3 15 e9 43 b2 f9 03 7f
00c0   7e 91 dc 2c b3 c5 f1 bf 41 76 a8 b6 ff 07 a7 d1
00d0   7e 57 13 c2 7b d8 16 12 43 08 4e e7 2e be 9f 99
00e0   a0 4d 93 30 fb de 37 27 30 9e 54 f8 46 b3 52 6b
00f0   59 37 99 6d 73 0a ec b6 36 9b 3e 31 6b 78 b6 36
0100   aa ce fe fb 23 71 95 74 a7 b3 03 1c 5d d4 f5 fb
0110   93 6f c7 b3 40 9c 0d f3 cf 8d 13 67 f0 b7 d9
```
In the above hex dump, locate the magic bytes de ad be ef, skip the next 12 bytes, and then extract the following 32 bytes, which represent the AES key:
```
08 da 26 84 0e c4 d8 c2 3e 32 5e ea e6 ea e6 48 f6 5a 2c d0 48 50 6e 64 32 dc d2 c4 76 86 d6 8a
```
Note that 1 byte = 2 hexadecimal characters.

Removing the white spaces, the AES Key is:
```
08da26840ec4d8c23e325eeae6eae648f65a2cd048506e6432dcd2c47686d68a
```
The next 16 bytes is the IV:
```
9af884b068dc38d02ca6b2ca2c8e9682
```
To summarize, the key extraction process for Havoc C2 is as follows:

Find deadbeef → skip 12 bytes → next 32 bytes = AES key → next 16 bytes = AES IV

Step 8: Next, observe that after the first POST request, the subsequent POST requests and responses are of fixed lengths:

[Content Image](https://assets.ine.com/lab/learningpath/99c69a4072837f2198c6b24df7345dad465a95a3345b91e18082acf177b549e8.png)

After a few of these exchanges, notice that there is a sudden and significant change in the length of the POST request. This likely indicates the output of a command executed by the C2 implant.

[Content Image](https://assets.ine.com/lab/learningpath/8ce736d15eec19712117c841135a9137c6cf84847b9808b52a47799611af8089.png)

Double-click on that frame and expand "Hypertext Transfer Protocol".

[Content Image](https://assets.ine.com/lab/learningpath/c0728f1a787452c588e8262f2c76526b139b068eb73390dc25ec26b2a61c06dd.png)

Right-click on File Data and select Copy > Value.

Go to CyberChef (https://gchq.github.io/CyberChef/) and paste it in the Input box:

[Content Image](https://assets.ine.com/lab/learningpath/a4ec30e82845ed5f618ff3530c33102d79c85ba0b79d5f049c8b017bde10c247.png)

We now have an AES-encrypted input that needs to be decrypted using the keys we discovered.

To do this, first drag & drop the AES Decrypt operation in the Recipe box.

[Content Image](https://assets.ine.com/lab/learningpath/15e8d70ab86be589a9ebc2f3440aab06fee2e50876048098bf8df05c947fbe3d.png)

Paste the AES Key and the IV in their respective fields.

[Content Image](https://assets.ine.com/lab/learningpath/5184011f95eb9572d8739d20647ec4261a8476423458578998e8926078f54fca.png)

Havoc C2 uses the CTR (Counter) block cipher mode for encrypting/decrypting data. This can be confirmed by examining the AesCrypt.h file in the Havoc C2 repository.

[Content Image](https://assets.ine.com/lab/learningpath/3af19e9344418257fda6cb8f4cf8ab60f6351a74ce598cd360f36a94c6a72d58.png)

Link: https://github.com/HavocFramework/Havoc/blob/ea3646e055eb1612dcc956130fd632029dbf0b86/payloads/Demon/include/crypt/AesCrypt.h

Set the mode to CTR in the recipe.

[Content Image](https://assets.ine.com/lab/learningpath/540a1b8bf3b501358bcb364f98ac0f168b76c348193bf0981df83c906ba3307d.png)


[Content Image](https://assets.ine.com/lab/learningpath/58f0d3a469fe07083e905fc388730aabf6beac9e7affe51fe2aa498e9bc69015.png)

Notice that we still see garbage in the Output. This is because the input content contains headers that need to be removed.

Remove exactly 20 bytes (40 characters) from the beginning of the input.

[Content Image](https://assets.ine.com/lab/learningpath/e2294c7ea29ef8c61ff45053ef90ffd148c45b36ffbcc9f41ae9f100d608d282.png)

Success! We can now see the decrypted content, which appears to be the output of the "whoami /all" command, indicating that the attacker has administrative-level access on the compromised machine at 10.0.0.156.

Similarly, you can try decrypting other command outputs seen in the capture as well.
In this lab, through systematic packet analysis, we identified malicious executable downloads, identified C2 communication patterns, recognized the C2 framework through magic byte signatures (deadbeef), extracted cryptographic parameters (AES key and IV), and decrypted encrypted communications to reveal the attacker's commands and access level.

Key findings from this analysis include:
1. Confirmed Compromise: The system at 10.0.0.156 was compromised and downloaded checkmate.exe from an internal host at 10.0.0.155.
2. C2 Framework Identification: The traffic was positively identified as Havoc C2 through the characteristic deadbeef magic bytes.
3. Beaconing Pattern: Regular beaconing activity occurred every 2 seconds, indicating an active C2 connection with persistent communication.
4. Encryption Analysis: Successfully extracted the AES key and IV from the initial POST request, enabling decryption of all subsequent C2 communications.
5. Administrative Access: The decrypted command output revealed that the attacker executed whoami /all and gained administrative-level privileges on the compromised system.