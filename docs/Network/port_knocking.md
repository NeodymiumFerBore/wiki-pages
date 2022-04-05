# Port Knocking

!!! warning "Disclaimer"
    Port knocking is NOT a security measure. Port knocking only offers an obfuscation layer that prevents port scanners to detect your service. Services available behind this mechanism should be maintained and hardened as if they were directly exposed to the internet.

    I ellegibly decline all responsibility for what could happen to you, your services, your hardware, your data, your privacy or your integrity if you decide to setup this mechanism on your own facilities.

[Direct link to the configuration example](./#example-of-implementation-100-based-on-iptables)

---

## Introduction

Sometimes, you want to expose a service to the internet, from within your private network. But you don't want your service to appear on a network scan result.

To do so, you have several solutions. Here is a mechanism that allows you to expose many services on the internet, and remain invisible to most of network scans: port knocking.

---

## What is Port Knocking?

The principle of port knocking is quite simple: a client has to successfully attempt a connection to a series of closed ports, in the correct order, before being able to initiate a connection to a real service on a different port. If they fail to "knock" at the correct ports, then they must do it again from the beggining.

---

## Port Knocking mechanism with iptables

Here is a more detailed way of what happens during the port knocking process. Note that there are some omitted details for clarity purposes (such as untagging sources IP before submitting it to a check rule). Complete detail is available in the [configuration example](./#example-of-implementation-100-based-on-iptables).

- You want to expose an SSH service on the internet. Your server is listening on port `22/tcp`
- You select `n` dummy ports. For example, let's say you choose ports `1234`, `2345` and `3456` (in this order). No service should run behind these ports
- On your ISP router ("box"), you forward those 4 ports to your server (with PAT rules)
- You create iptables chains and rules to implement the following logic:
    1. If a TCP packet is received and does not match any rule, `DROP` it
    2. If a TCP packet is received on port `1234/tcp`, then the source IP is tagged `OK1`. Packet is dropped
    3. If the same source IP sends another packet on port `2345/tcp`, and the IP is tagged `OK1` , then the source IP is tagged `OK2`. Packet is dropped
    4. If the same source IP sends another packet on port `3456/tcp`, and the IP is tagged `OK2`, then the source IP is tagged `OK3`. Packet is dropped
    5. If the same source IP initiates a connection to port `22/tcp`, and the IP is tagged `OK3`, then the connection is allowed. Else, packet is dropped
    6. From point **b** to point **e**, if the source IP sends a packet to an unexpected port, remove any `OK` flag from it, so it must go back to point **b**.

With this logic, you can allow any port under the condition that the source IP is tagged `OK3`. This permits you to expose services on the internet only to clients that know your exact port knocking chain. Your chain can be as long as you want.

**Be aware though**: systems do send retransmission packets (to overcome packet losses). As a client, it can screw up your port knocking. If you are at the third port in the chain, and your system decides to send a retransmission packet to the first port, then your IP will be untagged and forgotten by your firewall. Meaning that your connection to the service will fail, forcing you to re-perform the port knocking.

The `DROP` rule can be replaced with `REJECT`. Just use anything that would look like another closed/filtered port on your router, so the first port of the knocking chain does not look suspicious.

---

## How to perform port knocking and open an SSH connection?

You can make a simple script that perform the knocking for you, and connect to the ssh target. This script just serves as a simple example. You can customize it to make tunnels, connect to other services, accept arguments,... Make it as sexy as you want!

```sh
#!/bin/bash

# Replace this with your server. It can be a public IP if you made
# the correct PAT rules on your ISP router
your_server='192.168.1.123'
your_user='user'

for i in 1234 2345 3456; do
  ncat -w 0.1 "$your_server" "$i"
done
ssh -l "$your_user" "$your_server"
```

- `for i in 1234 2345 3456`: loop over your ports, in the correct order
- `ncat`: open TCP connection
  - `-w 0.1`: set a timeout of 0.1 second, so ncat instantly stops trying to connect. This is useful to be fast enough so the knocking process finishes before any packet retransmission happens
  - `"$your_server"`: **¯\\\_(ツ)\_/¯**
  - `"$i"`: the port to knock on
- `ssh (...)`: connect to your service

A one-liner could also be:

```
for i in 1234 2345 3456; do ncat -w 0.1 192.168.1.123 $i; done; ssh -p 22 user@192.168.1.123
```

---

## Example of implementation, 100% based on iptables

Here is an extract of a working, minimal iptables port knocking configuration (`iptables-save -t filter`). Everything is configured in the `filter` table. I explain the process details in the comments. Sorry if there are too much of these!

The following configuration re-use our previous example: knock chain is `1234/tcp`, `2345/tcp`, `3456/tcp`, and the service is `22/tcp` on the same host.

{# How to reference mkdocs configuration variable inside a page with Jinja2: https://timvink.github.io/mkdocs-print-site-plugin/how-to/cover_page.html #}

If you find any error in this configuration, please open an issue on [my GitHub]({{ config.repo_url }})!

??? abstract "Click to reveal the iptables port kocking configuration with comments"

    ```sh
    *filter
    # Default chains
    :INPUT DROP [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]

    # Port knocking chains
    :KNOCKING - [0:0]
    :GATE1 - [0:0]
    :GATE2 - [0:0]
    :GATE3 - [0:0]
    :SUCCESS - [0:0]

    # ------------------------ INPUT ------------------------

    # Always accept 127.0.0.1 on loopback iface
    -A INPUT -s 127.0.0.1/32 -d 127.0.0.1/32 -i lo -j ACCEPT

    # Bonus: Drop XMAS & NULL scans
    -A INPUT -p tcp -m tcp --tcp-flags FIN,PSH,URG FIN,PSH,URG -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN,RST,PSH,ACK,URG -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags SYN,RST SYN,RST -j DROP

    # Be stateful. This is required.
    -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

    #------------------------------------------------

    # Here would be your INPUT rules that don't require port knocking,
    # for local traffic for example

    #------------------------------------------------

    # Send anything unmatched to the KNOCKING chain. This is the entrypoint of your port knocking
    -A INPUT -j KNOCKING

    #------------------------ KNOCKING ------------------------

    # Here beggins the interesting stuff. KNOCKING will send the traffic
    # to the correct chain, regarding the tag the source IP is carrying

    # First, check if the source IP has the tag 'OK3' since less than 10s ago
    # If it does, then send the packet to the chain SUCCESS
    -A KNOCKING -m recent --rcheck --seconds 10 --name OK3 --mask 255.255.255.255 --rsource -j SUCCESS

    # If previous check failed, then check if it has 'OK2' since less than 3s ago
    # If it does, then send the packet to GATE3
    -A KNOCKING -m recent --rcheck --seconds 3 --name OK2 --mask 255.255.255.255 --rsource -j GATE3

    # If previous check failed, then check if it has 'OK1' since less than 3s ago
    # If it does, then send the packet to GATE2
    -A KNOCKING -m recent --rcheck --seconds 3 --name OK1 --mask 255.255.255.255 --rsource -j GATE2

    # If previous check failed, then it's the first time we see the source IP.
    # Send it to GATE1
    -A KNOCKING -j GATE1

    #------------------------ GATE1 ------------------------

    # GATE1 will be a real traffic trash. Do not log connection attempts,
    # or internet scanners will flood your logs

    # If proto is TCP, and destination port is 1234, then set the name OK1 on the remote source
    # and drop the packet
    -A GATE1 -p tcp -m tcp --dport 1234 -m recent --set --name OK1 --mask 255.255.255.255 --rsource -j DROP
    # If the packet does not match the previous rule, then drop it
    -A GATE1 -j DROP

    #------------------------ GATE2 ------------------------

    # GATE2 is less likely to be reached. You can log attempts here
    # If you receive a packet in this chain, the the KNOCKING chain already checked
    # that the tag OK1 was present.

    # Remove the tag OK1 from the remote source. It is important to do so. If you don't,
    # then a port scanner would have 3 seconds to trigger the next rule. It also serves to
    # legitimately reset the client tags
    -A GATE2 -m recent --remove --name OK1 --mask 255.255.255.255 --rsource

    # Log the knocking attempt
    -A GATE2 -j LOG --log-prefix "In knock 2: " --log-level 6

    # If proto is TCP, and destination port is 2345, then set the name OK2 on the remote source
    # and drop the packet
    -A GATE2 -p tcp -m tcp --dport 2345 -m recent --set --name OK2 --mask 255.255.255.255 --rsource -j DROP

    # If previous rule didn't match, then send the packet to GATE1
    # (which will ensure to DROP the packet if the port does not match your first rule)
    -A GATE2 -j GATE1

    #------------------------ GATE3 ------------------------

    # GATE3 is pretty much the same than GATE2. Now that you know how it works,
    # you can add as much "GATE" chains as you want, and adapt the KNOCKING chain accordingly

    # Remove the tag OK2 from the remote source (important to reset the client)
    -A GATE3 -m recent --remove --name OK2 --mask 255.255.255.255 --rsource

    # Log attempt (it becomes more and more interesting to log)
    -A GATE3 -j LOG --log-prefix "In knock 3: " --log-level 6

    # If proto is TCP and destination port is 3456, then set the name OK3 on the remote source
    # and drop the packet
    -A GATE3 -p tcp -m tcp --dport 3456 -m recent --set --name OK3 --mask 255.255.255.255 --rsource -j DROP

    # If previous rule didn't match, then send the packet to GATE1
    -A GATE3 -j GATE1

    #------------------------ SUCCESS ------------------------

    # SUCCESS is the chain that will be reached by any client who successfuly knocked to all
    # ports in the correct order. Here you can add finer control, send traffic to FORWARD, etc.
    # In this example, it just opens port 22 on the same host

    # Remove the tag OK3 from the remote source
    -A SUCCESS -m recent --remove --name OK3 --mask 255.255.255.255 --rsource

    # Log the knock success
    -A SUCCESS -j LOG --log-prefix "Knock accepted!: " --log-level 6

    # ACCEPT the connection on 22/tcp
    -A SUCCESS -p tcp -m tcp --dport 22 -j ACCEPT

    # Send any unmatched packet to GATE1
    -A SUCCESS -j GATE1

    #------------------------------------------------

    # Commit the iptables config (for iptables-restore)
    COMMIT
    ```

??? abstract "Click to reveal the iptables port kocking configuration without comments"

    ```sh
    *filter
    :INPUT DROP [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    :KNOCKING - [0:0]
    :GATE1 - [0:0]
    :GATE2 - [0:0]
    :GATE3 - [0:0]
    :SUCCESS - [0:0]
    -A INPUT -s 127.0.0.1/32 -d 127.0.0.1/32 -i lo -j ACCEPT
    -A INPUT -p tcp -m tcp --tcp-flags FIN,PSH,URG FIN,PSH,URG -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN,RST,PSH,ACK,URG -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
    -A INPUT -p tcp -m tcp --tcp-flags SYN,RST SYN,RST -j DROP
    -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    -A INPUT -j KNOCKING
    -A KNOCKING -m recent --rcheck --seconds 10 --name OK3 --mask 255.255.255.255 --rsource -j SUCCESS
    -A KNOCKING -m recent --rcheck --seconds 3 --name OK2 --mask 255.255.255.255 --rsource -j GATE3
    -A KNOCKING -m recent --rcheck --seconds 3 --name OK1 --mask 255.255.255.255 --rsource -j GATE2
    -A KNOCKING -j GATE1
    -A GATE1 -p tcp -m tcp --dport 1234 -m recent --set --name OK1 --mask 255.255.255.255 --rsource -j DROP
    -A GATE1 -j DROP
    -A GATE2 -m recent --remove --name OK1 --mask 255.255.255.255 --rsource
    -A GATE2 -j LOG --log-prefix "In knock 2: " --log-level 6
    -A GATE2 -p tcp -m tcp --dport 2345 -m recent --set --name OK2 --mask 255.255.255.255 --rsource -j DROP
    -A GATE2 -j GATE1
    -A GATE3 -m recent --remove --name OK2 --mask 255.255.255.255 --rsource
    -A GATE3 -j LOG --log-prefix "In knock 3: " --log-level 6
    -A GATE3 -p tcp -m tcp --dport 3456 -m recent --set --name OK3 --mask 255.255.255.255 --rsource -j DROP
    -A GATE3 -j GATE1
    -A SUCCESS -m recent --remove --name OK3 --mask 255.255.255.255 --rsource
    -A SUCCESS -j LOG --log-prefix "Knock accepted!: " --log-level 6
    -A SUCCESS -p tcp -m tcp --dport 22 -j ACCEPT
    -A SUCCESS -j GATE1
    COMMIT
    ```

Some of the parameters set in these extracts are set automatically by iptables (and you find them in `iptables-saves` output). Here is an iptables script that does exactly the same thing than the previous files, but using the least required parameters. Be careful, executing it will reset all your iptables `INPUT`/`OUTPUT` filters (but will not end active connections)

??? abstract "Click to reveal the iptables port kocking script"

    ```sh
    # Consider that you start from a fresh, default, empty iptables

    # Stateful
    iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    # DEFAULT POLICIES
    iptables -P INPUT DROP

    # CHAINS FOR PORT KNOCKING
    # main chain
    iptables -N KNOCKING
    # subchains
    iptables -N GATE1
    iptables -N GATE2
    iptables -N GATE3
    iptables -N SUCCESS

    # Loopback
    iptables -A INPUT -i lo -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT

    # Drop XMAS & NULL scans
    iptables -A INPUT -p tcp --tcp-flags FIN,URG,PSH FIN,URG,PSH -j DROP
    iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
    iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
    iptables -A INPUT -p tcp --tcp-flags SYN,RST SYN,RST -j DROP

    # (your INPUT rules...)

    # PORT KNOCKING

    iptables -A INPUT -j KNOCKING

    # Knocking chain
    # If rhost is tagged OK3 since less than 10s, redirect to SUCCESS
    iptables -A KNOCKING -m recent --rcheck --seconds 10 --name OK3 -j SUCCESS
    # Restrict time between each knock to 3s
    iptables -A KNOCKING -m recent --rcheck --seconds 3 --name OK2 -j GATE3
    iptables -A KNOCKING -m recent --rcheck --seconds 3 --name OK1 -j GATE2
    iptables -A KNOCKING -j GATE1

    # first target: set initial flag
    iptables -A GATE1 -p tcp --dport 1234 -m recent --name OK1 --set -j DROP
    iptables -A GATE1 -j DROP

    # second target: remove previous flag and set the new one
    iptables -A GATE2 -m recent --name OK1 --remove
    iptables -A GATE2 -j LOG --log-level 6 --log-prefix "In knock 2: "
    iptables -A GATE2 -p tcp --dport 2345 -m recent --name OK2 --set -j DROP
    # if the host doesnt match required port, redirect to GATE1
    iptables -A GATE2 -j GATE1

    # third target
    iptables -A GATE3 -m recent --name OK2 --remove
    iptables -A GATE3 -j LOG --log-level 6 --log-prefix "In knock 3: "
    iptables -A GATE3 -p tcp --dport 3456 -m recent --name OK3 --set -j DROP
    # if the host doesnt match required port, redirect to GATE1
    iptables -A GATE3 -j GATE1

    # Knocking process complete, open the way to SSH port
    iptables -A SUCCESS -m recent --name OK3 --remove
    iptables -A SUCCESS -j LOG --log-level 6 --log-prefix "Knock accepted!: "
    iptables -A SUCCESS -p tcp --dport 22 -j ACCEPT
    iptables -A SUCCESS -j GATE1
    ```

---

## Is it possible to open multiple connections simultaneously?

Yes, it is. Our firewall here is stateful (which is obviously required to maintain a connection longer than 10 seconds). Your established connections fall into the `ESTABLISHED` rule. Therefore, they do not interfer with a new port knocking process (which will attempt new connections).

---

## Going further

A thing I like to do is to perform the port knocking, open SSH tunnels, and then access to multiple services without having to do the port knocking again. You can also configure your last rules to forward the connection to other equipments on your network.

I haven't tried it yet, but you can imagine a VPN server behind your port knocking. This way, your VPN service is hidden from the internet, and once connected to it, you don't have to care about port knocking anymore.

The limit here is your imagination!
