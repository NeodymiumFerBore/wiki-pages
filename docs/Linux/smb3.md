# SMB 3.1.1 using Samba 4.1+ , for client and server

Since October 2013, samba 4.1.0 implements SMB3 protocol. Debian 9.12 ships version 4.5.16 from official repository. An up-to-date recent distro should be able to get samba4 from its official repositories.

## Configuration

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

Note that the `SMB2` and `SMB3` keywords map to the highest supported sub-version of that protocol. This means that setting the minimum protocol to `SMB3` would actually exclude Windows 8.1.

=== "client min/max protocol"

  - `client min protocol`: This setting controls the minimum protocol version that the client will attempt to use.
  - `client max protocol`: The value of the parameter (a string) is the highest protocol level that will be supported by the client.

=== "SMB2"

    - `SMB2`: Re-implementation of the SMB protocol. Used by Windows Vista and later versions of Windows. SMB2 has sub protocols available:
        - `SMB2_02`: The earliest SMB2 version.
        - `SMB2_10`: Windows 7 SMB2 version. (By default SMB2 selects the SMB2\_10 variant.)
        - `SMB2_22`: Early Windows 8 SMB2 version.
        - `SMB2_24`: Windows 8 beta SMB2 version.

=== "SMB3"

    - `SMB3`: The same as SMB2. Used by Windows 8. SMB3 has sub protocols available:
        - `SMB3_00`: Windows 8 SMB3 version. (mostly the same as SMB2\_24)
        - `SMB3_02`: Windows 8.1 SMB3 version.
        - `SMB3_10`: early Windows 10 technical preview SMB3 version.
        - `SMB3_11`: Windows 10 technical preview SMB3 version (maybe final). By default SMB3 selects the SMB3\_11 variant.

---

## Sources

- [Default values info](https://superuser.com/questions/1226973/how-to-force-linux-cifs-mount-to-default-to-smb3)
- [Settings details](https://www.cyberciti.biz/faq/how-to-configure-samba-to-use-smbv2-and-disable-smbv1-on-linux-or-unix)
