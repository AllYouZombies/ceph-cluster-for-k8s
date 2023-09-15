# CEPH cluster

---

## Prerequisites

- Get 3 or more nodes with Ubuntu installed.
- Get the SSH key with root access to the nodes.

> **NOTE:** The number of nodes must be odd.

### Clone the repository and install dependencies

```shell
git clone https://gitlab.anttek.io/devops/ceph-cluster-for-k8s.git ceph-cluster
cd ceph-cluster
python3 -m venv venv
```

### Activate the virtual environment

On Linux and macOS:
```shell
. venv/bin/activate
```

On Windows:
```shell
.\venv\Scripts\activate
```

### Install dependencies

```shell
pip install ansible
```

### Create the inventory file

```shell
cp -R example_cluster cluster
```

Now edit the inventory file and replace the example values with your own.

## Prepare nodes

Ansible will only does some preparations on the nodes.  
You will still need to initialize the cluster and join the nodes to it manually.

Task list:
- Update the system
- Set the hostnames
- Set the timezone
- Install the necessary packages like Docker, cephaadm, etc.

### Run the playbook

```shell
ansible-playbook playbook.yml -b
```

---

## Initialize the cluster

On the first node, run the following command:

```shell
# Replace `<NODE_IP>` with the IP address of the node.
cephadm bootstrap --mon-ip <NODE_IP> --initial-dashboard-user admin --initial-dashboard-password admin
```

Copy first node's SSH key to the other nodes:

```shell
ceph cephadm get-pub-key > ~/.ssh/id_rsa.pub
```

Add the other nodes to the cluster:

```shell
ceph orch host add node-2 <NODE_2_IP>
ceph orch host add node-3 <NODE_3_IP>
```

Add label for monitors:

```shell
ceph orch host label add node-1 mon
ceph orch host label add node-2 mon
ceph orch host label add node-3 mon
```

Apply the labels:

```shell
ceph orch apply mon node-1
ceph orch apply mon node-2
ceph orch apply mon node-3
```

Add labels for OSDs:

```shell
ceph orch host label add node-1 osd
ceph orch host label add node-2 osd
ceph orch host label add node-3 osd
```

After some minute you should see your raw disks by executing this command:

```shell
ceph orch device ls
```

Output example:

```shell
HOST    PATH      TYPE  DEVICE ID   SIZE  AVAILABLE  REFRESHED  REJECT REASONS                                                 
node-1  /dev/vdb  hdd              1048k  No         20m ago    Insufficient space (<5GB)                                      
node-1  /dev/vdc  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
node-2  /dev/vdb  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
node-3  /dev/vdb  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
```

Create the OSDs:

```shell
ceph orch daemon add osd node-1:<DEVICE>
ceph orch daemon add osd node-2:<DEVICE>
ceph orch daemon add osd node-3:<DEVICE>
```

> **NOTE:** Replace `<DEVICE>` with the device name, for example `/dev/vdb`.

---

## Create a pool for Kubernetes

```shell
ceph osd pool create k8s
rbd pool init k8s
```

Create a user for Kubernetes:

```shell
ceph auth get-or-create client.kube mon 'profile rbd' osd 'profile rbd pool=k8s' mgr 'profile rbd pool=k8s'
```

Find your monitors IP addresses:

```shell
ceph mon dump
```

Find your Ceph cluster ID:

```shell
ceph -s
```

---

## Install the Ceph CSI driver

1. Update the `csi-config-map.yaml` file with your cluster ID and monitors IP addresses.
2. Update the `csi-rbd-secret.yaml` file with the key you created for Kubernetes.
3. Update the `csi-rbd-sc.yaml` file with your cluster ID.

Now apply the manifests:

```shell
kubectl apply -f k8s
```

---

## Notes

### By default, you can't access Grafana charts from Ceph dashboard.

Run the following commands to fix it:

```shell
ceph dashboard set-grafana-api-url https://<NODE_IP>:3000 # Replace <NODE_IP> with the IP address of the node.
ceph dashboard set-grafana-api-ssl-verify False
```

### Device is not available when running `ceph orch device ls`.

If you see this error, you need to remove the LVM metadata from the device.

```shell
ceph orch device ls
```

Output example:

```shell
HOST    PATH      TYPE  DEVICE ID   SIZE  AVAILABLE  REFRESHED  REJECT REASONS
node-1  /dev/vdc  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
node-2  /dev/vdb  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
node-3  /dev/vdb  hdd              53.6G  No         20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
```

Login to the node, find the device name and remove the LVM metadata:

```shell
# On every node
dmsetup remove dmsetup info | grep Name | awk '{print $2}'
wipefs -a <DEVICE> # Replace <DEVICE> with the device name, for example /dev/vdc.
```

Now you should see the device in the list:

```shell
ceph orch device ls
```

Output example:

```shell
HOST    PATH      TYPE  DEVICE ID   SIZE  AVAILABLE  REFRESHED  REJECT REASONS
node-1  /dev/vdc  hdd              53.6G  Yes        20m ago
node-2  /dev/vdb  hdd              53.6G  Yes        20m ago
node-3  /dev/vdb  hdd              53.6G  Yes        20m ago
```

