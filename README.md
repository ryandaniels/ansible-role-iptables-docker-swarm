# Ansible Role: iptables for Docker (and Docker Swarm)

Add firewall rules to server via iptables, for Docker, and Docker Swarm. This will actually protect your Docker containers!  
This Ansible Role exists because firewalld and Docker (and Docker Swarm) do not get along.  

Problem being solved: When starting a container in Docker with a "published" port, you have no control and the port is exposed through your server's firewall. Even if you were using iptables, or another firewall on your server. Docker opens that "published" port to everyone, and bypasses your firewall.  

Use case for this solution: Allow trusted IPs to connect to Docker containers (and Docker Swarm containers), along with other open OS ports. With an option to expose specified ports publicly (Docker/Docker Swarm and OS). The trusted IPs are not in the same network IP range, or even network subnet.  

This was suppose to be simple. Secure Docker with a firewall. But unfortuanately it is not. I've tried to keep this as simple as possible.  

There could be unknown problems with this.. use at your own risk!  

Currently tested and working on CentOS/RHEL 7.  

## Features

* Works with Docker, and Docker Swarm.
* Secure by default. Once configured, only Docker IPs can access all containers, and all other OS processes that have open ports on the server.  
* Simple as possible. The less iptables rules, the faster performance will be (in theory at least).  
* Automatic. No manually adding ports to the firewall config (if you use a trusted set of IPs)
* Add "trusted" IPs that are allowed to communicate with all Docker containers, and all other OS processes that have open ports on the server.  
* Open specified Docker container ports, or the server's OS ports to the public (everyone) through the firewall, like SSH.  
* Interfaces can also be specified. By default all interfaces are filtered. You could filter specific network interface(s) and allow all other interfaces (only specify an untrusted interface).  
* Everything done in "offline" mode. So there should be no issues with Docker when iptables rules are activated.
* You don't need to be an expert with iptables to use this.  
* Works with Docker Swarm's undocumented use of iptables and encrypted overlay networks. (iptables rules are appened to the INPUT chain).

This solution is using `iptables` as the firewall, and `ipset` to allow iptables to have a list of IPs that are allowed.  

iptables chains used, and how:  
**INPUT**, not flushed. Rule inserted at top to jump to custom chain for OS related rules.  
**DOCKER-USER**, flushed. All Docker (and Docker Swarm) related rules are here to block containers from being exposed to everyone by default. By default only the Docker server IPs are allowed.  
**FILTERS**, flushed. Custom chain for server's processes (that aren't Docker). By default only the Docker server IPs are allowed.  

iptables manual: <http://ipset.netfilter.org/iptables.man.html>  

## Warnings

Don't lock yourself out of your server. This is modifying your firewall. Always have another way to get in, like a "console".  

**Note about IPs**: This is for IPv4 only. IPv6 has not been tested. It is safer if you disable IPv6 on your servers.  

**Other security consideration**:  
If using non-Swarm (normal Docker), consider also binding a port to an internal IP for better security.
If using Swarm, consider using specific IPs for Docker Swarm communication.  
Eg. `docker swarm init --advertise-addr 192.168.100.100 --listen-addr=192.168.100.100 --data-path-addr=192.168.100.100`

**Important Note**: Docker and firewalld do not get along. This Ansible Role has a check enabled to fail this role if the firewalld service is running or enabled.  
For more information about firewalld and Docker:  
<https://success.docker.com/article/why-am-i-having-network-problems-after-firewalld-is-restarted>  
<https://www.tripwire.com/state-of-security/devops/psa-beware-exposing-ports-docker/>  
<https://docs.docker.com/network/iptables/>  

**WARNING**:  
Make sure you test in non-production first, I cannot make any guarantees or held responsible.  
Be careful, this will remove and add iptables rules on the OS. Use with caution.  
Existing iptables rules could be removed! Confirm what you have setup before running this.  

There could be unknown problems with this.. use at your own risk!  

## Docker versions tested

Docker Engine - Community Edition version:

* 19.03.8
* 19.03.9

Tested in normal Docker mode, and with a 3 node Docker Swarm cluster.  

## Distros tested

* CentOS: 7.7, 7.8

## Dependencies

* iptables & iptables-services

Tested with v1.4.21 (Latest available in CentOS 7)  

* ipset & ipset-service

Tested with v7.1 (Latest available in CentOS 7)  

## Default Settings

* Enable debug

```yaml
debug_enabled_default: false
```

* Proxy (Needed when installing required packages if behind a proxy)

```yaml
proxy_env: []
```

* Role disabled by default. Change to true in group_vars or playbook etc

```yaml
iptables_docker_managed: false
```

* Check if (Docker) service is running or enabled, and fail the role

```yaml
iptables_docker_check_problem_service_managed: true
```

* Services to check, and fail the role

```yaml
iptables_docker_check_problem_service:
  - docker.service
```

* Show configuration from variables

```yaml
iptables_docker_show_config: true
```

* Start iptables service

```yaml
iptables_docker_start: true
```

* Install iptables package

```yaml
iptables_docker_managed_pkg: true
iptables_docker_packages:
  - iptables
  - iptables-services
  - ipset
  - ipset-service
```

* Force copy of ipset file to trigger ipset reload

```yaml
iptables_docker_copy_ipset_force: false
```

* Force copy of iptables file to trigger iptables reload

```yaml
iptables_docker_copy_iptables_force: false
```

* iptables saved configuration location

```yaml
iptables_docker_iptables_config_save: /etc/sysconfig/iptables
```

* ipset saved configuration location

```yaml
iptables_docker_ipset_config_dir: /etc/sysconfig/ipset.d
```

* ipset maximum elements (IPs in the allow list)

    If changed after first creation, must be deleted and re-created manually. 64k IPs should be enough.

```yaml
iptables_docker_ipset_maxelem: 65536
```

## User Settings

* Override Docker server IPs (Optional)

    Optionally specify the Docker server IPs. If not set, IPs will be determined from docker_hosts group in Ansible inventory.

```yaml
# iptables_docker_server_ip_allow_set:
#   - 192.168.100.100
#   - 192.168.100.101
#   - 192.168.100.102
```

* IPs allowed to use all Docker container's exposed ports and all server's processes' exposed ports.

```yaml
# iptables_docker_ip_allow_set: []
iptables_docker_ip_allow_set:
  - 192.168.100.1
  - 192.168.101.0/24
  - 192.168.102.0/24
```

* Network adapter to restrict for OS rules

    Only listed adapters will be blocked. Others will be allowed through. Defaults to block all (with '+').  
    If you want to restrict only specific network interface use exact name.  
    If you want to restrict all interfaces of the same type, use "interface+" to match every interface, since + is the wildcard for iptables.  
    Eg. To restrict the ethX interfaces, use "eth+". "eth+" is a wildcard for anything starting with eth.  
    DO NOT use "*". This is not a wildcard and matches nothing!  
    The less here the better. Safer to block all ('+') but if cannot, add network adapters with high traffic first.  
    local (lo) is not needed here.  

```yaml
iptables_docker_external_network_adapter:
  - "+" #Wildcard for everything
  # - "eth+"
  # - "enp0s+"
  # - "wlp1s+"
```

* OS tcp ports open to public

    Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all tcp traffic to these ports from iptables level.  
    Only for ports on OS, not for Docker containers.

```yaml
iptables_docker_global_ports_allow_tcp:
  - 22                   # SSH
```

* OS udp ports open to public

    Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all udp traffic to these ports from iptables level.  
    Only for ports on OS, not for Docker containers.

```yaml
iptables_docker_global_ports_allow_udp: []
```

* Network adapter to restrict for Docker rules

    Defaults to use the same setup as the network adapter for the OS.

```yaml
iptables_docker_swarm_network_adapter: "{{ iptables_docker_external_network_adapter }}"
# iptables_docker_swarm_network_adapter:
#   - "+" #Wildcard for everything
#   # - "eth+"
```

* Docker tcp ports open to public

    Docker Swarm ports aren't needed here.
    Add container tcp ports you want open to everyone.

```yaml
# iptables_docker_swarm_ports_allow_tcp: []
iptables_docker_swarm_ports_allow_tcp:
  - 9000
```

* Docker udp ports open to public

    Docker Swarm ports aren't needed here.
    Add container udp ports you want open to everyone.

```yaml
iptables_docker_swarm_ports_allow_udp: []
```

* Docker bridge network name (docker0), and IP range (for DOCKER-USER iptables source allow)

```yaml
iptables_docker_bridge_name: docker0
iptables_docker_bridge_ips: 172.17.0.0/16
```

* Docker Swarm bridge network IP range (docker_gwbridge) (for DOCKER-USER iptables source allow)

```yaml
iptables_docker_swarm_bridge_name: docker_gwbridge
iptables_docker_swarm_bridge_ips: 172.18.0.0/16
```

## Example config file (inventories/dev-env/group_vars/all.yml)

From the example below:  
IPs will be added to the trusted list:

* `192.168.100.1`
* `192.168.101.0/24`

All network interfaces will be restricted since using wildcard '+' for iptables_docker_external_network_adapter.  

Port 22 will be open publicly.

```yaml
---
iptables_docker_ip_allow_set:
  - 192.168.100.1
  - 192.168.101.0/24

iptables_docker_external_network_adapter:
  - "+" #Wildcard for everything

iptables_docker_global_ports_allow_tcp:
  - 22                   # SSH
```

## Example inventory file

```ini
[docker_hosts]
centoslead1 ansible_host=192.168.100.100
centoswork1 ansible_host=192.168.100.101
centoswork2 ansible_host=192.168.100.102
```

## Example Playbook iptables_docker.yml

```yaml
---
- hosts: '{{ inventory }}'
  become: yes
  vars:
    # Use this role
    iptables_docker_managed: true
  roles:
  - iptables_docker
```

## Usage

By default no tasks will run unless you set `iptables_docker_managed=true`. This is by design to prevent accidents by people who don't RTFM.

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true" -i hosts-dev
```

Skip installing packages (if known already there - speeds up task)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true" -i hosts --skip-tags=iptables_docker_pkg_install
```

Show more verbose output (debug info)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true debug_enabled_default=true" -i hosts-dev
```

Do not start iptables service or add config for iptables

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true iptables_docker_start=false" -i hosts-dev
```

Only show configuration (from variables)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true iptables_docker_show_config=true" -i hosts --tags "iptables_docker_show_config"
```

## Note about ipset size limit

Important: Make note of the size of "Number of entries". If that number is close to the maxelem size (65536), then you need to delete the ipset "ip_allow" and re-create it with a larger max size.  
64K ought to be enough for anyone.  

File is in: `templates\ip_allow.set.j2`

```text
create -exist ip_allow hash:ip family inet hashsize 1024 maxelem 65536
```

Check size of ipset list:

```bash
ipset list |grep "Number of entries"
```

Important output:

```bash
Number of entries: 3
```

## iptables Command Reference

More commands can be found in iptables documentation: <http://ipset.netfilter.org/iptables.man.html>

List iptables that are active:

```bash
iptables -nvL --line-numbers
```

Misc useful commands:

```bash
cat /etc/sysconfig/ipset.d/ip_allow.set
systemctl restart ipset
ipset list | head

iptables -F DOCKER-USER
iptables -F FILTERS
iptables-restore -n < ansible_iptables_docker-iptables

grep -v "^#" ansible_iptables_docker-iptables
iptables -S INPUT
iptables -S DOCKER-USER
iptables -S FILTERS
```

## TODO

* [x] Check for firewalld and fail out if running or enabled
* [x] Problem with iptables saving Docker rules in iptables rules? Edit iptables-service to not save Docker-* (besides DOCKER-USER)? grep out Docker rules when saving.
* [x] iptables_docker_ip_allow_set can't be empty. If it is, there's no point to this since nothing is blocked!
* [x] add check in network adapters for * and error
* [x] add automatic list of docker IPs in allowed list (uses IPs from inventory group docker_hosts)
* [x] Change auto Docker server trusted IPs so can override
* [x] confirm "when" and "tags" are ok
* [ ] Ubuntu? Ubuntu doesn't have iptables-services or ipset-service. has iptables-persistent and ipset-?
* [ ] ipv6?? This is for ipv4 only
* [ ] cleanup iptables.j2 to remove old junk that's commented out and useless
* [x] test UDP container and OS port
* [ ] add test? Molecule? Single node swarm mode only? how to test connection doesn't work from "untrusted" ip?

## Author

Ryan Daniels
