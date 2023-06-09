# iptables-examples
Some usual iptables rules


## Usuall rules

```
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# ssh port
sudo iptables -A INPUT -p tcp --dport ssh -j ACCEPT
# When a machine is DNS server
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT

# Add loopback rules
sudo iptables -I INPUT 1 -i lo -j ACCEPT
```

## Block Invalid packets

* Block invalid packet in filter table
```
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```
* Block invalid packets in Mangle table before Input table to increase performance
```
sudo iptables -t mangle -A PREROUTING -m conntrack --ctstate INVALID -j DROP
sudo iptables -t mangle -A PREROUTING -p tcp -m tcp ! --tcp-flags FIN,SYN,RST,ACK SYN -m conntrack --ctstate NEW -j DROP
# Test : Should block these scans
# Window scan (Send ack packets)
sudo nmap -sW <serverIP>
# Xmas scan
sudo nmap -sX <serverIP>
```

## Delete rule
* Delete by line number
```
iptables -t mangle -L -nv --line-numbers
iptables -t mangle -D PREROUTING <linenumber>
```
## Drop policy
To drop packets as default policy, You have two ways:
1. Add DROP rule at the end of the table
```
sudo iptables -A INPUT -j DROP
```
2. Set default DROP policy
```
sudo iptables -P INPUT DROP
```

## Block ICMP
ICMP types:
* type-3: Desitination unreachable(Used to say large packets should be fragmented)
* type-11: Time excceeded messages
* type-12: Bad IP header
* type-0 and type-8: Infamous ping packets (echo and it reply)
* type-5: Redirect messages, which a hacker may use to redirect to someplace
```
-A INPUT -m conntrack -p icmp --icmp-type 3 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
-A INPUT -m conntrack -p icmp --icmp-type 11 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
-A INPUT -m conntrack -p icmp --icmp-type 12 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
```

## IPV6
In iptables you need to set IPV6 rules separately with **ip6tables** command.
You must set this rule even if you don't use ipv6 facing Internet, they are ways to tunnel ipv6 over ipv4.
```
sudo ip6tables -A INPUT -i lo -j ACCEPT
sudo ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport ssh -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 53 -j ACCEPT
sudo ip6tables -A INPUT -i lo -j ACCEPT
```
ICMPv6 messages have more usage than ipv4 version because ipv6 use icmp for ARP, DHCP, tunneling and etc.
```
##
# Destination unreachable
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 1 -j ACCEPT
# Packet too big
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT
# Time exceeded
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 3 -j ACCEPT
# Packet header issue
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 4 -j ACCEPT
# echo request and reply for tunneling
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 128 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 129 -j ACCEPT

# Multicast messages
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 130 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 131 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 132 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 143 -j ACCEPT

sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 151 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 152 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 153 -j ACCEPT

# Neighbor and Reouter discovery messages
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 136 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 141 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 142 -j ACCEPT

# Authentication with routers
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 148 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 149 -j ACCEPT

```

Drop policy or Default rule
```
sudo ip6tables -A INPUT -j DROP
```

## Logging
Logging to syslog
```
sudo iptables -I INPUT 5 -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
```

## Allow All
Sometimes for TShooting you need to allow all traffic
```
echo "Stopping firewall and allowing everyone..."
# Delete rules
iptables -F
# Delete user-defined chains
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```
Run it as a bash script

## User-Defined chains
```
# Create Chain and its rules
sudo iptables -N MYCHAIN
sudo iptables -A MYCHAIN -s <ip> -j ACCEPT
# Return tracking jump back the callee chain
sudo iptables -A MYCHAIN -j RETURN

# Add chain to iptables main chains
sudo iptables -A INPUT -j MYCHAIN
```

## Useful Commands

* Show All tables rules:
```
for t in filter nat mangle raw security; do
   echo "table $t:"
   sudo iptables -t $t -L -nv
done
```
* Show number of pkts and bytes a (Filter rule) rule passed/dropped:
```
sudo iptables -L -nv
```
To reset counters:
```
sudo iptables -Z
```



# Examples
## Example 1: Some filter rules compatible with the DOCKER-USER chain
```
-A OUTPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -o lo -j ACCEPT
-A OUTPUT -p icmp -m conntrack --ctstate NEW -m limit --limit 1/sec --limit-burst 1 -m icmp --icmp-type 8 -j ACCEPT
-A OUTPUT -d <ip> -j ACCEPT
-A OUTPUT -d <ip1>,<ip2> -j ACCEPT
-A OUTPUT -p udp -m multiport --dport 53,123 -j ACCEPT
-A OUTPUT -p tcp -m multiport -d <mail_server_ip> --dport 25,26,465,587,993 -j ACCEPT -m comment --comment "output main server"
-A OUTPUT -j DROP
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -p icmp -m conntrack --ctstate NEW -m limit --limit 3/sec --limit-burst 1 -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -s <ip/cidr> -j ACCEPT
-A INPUT -s <ip1>,<ip2> -j ACCEPT
-A INPUT -p udp -m multiport --sport 53,123 -j ACCEPT
-A INPUT -j DROP
-A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A DOCKER-USER -s 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16 -j ACCEPT
# It is needed because some packets related to docker containers go to DOCKER-USER chain while some others go through INPUT chains. So you should add same rule for DOCKER-USER and INPUT chains
-A DOCKER-USER -s <ip1>,<ip2> -j ACCEPT
-A DOCKER-USER -j DROP
-A DOCKER-USER -j RETURN

```

# Notes
* If you don't set `-t` in iptables commands, *filter* tables is used as default.
* You can use *UFW* instead of pure iptables. It defined some usuall protection rules in advance. For example in `/etc/ufw/before.rules` you can find some predefined icmp filters.
```
# ok icmp codes for INPUT
-A ufw-before-input -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-input -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-input -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT

# ok icmp code for FORWARD
-A ufw-before-forward -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT
```
Thus, it seems a good idea to use ufw and add custom iptables rules to **.rules* in `/etc/ufw` directory.

# Referece:
* My main reference is [Masting Linux Security and Hardening](https://www.packtpub.com/product/mastering-linux-security-and-hardening-second-edition/) book by Donald A.Tevault.
* [Ubuntu basic tutorial](https://help.ubuntu.com/community/IptablesHowTo)
* [A Deep Dive into Iptablse and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
* [Iptables targets and jumps](https://www.frozentux.net/iptables-tutorial/chunkyhtml/c3965.html)
* [NAT iptables tutorial](https://www.karlrupp.net/en/computer/nat_tutorial)