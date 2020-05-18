# ansible-role-iptables_docker

Add local firewall rules to server via iptables for Docker, and Docker Swarm. This will actually protect your Docker containers!  
This role exists because firewalld and Docker (and Docker Swarm) do not get along.  

Works with Docker (and Docker Swarm).  
Also works with Docker Swarms undocumented use of encrypted overlay networks.  

This was suppose to be a simple task. Secure Docker with a firewall. But unfortuanately it is not. I've tried to keep this as simple as possible.  

Features

* Secure by default. Once installed only Docker IPs can access all containers, and other OS processes that have open ports on the server(s).  
* You don't need to be an expert with iptables to use this.  
* Add IPs that are allowed to communicate with Docker containers, and the other OS processes that have open ports.  
* Open specific Docker container port, or server process' port to everyone.  
* Interfaces can also be specified. By default all interfaces are filtered (Secure by default). You can also filter specific network interface(s) and allow all other interfaces.  
* Everything done in "offline" mode. So there should be no issues with Docker when iptables starts.

Make sure you test in non-production first, I cannot make any guarantees.  
Be careful, this will remove and add iptables rules on the OS. Use with caution.  
Existing iptables rules could be removed! Confirm what you have setup before running this.  

This is using iptables as the firewall, and ipset to allow iptables to have a list of IPs that are allowed.  

iptables chains used, and how:  
INPUT, not flushed. Rule inserted at top to jump to custom chain.  
DOCKER-USER, flushed. All Docker (and Docker Swarm related rules are here to block containers from being exposed to everyone by default.)  
FILTERS, flushed. Custom chain for server's processes (that aren't Docker)  

iptables manual: <http://ipset.netfilter.org/iptables.man.html>  

**Note about IPs**: This is for IPv4 only. IPv6 has not been tested. It is safer if you disable IPv6 on your servers.  

**Other security consideration**:  
If using non-Swarm (normal Docker), consider also binding a port to an internal IP for better security.
If using Swarm, consider using specific IPs for Docker Swarm communication
Eg. `docker swarm init --advertise-addr 192.168.100.100 --listen-addr=192.168.100.100 --data-path-addr=192.168.100.100`

**Important Note**: Docker and firewalld do not get along. This role has a check enabled to fail this role if the firewalld service is running or enabled.  
For more information about firewalld and Docker:  
<https://success.docker.com/article/why-am-i-having-network-problems-after-firewalld-is-restarted>  
<https://www.tripwire.com/state-of-security/devops/psa-beware-exposing-ports-docker/>  
<https://docs.docker.com/network/iptables/>  

## Docker versions tested

Docker Engine - Community Edition version:

* 19.03.8

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

## User Settings

* Zone Config

```txt
Optionally specify the Docker server IPs. If not set, IPs will be determined from docker_hosts group in Ansible inventory.
```

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

```txt
Only listed adapters will be blocked. Others will be allowed through. Defaults to block all (with '+').  
If you want to restrict only specific network interface use exact name.  
If you want to restrict all interfaces of the same type, use "interface+" to match every interface, since + is the wildcard for iptables.  
Eg. To restrict the ethX interfaces, use "eth+". "eth+" is a wildcard for anything starting with eth.  
DO NOT use "*". This is not a wildcard and matches nothing!  
The less here the better. Safer to block all ('+') but if cannot, add network adapters with high traffic first.  
local (lo) is not needed here.  
```

```yaml
iptables_docker_external_network_adapter:
  - "+" #Wildcard for everything
  # - "eth+"
  # - "enp0s+"
  # - "wlp1s+"
```

* OS tcp ports open to public

```txt
Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all tcp traffic to these ports from iptables level.  
Only for ports on OS, not for Docker containers.
```

```yaml
iptables_docker_global_ports_allow_tcp:
  - 22                   # SSH
```

* OS udp ports open to public

```txt
Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all udp traffic to these ports from iptables level.  
Only for ports on OS, not for Docker containers.
```

```yaml
iptables_docker_global_ports_allow_udp: []
```

* Network adapter to restrict for Docker rules

```txt
Defaults to use the same setup as the network adapter for the OS.
```

```yaml
iptables_docker_swarm_network_adapter: "{{ iptables_docker_external_network_adapter }}"
# iptables_docker_swarm_network_adapter:
#   - "+" #Wildcard for everything
#   # - "eth+"
```

* Docker tcp ports open to public

```txt
Docker Swarm ports aren't needed here.
Add container tcp ports you want open to everyone.
```

```yaml
# iptables_docker_swarm_ports_allow_tcp: []
iptables_docker_swarm_ports_allow_tcp:
  - 9000
```

* Docker udp ports open to public

```txt
Docker Swarm ports aren't needed here.
Add container udp ports you want open to everyone.
```

```yaml
iptables_docker_swarm_ports_allow_udp: []
```

## Example config file (inventories/dev-env/group_vars/all.yml)

From the example below:  
IPs will be added to the trusted list:

* `192.168.100.1`
* `192.168.101.0/24`

All network interfaces will be restricted since using wildcard '+' for iptables_docker_external_network_adapter.  

Port 22 will be open publically.

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

## iptables Command Reference

More commands can be found in iptables documentation: <http://ipset.netfilter.org/iptables.man.html>

List iptables that are active:

```bash
iptables -nvL --line-numbers
```

Misc useful commands:

```bash
cat /etc/sysconfig/ipset.d/ip_allow.set

ipset list | head


```

## TODO

* [x] Check for firewalld and fail out if running or enabled
* [x] Problem with iptables saving Docker rules in iptables rules? Edit iptables-service to not save Docker-* (besides DOCKER-USER)? grep out Docker rules when saving.
* [x] iptables_docker_ip_allow_set can't be empty. If it is, there's no point to this since nothing is blocked!
* [x] add check in network adapters for * and error
* [x] add automatic list of docker IPs in allowed list (uses IPs from inventory group docker_hosts)
* [x] Change auto Docker server trusted IPs so can override
* [x] confirm "when" and "tags" are ok
* [ ] works on Ubuntu? Ubuntu doesn't have iptables-services or ipset-service. has iptables-persistent and ipset-?
* [ ] ipv6?? This is for ipv4 only
* [ ] cleanup iptables.j2 to remove old junk that's commented out and useless
* [ ] test UDP container
* [ ] add Molecule test? Single node swarm mode only?
* [ ] Publish to Galaxy and add notifications back in .travis.yml

## Author

Ryan Daniels
