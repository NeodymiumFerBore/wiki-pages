# Difference between RUID and EUID

## What's the difference?

RUID is the *Real User ID* and it never (almost) changes. If `user1` logs in to the system, the shell is then launched with its real ID set to `user1`. All processes they start from the shell will inherit the real ID `user1` as their real ID.

EUID is the *Effective User ID*. When a user executes a file that have the *setuid* bit set, the EUID changes for the created process.

If `user1` executes `file.bin` with bit set 4755 (`chmod +s`), the RUID will still be `user1` and the EUID of the started process will be the uid of the *file owner*.

Let's use the case of `passwd`:

```
-rwsr-xr-x 1 root root 45396 may 25  2012 /usr/bin/passwd
```

- When `user1` wants to **change their password**, they execute `/usr/bin/passwd`.
- The RUID will be `user1` but the EUID of that process will be `root`.
- `user1` can use `passwd` to change **only** their own password because internally `passwd` checks the RUID and, if it is not `root`, its actions will be limited to real user's password.
- It's neccesary that the EUID becomes `root` in the case of `passwd` because the process needs to **write** to `/etc/passwd` and/or `/etc/shadow`.

---

## Example for spawning child process as EUID

Considering the following executable `reuid-poc` with setuid bit, in a session authenticated as `user1` with UID `1000`:

```console
user1@host:/tmp$ id -u user1
1000
user1@host:/tmp$ id -u root
0
user1@host:/tmp$ ls -l
-rwsr-xr-x 1 root root 12762 Jun 29 06:32 reuid-poc*
-rw-r----- 1 root root 12762 Jun 29 06:32 reuid-poc.c
```

As seen previously, when executed, the RUID of the process will be `1000`, but the EUID will be `0`. The actions done by `./reuid-poc` will be done as `root` (`0`).

However, a child process will be spawned as `1000`. The EUID of a child process will be the RUID of the caller process. Therefore, if one wants to spawn a child process in a setuid program with the privilege of the owner of this program (and not the caller), then he has to manually set the RUID of the current process beforehand.

Here is the content of `reuid-poc.c`:

??? abstract "Click to reveal reuid-poc.c"

    ```c
    #include <stdio.h>
    #include <stdlib.h> // for system()
    #include <unistd.h> // for setuid(), getuid(), uid_t

    /*
    gcc -o reuid-poc reuid-poc.c
    */

    int main(int argc, char **argv)
    {
        char child[]="sh -c /usr/bin/whoami";

        printf("RUID: %d\n",   getuid()); // getuid() returns RUID (RUID == UID)
        printf("EUID: %d\n\n", geteuid());

        printf("whoami: ");
        fflush(stdout);
        system(child);

        printf("--------------------------------------\n");

        if (setuid(0) == 0)
            printf("setuid success\n");
        else
            printf("setuid failed\n");

        printf("--------------------------------------\n");

        printf("RUID: %d\n",   getuid());
        printf("EUID: %d\n\n", geteuid());

        printf("whoami: ");
        fflush(stdout);
        system(child);

        return 0;
    }
    ```

Now we compile our code as root, set the setuid bit permission and allow execution for all users:

```console
root@host:/tmp# gcc -o reuid-poc reuid-poc.c
root@host:/tmp# chown 0:0 reuid-poc
root@host:/tmp# chmod 4755 reuid-poc
```

When we go back to our user `user1`, we execute this program:

```console
user1@host:/tmp$ ./reuid-poc
RUID: 1000
EUID: 0

whoami: user1
--------------------------------------
setuid success
--------------------------------------
RUID: 0
EUID: 0

whoami: root
```

We see that before the `setuid(1000)` call, the RUID of the process is the UID of our user, `1000`. The child process' EUID inherits the RUID of its parent, `1000`. After the `setuid` call, the RUID becomes the one of the owner of the file (here, `root`).

Note that a function `setreuid(uid_t ruid, uid_t euid)` also exist to set both RUID and EUID of the current process (for impersonation matters).

Same mechanism exists for GID (RGID and EGID in the context of setgid bit permission, and others...)

---

## Sources

- [Difference between owner/root and ruid/euid](https://unix.stackexchange.com/questions/191940/difference-between-owner-root-and-ruid-euid)
- [manpage of setreuid](http://manpagesfr.free.fr/man/man2/setreuid.2.html)
