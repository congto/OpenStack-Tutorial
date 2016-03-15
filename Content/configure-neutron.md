###Configure Neutron with flat external network
In the basic scenario with flat external network, only administrative users can manage external networks because they use the physical network infrastructure. We are going to create a shared external network to be used by all tenants.

On the Control node, login as ``admin`` Keystone user
```
# source keystonerc_admin
# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 22bdc5a0210e4a96add0cea90a6137ed | bcloud   |
| 5ccf7027366442709bde78831da6cce2 | services |
| 613b2bc016c5428397b3fea6dc162af1 | admin    |
+----------------------------------+----------+
```
Create the external network

```
# neutron net-create external-flat-network \
> --tenant-id 5ccf7027366442709bde78831da6cce2 \
> --shared \
> --provider:network_type flat \
> --provider:physical_network external \
> --router:external True

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 6ede0952-25f7-489d-9ce4-0126da7cb7d0 |
| mtu                       | 0                                    |
| name                      | external-flat-network                |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 5ccf7027366442709bde78831da6cce2     |
+---------------------------+--------------------------------------+
```

Pay attention to the following parameters:

* ``provider:network_type flat``
* ``provider:physical_network external``
* ``router:external True``
* ``shared``

The external network shares the same subnet and gateway associated with the physical network connected to the external interface on the network node. Specify an exclusive slice of this subnet for router and IP addresses to prevent interference with other devices on the same external network

```
# neutron subnet-create external-flat-network 172.16.1.0/24  \
--name external-flat-subnetwork \
--tenant-id 5ccf7027366442709bde78831da6cce2 \
--gateway 172.16.1.1 \
--disable-dhcp \
--allocation-pool start=172.16.1.200,end=172.16.1.220

Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "172.16.1.200", "end": "172.16.1.220"} |
| cidr              | 172.16.1.0/24                                    |
| dns_nameservers   |                                                  |
| enable_dhcp       | False                                            |
| gateway_ip        | 172.16.1.1                                       |
| host_routes       |                                                  |
| id                | 40f89cb3-9474-48e0-ab4c-7fa3fb57009e             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | external-flat-subnetwork                         |
| network_id        | 6ede0952-25f7-489d-9ce4-0126da7cb7d0             |
| subnetpool_id     |                                                  |
| tenant_id         | 5ccf7027366442709bde78831da6cce2                 |
+-------------------+--------------------------------------------------+
```

The tenant networks are created by the tenant users. Login as a tenant user and create a tenant network
```
# source keystonerc_bcloud
# neutron net-create tenant-network
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 72563bb0-b5ea-44d0-9a59-f95f7cc12671 |
| mtu             | 0                                    |
| name            | tenant-network                       |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 22bdc5a0210e4a96add0cea90a6137ed     |
+-----------------+--------------------------------------+
```

Like the external network, a tenant network also requires a subnet attached to it. By default, this subnet will use DHCP so instances can obtain IP addresses.
```
# neutron subnet-create tenant-network 192.168.1.0/24 \
--name tenant-subnetwork \
--gateway 192.168.1.1 \
--enable-dhcp \
--dns-nameserver 8.8.8.8 \
--allocation-pool start=192.168.1.10,end=192.168.1.250

Created a new subnet:
+-------------------+---------------------------------------------------+
| Field             | Value                                             |
+-------------------+---------------------------------------------------+
| allocation_pools  | {"start": "192.168.1.10", "end": "192.168.1.250"} |
| cidr              | 192.168.1.0/24                                    |
| dns_nameservers   | 8.8.8.8                                           |
| enable_dhcp       | True                                              |
| gateway_ip        | 192.168.1.1                                       |
| host_routes       |                                                   |
| id                | 45f5980b-5a15-4372-93ab-b77502a6104a              |
| ip_version        | 4                                                 |
| ipv6_address_mode |                                                   |
| ipv6_ra_mode      |                                                   |
| name              | tenant-subnetwork                                 |
| network_id        | 72563bb0-b5ea-44d0-9a59-f95f7cc12671              |
| subnetpool_id     |                                                   |
| tenant_id         | 22bdc5a0210e4a96add0cea90a6137ed                  |
+-------------------+---------------------------------------------------+
```