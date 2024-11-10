### Brief:

- #### Timeline:


![operation-ghost-timeline](https://github.com/user-attachments/assets/a321f6f9-7c68-4b3b-b842-98c18a79f989)


- #### Malwares:

	 * **PolyglotDuke:** which uses Twitter or other websites such as Reddit and Imgur to get its C&C URL. It also relies on steganography in pictures for its C&C communication.
	 * **RegDuke:** a recovery first stage, which uses Dropbox as its C&C server. The main payload is encrypted on disk and the encryption key is stored in the Windows registry. It also relies on steganography as above. 
	 * **MiniDuke:** backdoor, the second stage. This simple backdoor is written in assembly. It is very similar to older MiniDuke backdoors. 
	 * **FatDuke:** the third stage. This sophisticated backdoor implements a lot of functionalities and has a very flexible configuration. Its code is also well obfuscated using many opaque predicates. They re-compile it and modify the obfuscation frequently to bypass security product detections


---
## 1- The First Stage : 
### Initial Access:

- Using `SpearPhishing` sending a malicious document by mail: 
	-  A `macro-enabled Word document` and get the user to press enable 
	- The macro will install the **PowerDuke** backdoor Or **RegDuke** backdoor. 
	- **PowerDuke** will fetch the c2 url through online service like **Twitter** then download a steghided malware inside a picture that drops **MiniDuke**
	- While **RegDuke** will download the picture directly from **Dropbox** then drops **MiniDuke**
	- **MiniDuke** download a steghided picture from the c2 server then drops **FatDuke


### C2 Server:

* The stager scrape the C2 URL from a posted tweet with the encrypted URL  inside it then it decrypts the URL, It points to a PHP script.
* #### Communication with C2:
	- The compromised computer sends HTTP get request with arguments using the next format :  `GET example.com/name. php?[random_param1]=[random_string1]&[random_param2]=[random_string2]`,   the argument names is hardly encoded and can't be identified 
	- **user-Agent** is a common one = `Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.1; Trident/4.0; GTB7.4; InfoPath.2;SV1; .NET CLR 3.3.69573; WOW64; en-US)` 
	

### Behavior:

#### RegDuke:

* It is a backup payload. Stays hidden on the victim's device for the case of lost connection.
* **Structure and Persistence**:
	 *  **Components**: RegDuke has two main parts—a loader and an encrypted payload stored on the disk. Both are written in .NET.
	 * **Persistence Mechanism**: It uses Windows Management Instrumentation (WMI) with a consumer named `MicrosoftOfficeUpdates`, which activates RegDuke whenever `WINWORD.EXE` (Microsoft Word) starts
* **Loader Behavior**:
     - **Obfuscation Evolution**: The loader has gone through several versions. The earliest versions had hardcoded encryption keys, while later versions stored keys in the Windows registry and used obfuscation methods (e.g., control-flow flattening and .NET Reactor).
	 - **File Decryption**: The loader reads an encrypted file from a hardcoded path or a path specified in the registry. It then decrypts this file using a password and salt either hardcoded or stored in the registry. The decryption uses PBKDF2, a secure password-based key derivation function.
* **Registry Key Usage**:
	- The malware cleverly selects registry keys that might appear legitimate to avoid suspicion, using locations associated with Intel or Microsoft products (e.g., `HKEY_LOCAL_MACHINE\SOFTWARE\Intel\MediaSDK\Dispatch\0102`).


| Registery Key                                                  | Value containing the directory of the payload | Value containing the filename of the payload | Value containing the password and the salt |
| -------------------------------------------------------------- | --------------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| HKEY_LOCAL_MACHINE\SOFTWARE\Intel\ MediaSDK\Dispatch\0102      | PathCPA                                       | CPAmodule                                    | Init                                       |
| HKEY_LOCAL_MACHINE\SOFTWARE\Intel\ MediaSDK\Dispatch\hw64-s1-1 | RootPath                                      | APIModule                                    | Stack                                      |
| HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ MSBuild\4.0             | MSBuildOverrideTasksPath                      | DefaultLibs                                  | BinaryCache                                |

- **Payload Characteristics**:

	- **Fileless and Memory-Resident**: The payload is a backdoor that resides only in memory, avoiding storage on disk. It uses Dropbox as its command-and-control (C&C) server. 
	- **Configuration**: Hardcoded configuration in the payload includes client IDs like `collection_4` and `collection_6`, which are used to manage different targets.
	- **Steganography**: The payload downloads images (PNG files) from Dropbox that look like ordinary pictures. However, data is hidden within the pixels of these images using steganography, allowing the malware to extract hidden information. By using LSB stenography method
----

## 2- The Second Stage : 

1. **MiniDuke Background and Evolution:**
   - **MiniDuke's Purpose:** The MiniDuke backdoor is used as a second-stage malware. It is dropped after the initial stage is executed, typically by another piece of malware. In this case, it's analyzed from a sample compiled in 2018, but the backdoor is known to have been active since at least 2014.
   - **Sample Comparisons:** Researchers observe that the more recent versions share many similarities with earlier ones, such as the SHA-1 hash comparison with older versions described by Kaspersky in 2014.
   - **Recent Changes:** While no significant changes have been found in its core functionality, some modifications have been made, such as a new command and control (C&C) domain and the use of a suspicious or "invalid" digital signature (likely to evade detection by security tools).

2. **Code Complexity and Obfuscation:**
   - **Increased Size:** The backdoor's size increased significantly (from around 20 KB to over 200 KB) due to the implementation of obfuscation techniques.
   - **Obfuscation with Control-Flow Flattening:** One technique used is control-flow flattening, a common obfuscation strategy where the code’s control flow is intentionally made more complex (e.g., using switch/case structures within loops). This makes the code harder to understand or reverse engineer.
   - **Dynamic API Resolution:** The malware obfuscates Windows API calls by dynamically resolving them using a hash function. This further complicates analysis by masking the names of functions it calls.

3. **Network Communication and Evasion Techniques:**
   - **HTTP Communication:** The backdoor uses basic HTTP methods (GET, POST, PUT) to communicate with its hardcoded C&C server. To evade detection in network traffic, the data is prepended with a JPEG image header, making it look like a regular image file. The image itself isn’t valid, but security systems might not inspect the data's validity, letting it blend in with legitimate traffic.
   - **Named Pipe Communication:** The malware can send and receive data through a named pipe. This is especially useful when the compromised machine doesn't have internet access. In such cases, one compromised machine with internet access can act as a "proxy" and forward commands to others on the local network.
   - **HTTP Proxy Feature:** The malware can also listen on a socket (either port 8080 or a specified one) for incoming connections. When a connection is established, it proxies data between this socket and the second socket connected to the C&C server, allowing machines behind firewalls or with no direct internet access to still communicate with the attackers.

4. **Backdoor Capabilities:**
   - **General Functions:** The MiniDuke backdoor implements a variety of functions that allow attackers to control the infected system:
     - Upload or download files.
     - Create and manage processes.
     - Collect system information (e.g., hostname, system ID).
     - Retrieve information about local drives (e.g., their types).
     - Read/write in the named pipe.
     - Manage the proxy feature (start/stop).

5. **Overall Summary:**
   - **Advanced Evasion and Control:** MiniDuke’s use of obfuscation, HTTP-based communication with evasion tactics (JPEG header), and the named pipe functionality shows a sophisticated approach to maintaining persistence and controlling infected systems. The addition of proxying features ensures the malware can spread and communicate even on isolated or firewall-protected systems.
   - **Persistence in the Threat Landscape:** Despite being a relatively older malware, the persistence of MiniDuke and its ongoing adaptations (such as the digital signature trick) suggest that it remains a tool in the arsenal of its operators, adapting to new detection mechanisms.

---

## 3 - Third Stage : 

# **Stopped at page 24 **




