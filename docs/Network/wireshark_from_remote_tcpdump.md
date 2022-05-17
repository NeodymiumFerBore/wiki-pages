# Wireshark with remote tcpdump as input

!!! info "Info"
    I haven't tested this with a Linux local machine (e.g, Wireshark on Linux), but it should just work the same
    if you adjust the path in the given command at the end of this document

---

## TL;DR

```sh
user='me'
server='192.168.1.20'
local_ip='192.168.1.10'

ssh -l "$user" "$server" "sudo tcpdump -i any -U -w - 'not (host $local_ip and tcp port 22)'" | /mnt/c/Program\ Files/Wireshark/Wireshark.exe -k -i -
```

---

## The problem

Wireshark and tcpdump are very handy tools to inspect network traffic. Wireshark offers a neat, user-friendly user interface
to analyze network activity. But when it comes to inspect traffic on a headless server, especially in real time, it becomes tricky.

You have several solutions :

- Inspect traffic directly with tcpdump: not user friendly
- Dump your tcpdump capture to a pcap file, transfer it on your local machine, and inspect it offline with Wireshark: not handy at all
- Install Wireshark on the server, and do X Forwarding: the worst solution, not elegant in any way
- Use Wireshark to display the output of remote tcpdump in real time, and optionally save it locally as pcap: this is, in my opinion, the most elegant solution

So, how do you feed Wireshark with a remote tcpdump?

---

## The solution

You just run tcpdump remotely, write raw packets to stdout, and pipe stdout into the local Wireshark. To avoid flooding the capture,
filter out your own ssh connection from the capture.

The solution has some requirements:

On the local machine:

- an ssh client able to execute your Windows programs. I've only tested this with WSL, but it should work with Cygwin. I don't know about the builtin Windows 10 openssh client
- Wireshark must be installed

On the remote machine:

- tcpdump must be installed
- an ssh server must be running
- non-interactive login must be available (use an ssh key, with an agent if needed. sshpass may be used if you don't want to use a key (shame))
- your ssh user must be able to either use tcpdump, or to use sudo without password (you can also use root, but meh)

The key is to be able to run tcpdump remotely without any interaction.

---

## Do it in WSL!

Given that you are running any Linux distro in WSL, just run the following command. Please adjust the variables:

- `user`: your user to connect with
- `server`: the server you want to connect to
- `local_ip`: the IP address of your local machine that you are going to use to connect to the server

```sh
user='me'
server='192.168.1.20'
local_ip='192.168.1.10'

ssh -l "$user" "$server" "sudo tcpdump -i any -U -w - 'not (host $local_ip and tcp port 22)'" | /mnt/c/Program\ Files/Wireshark/Wireshark.exe -k -i -
```

---

## Commands options

`tcpdump` options:

- `-i any`: listen on any interface. You may want to adjust this to your needs
- `-U`: "packet-buffered": each raw packet will be written as soon as it is saved, without waiting for the output buffer to be filled
- `-w -`: write raw packets to `stdout` instead of parsing it and printing result to stdout
- `-U -w -`: the conjunction of those 2 makes tcpdump to write each raw packet to `stdout` as soon as the packet is received
- `'not (host $local_ip and tcp port 22)'`: If the remote interface you are connecting to is included in the listened interface(s),
  you may want to filter your ssh connection out from the capture, or it will flood your output

`Wireshark` options:

- `-k`: start capturing immediately (no prompt when the application starts)
- `-i -`: use stdout as the listening interface (hence, the output of the remote `tcpdump` command)

---

## Stop and save the capture

Clicking the "Stop capturing" button on Wireshark will work as normal, but keep in mind that your `tcpdump` command will still be
running on your remote machine. This may have a performance impact.

The best way to stop capturing is to interrupt your ssh connection (++ctrl+c++). The `tcpdump` command will be killed
and Wireshark will just act as if you clicked on the "Stop capturing" button, because its input interface does not exist anymore.

You can save the capture locally as a pcap file, just like you would do if you were capturing from a local interface. Magic!
