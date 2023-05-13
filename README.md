# iptables-examples
Some usual iptables rules


# Block Invalid packets

* Block invalid packet in filter table
```
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```
* Block invalid packets in Mangle table before Input table to increase performance
```
sudo iptables -t mangle -A PREROUTING -m conntrack --ctstate INVALID -j DROP
# Test : Should block these scans
# Window scan (Send ack packets)
sudo nmap -sW <serverIP>
# Xmas scan
sudo nmap -sX <serverIP>
```

# Delete rule
* Delete by line number
```
iptables -t mangle -L --line-numbers
iptables -t mangle -D PREROUTING <linenumber>
```

# Example 1: Some filter rules compatible with the DOCKER-USER chain
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