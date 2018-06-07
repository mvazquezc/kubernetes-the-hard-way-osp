# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Public Network
The public network is a network that contains external access and can be
reached by the outside world. The public network creation can be only done by
an OpenStack administrator.

The following commands provide an example of creating an OpenStack provider
network for public network access.

As an OpenStack administrator:

```
source /path/to/admin-rc

openstack network create \
    --external \
    --provider-network-type flat \
    --provider-physical-network datacentre \
    public_network

openstack subnet create \
    --gateway <ip> \
    --allocation-pool start=<float_start>,end=<float_end> \
    --network public_network \
    --subnet-range <CIDR> \
    public_subnet
```

> <float_start> and <float_end> are the associated floating IP pool provided to the network labeled public network. The Classless Inter-Domain Routing (CIDR) uses the format <ip>/<routing_prefix>, i.e. 10.5.2.1/24

### Private Tenant Network
The private tenant network is connected to the public network via a router during the network setup. This allows each OpenStack Platform instance attached to the internal network the ability to request a floating IP from the public network for public access.

> Follow the OpenStack [documentation](https://docs.openstack.org/neutron/latest/admin/) for more information.

Create the `kubernetes-the-hard-way` private network:

```
openstack network create kubernetes-the-hard-way
```

The `kubernetes-the-hard-way-router` router:

```
openstack router create kubernetes-the-hard-way-router
```

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` network:

```
openstack subnet create --network kubernetes-the-hard-way \
  --subnet-range 10.240.0.0/24 \
  kubernetes
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

Once the subnet is created, the router should be configured as the default
gateway

```
openstack router add subnet kubernetes-the-hard-way-router $(openstack subnet show kubernetes -f value -c id)
openstack router set \
  --external-gateway $(openstack network show public_network -f value -c id) \
  kubernetes-the-hard-way-router
```

### Security groups

Create a security group that allows internal communication across all protocols:

```
openstack security group create kubernetes-the-hard-way-allow-internal

for protocol in tcp udp icmp
do
  openstack security group rule create \
      --ingress \
      --protocol ${protocol} \
      --remote-group kubernetes-the-hard-way-allow-internal \
      kubernetes-the-hard-way-allow-internal
done
```

Create a security group that allows external SSH, ICMP, and HTTPS:

```
openstack security group create kubernetes-the-hard-way-allow-external

openstack security group rule create \
    --ingress \
    --protocol icmp \
    kubernetes-the-hard-way-allow-external

for port in 22 6443
do
  openstack security group rule create \
      --ingress \
      --protocol tcp \
      --dst-port ${port} \
      kubernetes-the-hard-way-allow-external
done
```

List the security groups:

```
openstack security group list -f table -c ID -c Name
```

> output

```
+--------------------------------------+----------------------------------------+
| ID                                   | Name                                   |
+--------------------------------------+----------------------------------------+
| 283e6cf0-031d-46f8-b13c-d08556b1876e | kubernetes-the-hard-way-allow-internal |
| 665ce130-e098-4284-b784-319db7a6855f | kubernetes-the-hard-way-allow-external |
+--------------------------------------+----------------------------------------+
```
### Images
To be able to create instances, an image should be provided. Depending on the
OpenStack environment an image catalog can be provided. In this guide we will
use CentOS as the base operating system. Follow the nexts steps to upload a new
CentOS image:

Download the latest CentOS Generic Cloud image (at the time of writting this guide, 1804)

```
wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1804_02.raw.tar.gz
tar xzvf CentOS-7-x86_64-GenericCloud-1804_02.raw.tar.gz
```

Upload it to the Glance service:

```
openstack image create CentOS-7-x86_64-GenericCloud-1804_02 \
    --disk-format=raw \
    --container-format=bare < CentOS-7-x86_64-GenericCloud-1804_02.raw
```

### DNS

It is required to have a proper DNS configuration. If not using DNSaaS, you can
create an instance and setup a proper DNS following the next instructions.

Create a security group to allow DNS communication within the internal network:

```
openstack security group create kubernetes-the-hard-way-allow-dns
openstack security group rule create \
    --ingress \
    --protocol icmp \
    kubernetes-the-hard-way-allow-dns
openstack security group rule create \
    --ingress \
    --protocol tcp \
    --dst-port 22 \
    kubernetes-the-hard-way-allow-dns
for PROTO in udp tcp;
do
  openstack security group rule create \
    --ingress \
    --protocol $PROTO \
    --dst-port 53 \
    --remote-ip 10.240.0.0/24 \
    kubernetes-the-hard-way-allow-dns;
done
```

Create an instance to host the DNS service. In this case the CentOS image is
used.

```
export DOMAIN='k8s.lan'
openstack server create \
    --nic net-id=$(openstack network show kubernetes-the-hard-way -f value -c id),v4-fixed-ip=10.240.0.100 \
    --security-group=kubernetes-the-hard-way-allow-dns \
    --flavor m1.small \
    --image CentOS-7-x86_64-GenericCloud-1804_02 \
    --key-name k8s-the-hard-way \
    dns.${DOMAIN}
```

Add a floating ip to make it reachable externally:

```
openstack server add floating ip dns.${DOMAIN} $(openstack floating ip create public_network --description dns -f value -c floating_ip_address)
```

> The domain used in this guide is configured to be k8s.lan.

#### DNS service

The following steps shows how to install a DNS service using named in the
instance previously created.

First, connect to the instance:

```
export DNS_EXTERNAL_IP=$(openstack server show dns.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
ssh -i ~/.ssh/k8s.pem centos@${DNS_EXTERNAL_IP}
```

Configure network resolution and install named:

```
export DOMAIN='k8s.lan'
export UPSTREAM_DNS=$(awk '/nameserver/ { print $2 }' /etc/resolv.conf)
sudo tee -a /etc/sysconfig/network-scripts/ifcfg-eth0 << EOF
DNS1=${UPSTREAM_DNS}
PEERDNS=no
EOF

sudo yum install -y firewalld python-firewall bind-utils bind

sudo systemctl enable firewalld --now
sudo firewall-cmd --zone public --add-service dns
sudo firewall-cmd --zone public --add-service dns --permanent

sudo cp /etc/named.conf{,.orig}

sudo tee /etc/named.conf << EOF
options {
  listen-on port 53 { any; };
  directory "/var/named";
  dump-file "/var/named/data/cache_dump.db";
  statistics-file "/var/named/data/named_stats.txt";
  memstatistics-file "/var/named/data/named_mem_stats.txt";
  allow-query { any; };
  forward only;
  forwarders { ${UPSTREAM_DNS}; };
  managed-keys-directory "/var/named/dynamic";
  pid-file "/run/named/named.pid";
  session-keyfile "/run/named/session.key";
};

logging {
  channel default_debug {
    file "data/named.run";
    severity dynamic;
  };
};

zone "." IN {
  type hint;
  file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

include "/etc/named/zones.conf";
EOF

sudo tee /etc/named/zones.conf << EOF
include "/etc/named/update.key" ;

zone ${DOMAIN} {
  type master ;
  file "/var/named/dynamic/zone.db" ;
  allow-update { key update-key ; } ;
};
EOF

sudo rndc-confgen -a -c /etc/named/update.key -k update-key -r /dev/urandom
sudo chown root.named /etc/named/update.key
sudo chmod 640 /etc/named/update.key

export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
sudo tee /var/named/dynamic/zone.db << EOF
\$ORIGIN .
\$TTL 300	; 5 minutes
${DOMAIN} IN SOA dns.${DOMAIN}. admin.${DOMAIN}. (
        1          ; serial
        28800      ; refresh (8 hours)
        3600       ; retry (1 hour)
        604800     ; expire (1 week)
        86400      ; minimum (1 day)
        )
        NS dns.${DOMAIN}.
\$ORIGIN ${DOMAIN}.
\$TTL 3600	; 1 hour
dns          A ${INTERNAL_IP}
controller-0 A 10.240.0.10
controller-1 A 10.240.0.11
controller-2 A 10.240.0.12
worker-0     A 10.240.0.20
worker-1     A 10.240.0.21
worker-2     A 10.240.0.22
EOF
```

Verify everything is properly configured:

```
sudo named-checkconf /etc/named.conf
sudo named-checkzone ${DOMAIN} /var/named/dynamic/zone.db
```

Start and enable the named service:

```
sudo systemctl enable named --now
```

Verify it is working

```
dig @${INTERNAL_IP} ${DOMAIN} axfr
```

Finally, update all packages to latest version and reboot the instance just in
case:

```
sudo yum clean all
sudo yum update -y
sudo reboot
```

After installing the DNS service, modify the private subnet to use that DNS
service:

```
DNS_INTERNAL_IP=$(openstack server show dns.${DOMAIN} -f value -c addresses | awk -F'[=,]' '{print $2}')
openstack subnet set --dns-nameserver ${DNS_INTERNAL_IP} kubernetes
```

### Load Balancer
In order to have a proper Kubernetes high available environment, a Load balancer
is required to distribute the API load. If not using LBaaS, you can create an
instance and setup a proper Load balancer following the next instructions.

Create an instance to host the DNS service. In this case the CentOS image is
used.

```
export DOMAIN='k8s.lan'
openstack server create \
  --nic net-id=$(openstack network show kubernetes-the-hard-way -f value -c id),v4-fixed-ip=10.240.0.200 \
  --security-group kubernetes-the-hard-way-allow-internal \
  --security-group kubernetes-the-hard-way-allow-external \
  --flavor m1.small \
  --image CentOS-7-x86_64-GenericCloud-1804_02 \
  --key-name k8s-the-hard-way \
  k8sosp.${DOMAIN}
```
Add a floating ip to make it reachable externally:

```
openstack server add floating ip k8sosp.${DOMAIN} $(openstack floating ip create public_network --description k8s-lb -f value -c floating_ip_address)
```

#### Load balancer service

The following steps shows how to install a load balancer service using HAProxy in the instance previously created.

First, connect to the instance:

```
export LB_EXTERNAL_IP=$(openstack server show k8sosp.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
ssh -i ~/.ssh/k8s.pem centos@${LB_EXTERNAL_IP}
```

Install and configure HAProxy

```
export DOMAIN='k8s.lan'
sudo yum install -y haproxy

sudo tee /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats :9000
    stats enable
    stats realm Haproxy\ Statistics
    stats uri /haproxy_stats
    stats auth admin:password
    stats refresh 30
    mode http

frontend  main *:6443
    default_backend mgmt6443

backend mgmt6443
    balance source
    mode tcp
    # MASTERS 6443
    server controller-0.${DOMAIN} 10.240.0.10:6443 check
    server controller-1.${DOMAIN} 10.240.0.11:6443 check
    server controller-2.${DOMAIN} 10.240.0.12:6443 check
EOF
```

As the Kubernetes port is 6443, the selinux policy should be modified to allow
haproxy to listen on that particular port:

```
sudo semanage port --add --type http_port_t --proto tcp 6443
```

Verify everything is properly configured:

```
haproxy -c -V -f /etc/haproxy/haproxy.cfg
```

Start and enable the service

```
sudo systemctl enable haproxy --now
```

Finally, update all packages to latest version and reboot the instance just in
case:

```
sudo yum clean all
sudo yum update -y
sudo reboot
```

### Add Load balancer to DNS

Now that the load balancer has been created, the DNS zone needs to be updated
to add the load balancer.

Connect to the DNS instance:

```
openstack server show k8sosp.${DOMAIN} -f value -c addresses | awk -F'[=,]' '{print $2}' > lb_internal_ip
export DNS_EXTERNAL_IP=$(openstack server show dns.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
scp -i ~/.ssh/k8s.pem lb_internal_ip centos@${DNS_EXTERNAL_IP}:lb_internal_ip
ssh -i ~/.ssh/k8s.pem centos@${DNS_EXTERNAL_IP}
```

Update the zone:

```
export DOMAIN="k8s.lan"
export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
export LB_INTERNAL_IP=$(cat lb_internal_ip)
rm -f lb_internal_ip
sudo nsupdate -k /etc/named/update.key <<EOF
server ${INTERNAL_IP}
zone ${DOMAIN}
update add k8sosp.${DOMAIN} 3600 A ${LB_INTERNAL_IP}
send
quit
EOF
```

Verify it:

```
dig @${INTERNAL_IP} ${DOMAIN} axfr
```

## Compute Instances

Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
export DOMAIN="k8s.lan"
for i in 0 1 2;
do
  openstack server create \
    --nic net-id=$(openstack network show kubernetes-the-hard-way -f value -c id),v4-fixed-ip=10.240.0.1${i} \
    --flavor m1.medium \
    --image CentOS-7-x86_64-GenericCloud-1804_02 \
    --key-name k8s-the-hard-way \
    --security-group kubernetes-the-hard-way-allow-external \
    --security-group kubernetes-the-hard-way-allow-internal \
    controller-${i}.${DOMAIN};
   openstack server add floating ip controller-${i}.${DOMAIN} $(openstack floating ip create public_network -f value -c floating_ip_address)
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
export DOMAIN="k8s.lan"
for i in 0 1 2;
do
  openstack server create \
    --nic net-id=$(openstack network show kubernetes-the-hard-way -f value -c id),v4-fixed-ip=10.240.0.2${i} \
    --flavor m1.medium \
    --image CentOS-7-x86_64-GenericCloud-1804_02 \
    --key-name k8s-the-hard-way \
    --property pod-cidr=10.200.${i}.0/24 \
    --security-group kubernetes-the-hard-way-allow-external \
    --security-group kubernetes-the-hard-way-allow-internal \
    worker-${i}.${DOMAIN};
   openstack server add floating ip worker-${i}.${DOMAIN} $(openstack floating ip create public_network -f value -c floating_ip_address)
done
```

### Verification

List the compute instances:

```
openstack server list -f table -c Name -c Networks -c Flavor -c Status
```

> output

```
+----------------------+--------+-----------------------------------------------+-----------+
| Name                 | Status | Networks                                      | Flavor    |
+----------------------+--------+-----------------------------------------------+-----------+
| worker-2.k8s.lan     | ACTIVE | kubernetes-the-hard-way=10.240.0.22, X.X.X.X  | m1.medium |
| worker-1.k8s.lan     | ACTIVE | kubernetes-the-hard-way=10.240.0.21, X.X.X.X  | m1.medium |
| worker-0.k8s.lan     | ACTIVE | kubernetes-the-hard-way=10.240.0.20, X.X.X.X  | m1.medium |
| controller-2.k8s.lan | ACTIVE | kubernetes-the-hard-way=10.240.0.12, X.X.X.X  | m1.medium |
| controller-1.k8s.lan | ACTIVE | kubernetes-the-hard-way=10.240.0.11, X.X.X.X  | m1.medium |
| controller-0.k8s.lan | ACTIVE | kubernetes-the-hard-way=10.240.0.10, X.X.X.X  | m1.medium |
| k8sosp.k8s.lan       | ACTIVE | kubernetes-the-hard-way=10.240.0.200, X.X.X.X | m1.small  |
| dns.k8s.lan          | ACTIVE | kubernetes-the-hard-way=10.240.0.100, X.X.X.X | m1.small  |
+----------------------+--------+-----------------------------------------------+-----------+
```

> As the DNS server is created internally, it can be a good idea to configure an
external DNS server to use the public ips to avoid using IPs to connect to the
instances (or the local /etc/hosts in your workstation)

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
