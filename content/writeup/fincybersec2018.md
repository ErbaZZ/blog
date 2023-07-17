---
title: "[Writeup] Financial Cybersecurity Bootcamp 2018"
date: 2018-11-11T02:35:49+07:00
categories: [writeup]
tags: [ctf, writeup, competition]
---

In August, I had a chance to participate in an online CTF competition called Financial Cybersecurity Bootcamp 2018, which is a jeopardy style competition with the time limit of 24 hours. Together with my team, !IsCaptured, we solved 4 of the challenges and became eligible to participate in the final round.
<!--more-->

![Finalist Teams](/images/fincybersec2018/fincybersec2018_finalists.jpg)*Finalist Teams Announcement*

The final round was held at the Bank of Thailand Learning Center at 27 October. The style of the competition is different from the first round, as the problems are sequential, such that you need to solve the first challenge in order to continue to the next and so on. The goal is to hack into the system of an imaginary bank called **Cat Bank**.

![!IsCaptured](/images/fincybersec2018/fincybersec2018_team.jpg)*!IsCaptured Team*

There are a total of 5 challenges, the first one provides an NTLM hash of a bank officer with the last two characters missing. It is hinted that we should use Pass the Hash attack on the server using the given hash.

Since there are only two characters missing, and the hash are in hexadecimal, there are only 16^2 or 256 combinations to try, in which we could easily bruteforce it. However, even though we knew bruteforcing it is a logical thing to do, we didn't know how to do the Pass the Hash, or which tools to use. After an hour of trials and errors (and some publicly announced hints from the organizer), we successfully wrote a working Python script that bruteforces the hash and call smbclient of Impacket to connect to the address of the Windows Domain Controller given.

{{< highlight py "linenos=table,linenostart=1" >}}
from subprocess import call

h = "2d3c0049f15843dd77a999a314b192"
charList = "0123456789abcdef"
for i in range (0, len(list)):
    for j in range (0, len(list)):
        fullHash = h + charList[i] + charList[j]
        print(fullHash)
        call(["./smbclient.py", "ccbankad.p7z.pw/bankofficer2@13.251.114.118", "-hashes" , ":"+fullHash])

{{< / highlight >}}

With the script, in less than a minute, we were able to access the domain controller using one of the back officers account.

![Domain Controller Terminal](/images/fincybersec2018/fincybersec2018_1_1.png)*Connected to the Domain Controller*

Since we had zero experience interacting with this type of server, we just run **help** as it suggested to see the list of all available commands.

And of course, we used **ls** to list the current directory, but it responded with "[-] No share selected", so we tried **shares**.

![Trying out the commands](/images/fincybersec2018/fincybersec2018_1_2.png)*Trying out the commands*

We could see 7 different share names, and the most suspicious one is **financial_reports**, so we accessed the share with **use financial_reports** and **ls** again.

![Accessing the share](/images/fincybersec2018/fincybersec2018_1_3.png)*Accessing the share*

Here, we could see an xlsx file, so we downloaded it using **get confidential_2.xlsx** and opened it, revealing the flag and another user account to continue with the next challenge.

![Excel file containing the flag](/images/fincybersec2018/fincybersec2018_1_flag.png)*Excel file containing the flag*

The next challenge tells us that "The system administrator really loves Linux, but his boss has ordered him to use Windows Server. Therefore, he made a hidden way to access his server like Linux".

For this, we started by using nmap to do the default scan, but to no avails, nothing interesting appeared. Therefore, we decided to use nmap to do service scan to scan the whole thing starting from port 1. After a long wait, we finally found something interesting.

![Nmap port scanning result](/images/fincybersec2018/fincybersec2018_2_1.png)*Nmap port scanning result*

We could see that there is an open port with OpenSSH on it, which could be the admin's hidden "Linux way" to access the server. Using the provided credential from the last challenge, we used ssh to connect to the server, and voila, we now have the access to the Windows command prompt.

After looking around for a bit, we found a file called *note.docx* on the desktop of the server. We then used WinSCP to download the file from the server and opened it.

This is what we have found:

![note.docx](/images/fincybersec2018/fincybersec2018_2_2.png)*note.docx*

Yes, an empty Microsoft Word file... or is it?

![Highlighting the content](/images/fincybersec2018/fincybersec2018_2_3.png)*Highlighting the content*

![Flag :)](/images/fincybersec2018/fincybersec2018_2_flag.png)*Flag :)*

And this is the furthest we could go before the end of the competition. Surprisingly, even though most teams passed the second challenge, we were faster than most and won as the Second Runner-up.

![Second Runner-up](/images/fincybersec2018/fincybersec2018_3rdplace.jpg)*!IsCaptured winning Second Runner-up*

This is our first time participating in a on-site CTF competition, and the first time to work on a sequential CTF challenges. It was a fun and memorable experience. I have learned a lot, and there are still much more interesting things for me to learn!