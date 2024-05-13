# Steps to create Galera VLAN from marketplace

# Overview:

The following document provides a step by step guide to convert the marketplace Galera cluster from private to VLAN based connectivity.

In this example, we will setup a VLAN CIDR of 10.0.0.0/24.

| Galera node 1 | 10.0.0.1/24 |
| --- | --- |
| Galera node 2 | 10.0.0.2/24 |
| Galera node 3 | 10.0.0.3/24 |

For your own implementation, feel free to use your own desired CIDR range and VLAN IP allocation based your preference.

This document assumes that you have provisioned a galera cluster successfully via [marketplace](https://www.linode.com/docs/products/tools/marketplace/guides/galera-cluster/)

---

# Steps Required:

### 1. Allocate VLAN IPs to Galera nodes

When the Galera cluster is created, depending on your configuration, the nodes may not have VLAN IP assigned. Hence, assign the VLAN IP to the nodes.

```jsx
1. Go to Linode Cloud Manager
2. Select linode -> configurations tab -> Click on Edit
3. Go to eth1 -> select VLAN -> input IPAM Address (ex 10.0.0.1/24)
4. Reboot node(s) **ALWAYS reboot the node(s) 1 at a time**
		
```

SSH to node and ensure that VLAN IP are assigned in eth1

```jsx
ip addr | grep 10.0.0
```

Exit Criteria: Ensure that all nodes has VLAN IP assigned.

### 2. Update each Galera Node firewall to accept VLAN CIDR

Each nodes are protected by firewalld daemon. Update firewalld daemon to accept request within the VLAN CIDR.

```jsx
1. Add internal firewall range of vlan cidr to all 3 nodes and restart each firewall service
firewall-cmd --zone=internal --add-source=10.0.0.0/24 --permanent
firewall-cmd --reload

//optional - clean up the private ips by removing them from firewalld
sudo firewall-cmd --zone=internal --permanent --remove-source=192.168.XX.XX/32 
sudo firewall-cmd --zone=internal --permanent --remove-source=192.168.XX.XX/32 
sudo firewall-cmd --zone=internal --permanent --remove-source=192.168.XX.XX/32 
firewall-cmd --reload
```

Check that the firewalld has been updates

```jsx
firewall-cmd --list-all --zone=internal
```

Exit Criteria: Ensure that CIDR range is added into the node’s firewall.

### 3. Update Galera configuration to use VLAN IPs

Update on every Galera node.

```jsx
nano /etc/mysql/conf.d/galera.cnf
wsrep_cluster_address="gcomm://10.0.0.1,10.0.0.2,10.0.0.3"
wsrep_node_address="10.0.0.X"

//also update bind-address to allow remote connections
nano /etc/mysql/mariadb.conf.d/50-server.cnf
bind-address=0.0.0.0

//restart service
service mysql restart
```

Verify mysql and Galera are working

```jsx
service mysql status

//check cluster health
sql -u root -p
SHOW STATUS LIKE 'wsrep_cluster_%';
```

Exit Criteria: Ensure that mysql and cluster are running health with 3 nodes registered.

At this stage, you will have a working Galera cluster that is communicating via vlan.

---

# Additional Test

Verify the entire cluster is working by create a database and run a remote sql test.

### 1. Create database and table

```jsx
mysql -u root -p 
create database test;
create table test.flowers (`id` varchar(10));
show tables in test;
```

### 2. Create a remote user

```jsx
CREATE USER  'app'@'10.0.0.%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON test.* TO 'app'@'10.0.0.%';

//Verify that user is created
SELECT User, Host FROM mysql.user;
```

### 3. Login from a remote client

Ensure that the remote client has a VLAN Ip address

```jsx
//install mysql client package
apt install mysql-client

//login mysql
mysql -u app -h 10.0.0.1 -p

//show tables
show tables in test;

```

Success Criteria: Remote Client is able to see flowers table.

---

# Conclusion

By including the additional steps above, we will be able to provision a VLAN based Galera cluster in Linode.
