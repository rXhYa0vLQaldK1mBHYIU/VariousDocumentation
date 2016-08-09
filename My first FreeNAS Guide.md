### FreeNAS Setup ###

The purpose of this guide is to set out instructions for:
1.  Installing FreeNAS and setting basic configuration required for
    basic file-sharing through CIFS.
2.  Install a BSD Jail to hold Transmission and Flexget.
3.  Setup configuration for storage and file-sharing of the Jail.

#### Chapter 1: Deployment of FreeNAS and BSD Jail ####

1.  Use Win32 Disk Imager to burn the .iso file of FreeNAS to a USB
    Memory Stick
2.  Boot the machine from that USB and follow on-screen instructions to
    install FreeNAS on a second memory stick.
>The second memory stick will be the production storage and will host the actual operating system. The first memory stick can be repurposed after installation.
>
>Ensure BIOS settings allow booting from USB;
>
>If you are installing on headless server, connect a screen for the installation part. I hove not yet found a way to bypass this requirement.

3.  Follow on-screen instructions to complete the installation.

    > For a home-setup consider whether it may be preferable to configure network interfaces to use DHCP.

4.  Access FreeNAS WebGUI to begin configuration.
5.  To enable gmail integration, configure smtp.gmail.com:587 to use TLS
    and if 2-step verification is enabled on gmail (recommended)
    generate an Application-Specific password for explicit use
    by FreeNAS.
6.  Configure Storage as follows:
  * Create a Jail specific dataset (say: ``/mnt/volume1/my-jail-dataset``);
  * Configure compression, deduplication and atime;
  * Create a dataset which will hold your downloads and data (say: ``/mnt/volume1/my-main-dataset``).

7. Check network configuration as follows:
 * Ensure domain name, IPv4 gateway and DNS are configured correctly;
 * Under interfaces, the network card should be configured to use DHCP and IPv4.

8. Configure services as follows:  
 * Turn on CIFS and SSH;
 * Set SSH to allow root login (CIFS will be configured later).
 * Create a new Jail and allocate the destination storage (i.e. the location within the Jail) whereas the source of the storage was created above.

    > The name given to the Jail cannot be changed afterwards. Example locations as used in the above examples:  
    > Source: ``/mnt/volume1/my-main-dataset``  
    > Destination: ``/mnt/volume1/my-jail-dataset/usr/local/etc/transmission/home``

#### Chapter 2: Transmission and Flexget configuration ####
FreeNAS plugins natively support Transmission but in this workcase we will be creating a Jail to install both Transmission and Flexget directly from the repositories.

1.  Prepare the Jail
>Note: use ``pkg install nano`` and use ``setenv VISUAL /usr/local/bin/nano`` to replace ``vi`` with ``nano``.  

  1. Set up direct SSH access to the Jail by editing ``sshd_enable="NO"`` to ``"YES"`` in ``/etc/rc.conf``. Start the SSHD with ``service sshd start``. Note: also run ``./sshd start`` in ``/etc/rc.d/``. Follow instructions in [freenas documentation](http://doc.freenas.org/9.3/freenas_jails.html#accessing-a-jail-using-ssh "instructions to accessing-a-jail-using-ssh") for more details.

  2. Enable root login by editing ``/etc/ssh/sshd_config`` and setting ``#PermitRootLogin no`` to ``PermitRootLogin yes`` and restart ``sshd`` by ``service sshd restart``.

  3. Use ``passwd`` to set a password for root user.

  4. Update ``pkg`` before anything as follows:  
     * run ``pkg clean`` to clean ``/var/cache/pkg/``. Use ``rm -rf /var/cache/pkg/*`` to just remove it all.
     * then use ``pkg update -f`` to force an update of the repository catalog ``rm /var/db/pkg/repo-\*.sqlite`` will remove all remote repository catalogs.
     * to force reinstall of ``pkg`` use ``pkg bootstrap -f ``
     * ``pkg upgrade -y``

2. Install **Transmission** and **Flexget**

 * Install dependencies ``pkg install -y net/py-urllib3 databases/py-sqlite3 devel/py-pip rar unrar``
 > Try without above steps first as my guide may need some updating to remove redundant steps.
 * Install **transmission** with ``pkg install -y transmission-daemon``
 * Upgrade pip with ``pip install --upgrade pip``
 * Upgrade setuptools with  ``pip install --upgrade setuptools``
 * Install **flexget** with ``pip install flexget``
 > To ensure latest version of **flexget** is installed use ``pip install --upgrade flexget``
 * Install the rpc element of **transmission** with ``pip install transmissionrpc``
 > To ensure latest version of **transmission** use ``pip install --upgrade transmissionrpc``
 * Start and stop **transmission** to create configuration files. Start with ``/usr/local/etc/rc.d/transmission onestart`` and stop with ``/usr/local/etc/rc.d/transmission stop``

3. Configure **transmission** parameters and locations
 * First look into ``settings.json`` found in ``/usr/local/etc/transmission/home``
    > Download location should be something like ``/usr/local/etc/transmission/home/Downloads``
 * Set ``transmission_enable="YES"`` in ``/etc/rc.conf`` to enable start up on every boot
 * Configure other settings as needed.

4. Configure **flexget** parameters.
 * Use the **flexget cookbook** to define ``config.yml`` in ``/root/.flexget/``
 > The flexget cookbook is a collection of scripts which when used together form a recipe. The config file contains these recipes.
 * Add **flexget** to **crontab** using ``crontab -u root -e`` and add ``<'@reboot> /usr/local/bin/flexget daemon start -d'``
> To start fresh with Flexget use: ``flexget database
    reset --sure``

5. Share the completed downloads data from the Jail over the network and set user permissions.
    * Create a new ``transmission`` user / group in FreeNAS and disable password login.
    * Set the user and group ID in FreeNAS to be precisely identical to the IDs used within the Jail.
    > Use ``nano /etc/passwd`` and ``nano /etc/group`` to determine user and group ID.
    * Create a ``guest`` user (if it does not already exist) and add it to the group ``guest``. Disable password login and add as an _auxiliary group_ the ``transmission`` group.
    * Create a new share for the path ``/mnt/volume1/data/Downloads`` (or similar) and set to _allow guest access_ with the default permissions.
    * Configure in services the CIFS shares to use the guest account.

6. Housekeeping (optional steps)
 * Enable ssh authentication to both the **FreeNAS** instance and the **Jail**. Key pairs will need to be created and the public key added to the user profile (root in this case). The private key stays out of server and will be only needed by the client that will connect remotely.
 > The public key needs to be copied to the jail (remember that if you will be using **PuTTY** to generate keys, the public key needs to be copied from the dialogue (as opposed to being saved).
 * Get **Transmission** remote GUI from [here](https://code.google.com/p/transmisson-remote-gui/ 'the best remote for transmission').
 * This [guide](https://forums.freenas.org/index.php?threads/seting-up-freenas-9-2-0-with-transmission-and-couchpotato-as-a-dlna-server.17165/'setting up freenas with transmission') is really good.
 * Use ``which flexget`` to determine working directory
 * Use ``killall -HUP transmission-daemon`` to force stop transmission when you want to reload settings
 * Use ``jls`` to get a list of all jails and use ``jexec`` to access the jail through the FreeNAS instance by ``sshd``.
