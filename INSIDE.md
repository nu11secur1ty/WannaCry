![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/0564d4ec4885448fc55b26b11c1ae743.png)
# The worm that spreads WanaCrypt0r

by Zammis Clark

Something that many security researchers have feared has indeed come true. Threat actors have integrated a critical exploit taking advantage of a popular communication protocol used by Windows systems, crippling thousands of computers worldwide with ransomware.

Within hours of being leveraged, a flaw that had been recently patched by Microsoft has been used to distribute the WanaCrypt0r ransomware and wreak havoc worldwide.

In this blog post, we will describe the worm responsible for spreading this ransomware by looking at its capabilities and what has made this threat so successful.
Main functionality

WanaCrypt0r has been most effective—not only does the ransomware loop through every open RDP session on a system and run the ransomware as that user, but the initial component that gets dropped on systems appears to be a worm that contains and runs the ransomware, spreading itself using the ETERNALBLUE SMB vulnerability (MS17-010).

The WinMain of this executable first tries to connect to the website www.iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com. It doesn’t actually download anything there, just tries to connect. If the connection succeeds, the binary exits.

This was probably some kind of kill switch or anti-sandbox technique. Whichever it is, it has backfired on the authors of the worm, as the domain has been sinkholed and the host in question now resolves to an IP address that hosts a website. Therefore, nothing will happen on any new systems that runs the executable. This only applies to the binary with the hash listed above; there may well be new versions released in the future. UPDATE: The second argument to InternetOpenA is 1 (INTERNET_OPEN_TYPE_DIRECT), so the worm will still work on any system that requires a proxy to access the Internet, which is the case on the majority of corporate networks. Thanks to Didier Stevens for spotting what was missed by most.

![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_winmain.png)


After this check passes, the first thing the worm does is check the number of arguments it was launched with. If it was run with less than two arguments passed, it installs a service called mssecsvc2.0 with display name Microsoft Security Center (2.0) Service (where the binary ran is itself with two arguments), starts that service, drops the ransomware binary located in the resources of the worm, and runs it.

If it was run with two arguments or more—in other words, if it was run as a service—execution eventually falls through to the worm function.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_the_worm_function.png)


The initialization function called first calls WSAStartup() to initialize networking, then CryptAcquireContext() to initialize the crypto API so it can use a cryptographically-secure pseudo-random number generator. It then calls a function that initializes two buffers used for storing the worm payload DLLs, one x86 and one x64. It copies the payload DLLs from the .data section of the worm and then copies the entire worm binary after it.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_payload_init.png)


The code of each payload DLL is very small, just getting the resource content (i.e. the worm binary), dropping to disk as C:\WINDOWS\mssecsvc.exe (this path is actually hardcoded) and executing it.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_payload_code.png)


# SMB vulnerability leveraged to spread ransomware worldwide

After initializing the functionality used by the worm, two threads are created. The first thread scans hosts on the LAN. The second thread gets created 128 times and scans hosts on the wider Internet.

The first thread (in charge of scanning LAN) uses GetAdaptersInfo() to get a list of IP ranges on the local network, then creates an array of every IP in those ranges to scan.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_getadaptorinfo.png)


The LAN scanning is multithreaded itself, and there is code to prevent scanning more than 10 IP addresses on the LAN at a time.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_scan_lan.png)


The scanning thread tries to connect to port 445, and if so creates a new thread to try to exploit the system using MS17-010/EternalBlue. If the exploitation attempts take over 10 minutes, then the exploitation thread is stopped.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_scan_lan_thread.png)


The threads that scan the Internet generate a random IP address, using either the OS’s cryptographically secure pseudo-random number generator initialized earlier, or a weaker pseudo-random number generator if the CSPRNG failed to initialize. If connection to port 445 on that random IP address succeeds, the entire /24 range is scanned, and if port 445 is open, exploit attempts are made. This time, exploitation timeout for each IP happens not after 10 minutes but after one hour.


![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_scan_inet_part1.png)
![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_scan_inet_part2.png)

The exploitation thread tries several times to exploit, with two different sets of buffers used (perhaps one for x86 and one for x64). If it detects the presence of DOUBLEPULSAR after any exploitation attempt, it uses DOUBLEPULSAR to load the relevant payload DLL.

![](https://github.com/nu11secur1ty/-WannaCry-/blob/master/inside/worm_exploitation_thread.png)


Protection

It is critical that you install all available OS updates to prevent getting exploited by the MS17-010 vulnerability. Any systems running a Windows version that did not receive a patch for this vulnerability should be removed from all networks. If your systems have been affected, DOUBLEPULSAR will have also been installed, so this will need to also be removed. A script is available that can remotely detect and remove the DOUBLEPULSAR backdoor. Consumer and business customers of Malwarebytes are protected from this ransomware by the premium version of Malwarebytes and Malwarebytes Endpoint Security, respectively.


# SHARE THIS ARTICLE
# Have fun with nu11secur1ty =)
























