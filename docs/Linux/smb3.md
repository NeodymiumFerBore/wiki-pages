# SMB3 using Samba 4.1+ , for client and server

Since October 2013, samba 4.1.0 implements SMB3 protocol. Debian 9.12 ships version 4.5.16 from official repository. An up-to-date recent distro should be able to get samba4 from its official repositories. SMB 3.1.1 is supported since Samba 4.3.

## Note: break the confusion

- [`LinuxCIFS`](https://wiki.samba.org/index.php/LinuxCIFS): "The CIFS VFS is a virtual file system for Linux to allow access to modern SMB3 servers (...) as well as older servers and storage appliances compliant with the SNIA CIFS Specification version 1.0 or later"
- [`cifs-utils`](https://wiki.samba.org/index.php/LinuxCIFS_utils): "The in-kernel CIFS filesystem relies on a set of user-space tools. That package of tools is called cifs-utils. Although not really part of Samba proper, these tools were originally part of the Samba package"
- [`Samba`](https://www.samba.org/samba/what_is_samba.html): "Samba is the standard Windows interoperability suite of programs for Linux and Unix". It is more used for server side operations. More details about capabilities [here](https://wiki.samba.org/index.php/Main_Page)
- [`smbclient`](https://manpages.debian.org/buster/smbclient/smbclient.1.en.html): "ftp-like client to access SMB/CIFS resources on servers. (...) It offers an interface similar to that of the ftp program"
- [`smbfs`](https://www.samba.org/samba/smbfs/): "The smbfs filesystem is a mountable SMB filesystem for Linux. (...) **smbfs has not been maintained in the last few years**"

## Samba configuration

Note that some SMB clients like `smbclient` will parse `/etc/smb.conf`. `cifs-utils` client WILL NOT (as it does not require samba to be installed [anymore](https://wiki.samba.org/index.php/LinuxCIFS_utils#Description)). You have to specify the version with `-o vers=VERS` (version references: [mount.cifs(8)](https://manpages.debian.org/buster/cifs-utils/mount.cifs.8.en.html), look for "vers="). Today's Linux kernel cifs client [defaults to SMB3](https://wiki.samba.org/index.php/LinuxCIFSKernel#Linux_Kernel_SMB3.2FCIFS_Client)

The variable names are pretty clear. You can also set it on a share basis, to maintain different policies. `protocol` can be use alone, but will only set max protocol, not min.

> /etc/samba/smb.conf
```ini
[global]
  ; client
  client min protocol = SMB2_10
  ; default was 'CORE' (MS-DOS era) not too long ago
  
  client max protocol = SMB3
  ; default was 'NT1' (aka CIFS) until Samba v4.6
  ; finally includes SMB3 as of Samba v4.7

  ; server
  min protocol = SMB2_10
  max protocol = SMB3

[homes]
  (...)
  min protocol = SMB3
```

---

## Understanding min and max protocol levels in smb.conf

Note that `SMB2` maps to `SMB2_10` and `SMB3` maps to `SMB3_11`.

=== "client min/max protocol"

    - `client min protocol`: minimum protocol version that the client will attempt to use.
    - `client max protocol`: highest protocol version that will be supported by the client.

=== "SMB2"

    - `SMB2`: Re-implementation of the SMB protocol. Used by Windows Vista and later versions of Windows. SMB2 has sub protocols available:
        - `SMB2_02`: The earliest SMB2 version.
        - `SMB2_10`: Windows 7 SMB2 version. (By default SMB2 selects the SMB2_10 variant.)
        - `SMB2_22`: Early Windows 8 SMB2 version.
        - `SMB2_24`: Windows 8 beta SMB2 version.

=== "SMB3"

    - `SMB3`: Used by Windows 8 and later. SMB3 has sub protocols available:
        - `SMB3_00`: Windows 8 SMB3 version. (mostly the same as SMB2_24)
        - `SMB3_02`: Windows 8.1 SMB3 version.
        - `SMB3_10`: early Windows 10 technical preview SMB3 version.
        - `SMB3_11`: Windows 10 technical preview SMB3 version (maybe final). By default SMB3 selects the SMB3_11 variant.

---

## Sources

- [Default values info](https://superuser.com/questions/1226973/how-to-force-linux-cifs-mount-to-default-to-smb3)
- [Settings details](https://www.cyberciti.biz/faq/how-to-configure-samba-to-use-smbv2-and-disable-smbv1-on-linux-or-unix)
- [smb.conf](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#CLIENTMAXPROTOCOL)
- [mount.cifs(8): available dialects and info about not parsing smb.conf](https://manpages.debian.org/buster/cifs-utils/mount.cifs.8.en.html)
- [Samba releases SMB3 support for client tools](https://www.samba.org/samba/history/samba-4.1.0.html)

## See also: a bit of history

- [CIFS vs SMB eplained](https://www.varonis.com/blog/cifs-vs-smb/)
- [Why You Should Never Again Utter The Word, “CIFS”](https://blog.fosketts.net/2012/02/16/cifs-smb/)
- [Nice Samba history (pdf)](https://events.static.linuxfound.org/sites/events/files/slides/smb3-in-samba.pr__0.pdf)

Today's implementations (thanks god no more cifs):

- [A bit of technical history](https://wiki.robotz.com/index.php/Linux_CIFS_Utils_and_Samba#What_are_the_differences_between_CIFS_and_SAMBA.3F)
- [See "Linux kernel breaks CIFS"](https://wiki.robotz.com/index.php/Linux_CIFS_Utils_and_Samba#fstab_persistent_mount)
- [Linux kernel SMB3/CIFS client now defaults to SMB3.11](https://wiki.samba.org/index.php/LinuxCIFSKernel#Linux_Kernel_SMB3.2FCIFS_Client)

So, quick reminder: you are actually using CIFS if you are setting your cifs-utils/samba version to 1.0/NT1 (Windows NT era). Which you should never do :) 
