### FreeNAS Setup ###
The purpose of this guide is to set out instructions for:
1. Installing FreeNAS and setting basic configuration required for basic file-sharing through CIFS.
2. Install a BSD Jail to hold Transmission and Flexget
3. Setup configuration for storage and file-sharing of the Jail

#### Chapter 1: Deployment of FreeNAS ####
1. Use Win32 Disk Imager to burn the .iso file of FreeNAS to a USB Memory Stick
2. Boot the machine from that USB and follow on-screen instructions to install FreeNAS on a second memory stick.

> The second memory stick will be the production storage for the operating system. The first memory stick can be repurposed after installation.

1.3 Ensure BIOS settings allow booting from USB as the second memory stick will permanently host the O/S.
	1.4 If you are installing on headless server, connect a screen for the installation part.
	1.5 As soon as installation finishes, configure network interfaces to use DHCP.

2. Access FreeNAS WebGUI to begin configuration.
	2.1. Email configuration:	smtp.gmail.com:587 | TLS
	2.2. Enable 2-step verification on gmail and generate an App Specific password for FreeNAS.
	2.3. Configure Storage:
		2.3.1. Create a Jail specific dataset (say: /mnt/volume1/jail_ds)
		2.3.2. Enable compression at default lz4 | turn off dedup and atime
		2.3.3. Create a dataset which will hold your downloads and data (say: /mnt/volume1/data)
	2.4. Configure Network:
		2.4.1. Set domain, to your preferred domain name and define IPv4 Gateway and DNS
		2.4.2. Under interfaces configure your network card to use DHCP and IPv4
	2.5. Configure Services:
		2.5.1. Turn on CIFS and SSH (Set SSH to allow root login, CIFS will be configured later)
	2.6. Install a new Jail and allocate storage
		2.6.1. Add a new Jail and give it a name. This name cannot be changed afterwards.
		2.6.2. Allocate some storage to the Jail on the dataset created above
		2.6.3. Source: /mnt/volume1/data | Destination: /mnt/volume1/jail_ds/usr/local/etc/transmission/home

3. Jail configuration to run transmission and flexget
Note: pkg install  nano | Use setenv VISUAL /usr/local/bin/nano to replace vi with nano
	3.1. Set up direct SSH access to the Jail
		3.1.1. Edit sshd_enable="NO" to "YES" in etc/rc.conf
		3.1.2. Start the SSHD with service sshd start also (run: "./sshd start" in /etc/rc.d/ )
		3.1.3. Optional: Follow instructions in http://doc.freenas.org/9.3/freenas_jails.html#accessing-a-jail-using-ssh for adding a user.
		3.1.4. Enable root login by editing /etc/ssh/sshd_config and setting '#PermitRootLogin no' to 'PermitRootLogin yes' and restart sshd by running 'service sshd restart'. Use passwd to set a password for root.
		3.1.5. Update pkg before anything:
			pkg clean 						# cleans /var/cache/pkg/
			rm -rf /var/cache/pkg/* 		# just remove it all
			pkg update -f 					# forces update  of repository catalog
			rm /var/db/pkg/repo-*.sqlite 	# removes all remote repository catalogs
			pkg bootstrap -f 				# forces reinstall of pkg
		3.1.6. Install Transmission, Flexget and dependancies
			pkg upgrade -y
			pkg install -y net/py-urllib3 databases/py-sqlite3 devel/py-pip rar unrar nano
			pkg install -y transmission-daemon
			pip install --upgrade pip
			pip install --upgrade setuptools
			pip install flexget
			pip install --upgrade flexget
			pip install transmissionrpc
			pip install --upgrade transmissionrpc
		3.1.7 Start and stop Transmission to create configuration files
				/usr/local/etc/rc.d/transmission onestart | /usr/local/etc/rc.d/transmission stop
		3.1.8. Configure settings.json in /usr/local/etc/transmission/home
		Note: Download location should be /usr/local/etc/transmission/home/Downloads
		3.1.9. Set transmission_enable="YES" in /etc/rc.conf to enable start up on every boot
		3.1.10.Configure config.yml Flexget in /root/.flexget/
		3.1.11.Add flexget to crontab using crontab -u root -e
 and adding '@reboot /usr/local/bin/flexget daemon start -d'
		Note: If for some reason you need to start fresh with Flexget use: 'flexget database reset --sure'
4. Share the completed downloads data from the Jail over the network
	4.1. Create a new 'transmission' user / group in FreeNAS. Set the user and group ID in FreeNAS to be precisely identical to the IDs used in the Jail
	4.2. Create a 'guest' user and add it to the group 'guest'. Disable password login and add auxiliary group the 'transmission' group.
	4.3. Create a new share for the path '/mnt/volume1/data/Downloads', set allow guest access and default permissions.
	4.4. Configure Services>CIFS to use the guest account.
5. House keeping
	5.1. disable password login for transmission user.
	5.2. enable ssh authentication (create key pairs and add to user profile the public key - private key stays out of server)
	5.3. the public key needs to be copied to the jail (remember to conver putty generated key to openssh format)
	5.4. the conversion is best done using VI editor (Ctrl + I, Shift + J)
	Note: https://code.google.com/p/transmisson-remote-gui/ is the best ever interface to manage your Transmission downloader on the server
	Note: This guide is really good https://forums.freenas.org/index.php?threads/seting-up-freenas-9-2-0-with-transmission-and-couchpotato-as-a-dlna-server.17165/
	Note: Use  nano /etc/passwd and nano /etc/group to determine user and group ID
	Note: Use which flexget to determine working directory
	Note: Use killall -HUP transmission-da to force stop transmission when you want to reload settings
	Note: jls, will show a list of all jails | use jexec <#> tcsh to enter by sshd
