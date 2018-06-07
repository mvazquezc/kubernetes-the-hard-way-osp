# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network routes.

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` network.

Create network routes for each worker instance:

```
for i in 0 1 2; do
  SUBNET=$(openstack server show worker-${i}.${DOMAIN} -f value -c properties | awk -F= '{ print $2 }' | tr -d "\'")
  INTERNAL_IP=$(openstack server show worker-${i}.${DOMAIN} -f value -c addresses | awk -F'[=,]' '{ print $2 }')
  openstack router set kubernetes-the-hard-way-router --route destination=${SUBNET},gateway=${INTERNAL_IP}
done
```

List the routes in the `kubernetes-the-hard-way-router` router:

```
openstack router show kubernetes-the-hard-way-router -f value -c routes
```

> output

```
destination='10.200.0.0/24', gateway='10.240.0.20'
destination='10.200.1.0/24', gateway='10.240.0.21'
destination='10.200.2.0/24', gateway='10.240.0.22'
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
