---
layout: post
title:  "Onsite-Procurement"
date:   2015-1-26 23:59:00
categories: Techniques
---

One of my favorite movie trilogies is the Bourne Trilogy. The Bourne Identity, The Bourne Supremacy and The Bourne Ultimatum (I don't count the fourth movie as I find it vastly inferior). It's an action series about Jason Bourne, a former military weapon who has lost his memory and is attempting to find out who he is.  

In each of the three movies, the protagonist has to fight sleeper agents with his bare hands and anything he can find. Typically you would see someone use a gun or a knife in an action movie and while these are used in many parts, Jason also uses tools that might not be so deadly in the hands of the average person. Tools such as a pen, a magazine and a book which are normally harmless are wielded by Bourne to handily defeat his opponents.  

He uses seemingly non-threatening devices to wreak havoc...

Well malicious software can do the exact same thing. We typically focus on the built-in capabilities of malware but oftentimes we overlook its capability to leverage benign tools on a targeted host to assist in the compromise. There are many examples of this activity in the wild but I would like to focus on a couple pieces of malware which are fully utilizing benign system utilities to assist in compromise and prevent remediation.

##Cryptowall

CryptoWall is common variant of ransomware that I have observed being distributed over the past year or so. For those that don't know, it is a type of malware that encrypts files on the infected file system and leaves a "ransom note" with instructions on how to decrypt the files for a fee. This can really put a user or company in a tough spot since the files on the local system and any mapped drives will become unusable. Once the files have been encrypted you are left with very few options. Restoring from backup or paying the ransom are the only options available once this happens. That latter option is not advisable because there is no guarantee you will receive the required decryption keys.

The best chance of recovery for most people and organizations would be to use file backups, if they exist. The authors of CryptoWall realize this but cannot prevent someone from utilizing off-host backup solutions so they go for the next best thing.  

Volume Shadow Copy is a Windows utility that allows users to revert files to a previous state. This has many useful applications such as reverting a saved file that has been erroneously edited and recovering files which have been deleted. The functionality would also give a user the ability to return to a previous copy of the encrypted files.

The authors of CryptoWall put a stop to this by utilizing the same Volume Shadow Copy utility to remove prior versions of the file. The following command is executed as a result of CryptoWall executing on the compromised host:  

`vssadmin.exe Delete Shadows /All /Quiet`  

This is done by injecting the Windows Explorer process and executing the vssadmin process.

![VSSAdmin] (https://lh3.googleusercontent.com/-JZx0K3uxNsA/VpnB25AxSRI/AAAAAAAAATo/_inYzE03gqg/s2048-Ic42/VSSAdmin.png)

This step is critically important if CryptoWall is to be an effective extortion tool. One would reason that this technique would be more effective against the average user in a non-corporate environment as they are less likely to utilize a backup solution. While a user in a corporate environment is more likely to have access to backups, my personal experience has shown that many do not. This obviously pays dividends for those who utilize CryptoWall for the purposes of extortion.  

##Poweliks

Poweliks is another interesting example of malware using benign software for illegitimate purposes. Its claim to fame is that it attempts to avoid detection by hiding a payload in the host's Registry. It is not uncommon for malware to interact with the compromised host's Registry but hiding code there is a creative way of executing a malicious payload. Two objectives are achieved by doing this:  

- Persistence  
- Detection Avoidance

Persistence is achieved when the code is stored as a value at the HKCU\software\microsoft\windows\currentversion\run key location.

![Poweliks-Reg](https://lh3.googleusercontent.com/-O2iqsMleWFE/VplvJ3ezeoI/AAAAAAAAAS4/0BoJSCSv1Nc/s2048-Ic42/RunKey.png)  

Placing the code in this location will cause it to be executed during log in. This is the persistence part I previously mentioned. Let us take a look at how the code is placed there in the first place.  

![Poweliks-rundll32.exe](https://lh3.googleusercontent.com/-qxkru_4mWto/VplvJi9DbkI/AAAAAAAAASs/3NsH-zPIw_E/s2048-Ic42/Poweliks-rundll32-crop2.png)

As we can see, Poweliks.exe executes and launches a number of child processes, most notably being the benign rundll32.exe. Rundll32.exe is typically utilized to directly run a DLL file since they cannot be launched directly. Considering what we know about this process it is very odd to see the following command line activity:

    rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write("<script language=jscript.encode>"+(new ActiveXObject("WScript.Shell")).RegRead("HKCU\\software\\microsoft\windows\\currentversion\\run\\")+"<script>")

It appears that the benign process is not only being utilized to assist in the compromise but also in an unconventional fashion. Instead of launching a DLL, rundll32 is being utilized to execute Javascript code. A Google search for this type of usage revealed a [great explanation](https://stackoverflow.com/questions/25131484/rundll32-exe-javascript). Rundll32 effectively allows the malware to place the above code within the Registry.

We also see Poweliks taking advantage of the popular Windows utility PowerShell. In fact, if the targeted host is lacking a PowerShell installation, Poweliks attempts to download and install it. Evidence of this is provided in the previous screenshot which shows Poweliks spawning the windowsxp-kb968930-x86-eng.exe child process. Microsoft .NET Framework (netfx20sp1_x86.exe) is also installed for the purposes of PowerShell functionality. The amount of effort exerted to install PowerShell indicates it is a necessity for the malware to effectively run so let's see how it is utilized.

![Poweliks-PowerShell](https://lh3.googleusercontent.com/-XT-HND_HzGo/VplvH2gye5I/AAAAAAAAAR4/stNpco3Mkh4/s2048-Ic42/Poweliks-Pshell-cmd.png)

The following command is observed being executed by the PowerShell process:

`powershell.exe iex $env:a`

The malware is using PowerShell to invoke an expression but unfortunately we are unable to see the entirety due to the code value being hidden in the variable "a". Although we are unable to explore what is being executed, it should now be clear that malware and threat actors have no shortage or resources available once a host has been penetrated.

##Conclusion

Just like Bourne can use a book to pummel his opposition, malware can utilize seemingly harmless operating system utilities to wreak havoc on the targeted. In the case of software such as CryptoWall, a security administrator's best defense is to deploy an IDS/IPS with relevant signatures that can prevent the initial phone home request. This will prevent encryption from taking place. The best remediation is to keep backups in a separate location so that they can be used to recover encrypted files.  

What about malware such as Poweliks? There are few avenues of prevention when it comes to mitigating methods such as this. Luckily Poweliks is adware and the consequences are not as dire but more and more malware authors are attempting to employ this technique. It will likely cause issues for end users and security administrators alike in the future.
