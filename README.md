# Migrating Virtual Machines from OpenStack to OpenShift Virtualization


## Table of Contents

<!-- TOC -->
  - [Introduction](#introduction)
  - [Environment](#environment)
  - [Prerequisites](#prerequisites)
  - [Part 1 - Download the image of a running VM in OpenStack](#part-1---download-the-image-of-a-running-vm-in-openstack)
  - [Part 2 - Create Storage Mapping](#part-2---create-storage-mapping)
  - [Part 3 - Create Network Mapping](#part-3---create-network-mapping)
  - [Part 4 - Start the migrated VM in OpenShift Virtualization](#part-4---start-the-migrated-vm-in-openshift-virtualization)
  - [Future Work](#future-work)
  - [References](#references)
<!-- TOC -->

## Introduction

[OpenShift Virtualization](https://docs.openshift.com/container-platform/4.8/virt/about-virt.html) is a feature of OpenShift that allows users to run and manage virtual machine workloads alongside container workloads on the same platform. It supports various virtualization tasks like creating new VMs, importing existing VMs from other platforms, managing network controllers and storage disks attached to VMs, etc.

A major attraction of OpenShift Virtualization was the feature to migrate existing VMs from other platforms like VMware vSphere and Red Hat Virtualization to OpenShift Virtualization using the [Migration Toolkit for Virtualization (MTV)](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/2.2/html/installing_and_using_the_migration_toolkit_for_virtualization/about-mtv) operator. But MTV operator does not support the migration of VMs (instances) from Red Hat OpenStack platform at the moment. This post will go through the steps to cold migrate a VM from Red Hat OpenStack platform to OpenShift Virtualization.

## Environment

* [Red Hat OpenStack Platform 16.2 All-In-One Installation](https://github.com/rh-telco-tigers/OSP16.2-AIO)
* [Red Hat OpenShift Container Platform 4.10 Single Node Installation](https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-installing-sno.html)
* JumpBox server to connect to OpenStack and OpenShift platforms. All the steps given below are performed in this jumpbox server.

The OpenStack environment has 2 networks that the VMs can connect to - an internal private network of type *Geneve* (Network100) and an external physical network of type *Flat* (ExternalNet).

![image info](./pictures/openstack-network.png)

Both these networks have to be mapped to equivalent networks in the OpenShift cluster. Hence, there are two network interfaces enabled on the worker node(s) of the OpenShift cluster. 
The OpenStack AIO host and the OpenShift cluster worker node(s) are on the same network subnet. 

## Prerequisites

* In the jumpbox, install -
    * [OpenStack command-line client](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html)
    * [OpenShift CLI](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html)

* Login to both OpenStack and OpenShift via the command-line from the jumpbox
* Install Container-native Virtualization (CNV) operator in the OpenShift cluster

## Part 1 - Download the image of a running VM in OpenStack 

For creating an image of a running VM (instance) in OpenStack, a new volume containing the VM's runtime ephemeral storage changes has to be created, which is then converted into an image that is downloaded. This way, a volume that is unattached is created, that can be used for creating a downloadable image. The detailed steps for the same using the OpenStack CLI commands are listed below - 

List VMs (instances) in OpenStack.

```
# openstack server list
+--------------------------------------+--------------------+--------+-------------------------------------------------------+--------------------------+--------+
| ID                                   | Name               | Status | Networks                                              | Image                    | Flavor |
+--------------------------------------+--------------------+--------+-------------------------------------------------------+--------------------------+--------+
| a478c7a7-15c5-4cdd-a536-3f9f6030c0e2 | Windows2012Server1 | ACTIVE | ExternalNet=192.168.11.82; Network100=192.168.100.114 | N/A (booted from volume) | Large  |
| 9e4e2e44-b54d-4ae8-9490-ff225305fdb1 | FedoraServer1      | ACTIVE | ExternalNet=192.168.11.84; Network100=192.168.100.102 | N/A (booted from volume) | Medium |
| 6f0113f1-afff-4bc4-a59e-768e4e1332a1 | CirrOSInstance1    | ACTIVE | Network100=192.168.100.110, 192.168.11.64             | N/A (booted from volume) | Tiny   |
+--------------------------------------+--------------------+--------+-------------------------------------------------------+--------------------------+--------+
```

From the list, get the name of the VM that is to be migrated. 

In this post, I will be migrating the VM *FedoraServer1*. There is a web page hosted on an httpd server on the VM that displays the server's IP address. 

![image info](./pictures/osp-website.png)

The VM would have to be stopped in order to cold migrate it. 

```
# openstack server stop FedoraServer1
# openstack server list
+--------------------------------------+--------------------+---------+-------------------------------------------------------+--------------------------+--------+
| ID                                   | Name               | Status  | Networks                                              | Image                    | Flavor |
+--------------------------------------+--------------------+---------+-------------------------------------------------------+--------------------------+--------+
| a478c7a7-15c5-4cdd-a536-3f9f6030c0e2 | Windows2012Server1 | ACTIVE  | ExternalNet=192.168.11.82; Network100=192.168.100.114 | N/A (booted from volume) | Large  |
| 9e4e2e44-b54d-4ae8-9490-ff225305fdb1 | FedoraServer1      | SHUTOFF | ExternalNet=192.168.11.84; Network100=192.168.100.102 | N/A (booted from volume) | Medium |
| 6f0113f1-afff-4bc4-a59e-768e4e1332a1 | CirrOSInstance1    | ACTIVE  | Network100=192.168.100.110, 192.168.11.64             | N/A (booted from volume) | Tiny   |
+--------------------------------------+--------------------+---------+-------------------------------------------------------+--------------------------+--------+
```

Since the VM is shut down, the web page is unresponsive on reloading it.

![image info](./pictures/osp-website-down.png)

Once the VM has been shut down, create a snapshot of the VM by specifying the VM name.

```
# openstack server image create --name fedora_img_snap FedoraServer1
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | d41d8cd98f00b204e9800998ecf8427e                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| container_format | bare                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| created_at       | 2022-07-31T21:29:10Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| disk_format      | qcow2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| file             | /v2/images/516a9eb3-2f9e-4082-a260-b7036557db7c/file                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| id               | 516a9eb3-2f9e-4082-a260-b7036557db7c                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| min_disk         | 15                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| min_ram          | 512                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| name             | fedora_img_snap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| owner            | 51238d9ffcd9416da13ff40dd6086292                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| properties       | architecture='x86_64', base_image_ref='', bdm_v2='True', block_device_mapping='[{"device_name": "/dev/vda", "source_type": "snapshot", "boot_index": 0, "guest_format": null, "volume_id": null, "destination_type": "volume", "no_device": null, "tag": null, "volume_size": 15, "disk_bus": "virtio", "volume_type": null, "delete_on_termination": false, "snapshot_id": "9f0c5fe6-d5a2-430d-ac7e-9059bfb73108", "device_type": "disk", "image_id": null}]', boot_roles='member,admin,reader', description='Fedora Cloud Image', direct_url='swift+config://ref1/glance/516a9eb3-2f9e-4082-a260-b7036557db7c', hw_machine_type='pc-i440fx-rhel7.6.0', os_hash_algo='sha512', os_hash_value='cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e', os_hidden='False', owner_project_name='admin', owner_user_name='admin', root_device_name='/dev/vda', stores='default_backend' |
| protected        | False                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| schema           | /v2/schemas/image                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| size             | 0                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| status           | active                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| tags             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| updated_at       | 2022-07-31T21:29:10Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| visibility       | private                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Get the ID of the snapshot.

```
# openstack volume snapshot list
+--------------------------------------+------------------------------+-------------+-----------+------+
| ID                                   | Name                         | Description | Status    | Size |
+--------------------------------------+------------------------------+-------------+-----------+------+
| f9db22c5-a490-438b-b1d9-a1ca7701d733 | snapshot for fedora_img_snap |             | available |   15 |
+--------------------------------------+------------------------------+-------------+-----------+------+
```

Create a volume from the snapshot by providing the snapshot ID, required size, and volume name.

```
# openstack volume create --snapshot f9db22c5-a490-438b-b1d9-a1ca7701d733 --size 15 --type tripleo --bootable fedora_migrate_vol
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | true                                 |
| consistencygroup_id | None                                 |
| created_at          | 2022-07-31T22:06:55.000000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 7580bd66-61f3-457e-8945-632b3ca450a6 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | fedora_migrate_vol                   |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 15                                   |
| snapshot_id         | f9db22c5-a490-438b-b1d9-a1ca7701d733 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | tripleo                              |
| updated_at          | 2022-07-31T22:06:55.000000           |
| user_id             | f8110b3150c84857b0ddf756afd0b137     |
+---------------------+--------------------------------------+
```

Wait until the volume is created. 

```
# openstack volume list
+--------------------------------------+--------------------+-----------+------+---------------------------------------------+
| ID                                   | Name               | Status    | Size | Attached to                                 |
+--------------------------------------+--------------------+-----------+------+---------------------------------------------+
| 7580bd66-61f3-457e-8945-632b3ca450a6 | fedora_migrate_vol | available |   15 |                                             |
| 9c927772-73de-4f56-a5bd-179b95d00c77 |                    | available |   10 |                                             |
| 7313a2d2-988c-4bb2-9799-f5b63582c70e |                    | in-use    |   25 | Attached to Windows2012Server1 on /dev/vda  |
| e321fdb8-ba6a-4b07-97ea-86119b5b35ad |                    | in-use    |   15 | Attached to FedoraServer1 on /dev/vda       |
| b4f6de2c-19bf-4b15-8e64-8e4da15e8417 |                    | in-use    |   10 | Attached to CirrOSInstance1 on /dev/vda     |
+--------------------------------------+--------------------+-----------+------+---------------------------------------------+
```

Using the volume ID, attach the newly created volume to an image by specifying the volume name and new image name.

**TODO - Check when the bug fix will be available in production (https://review.opendev.org/c/openstack/python-openstackclient/+/844268)**
```
# openstack image create --volume fedora_migrate_vol fedora_img_download
upload_to_image() got an unexpected keyword argument 'visibility'

Alternative command -

# cinder upload-to-image 7580bd66-61f3-457e-8945-632b3ca450a6 fedora_img_download
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| container_format    | bare                                 |
| disk_format         | raw                                  |
| display_description | None                                 |
| id                  | 7580bd66-61f3-457e-8945-632b3ca450a6 |
| image_id            | f61b244e-7d56-479d-b0e1-2cdb6c2d6196 |
| image_name          | fedora_img_download                  |
| protected           | False                                |
| size                | 15                                   |
| status              | uploading                            |
| updated_at          | 2022-07-31T22:06:56.000000           |
| visibility          | private                              |
| volume_type         | tripleo                              |
+---------------------+--------------------------------------+
```

Wait until the image is created and is in active state. You can check the status this way using the image name.

```
# openstack image show fedora_img_download
+------------------+--------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                  |
+------------------+--------------------------------------------------------------------------------------------------------+
| container_format | bare                                                                                                   |
| created_at       | 2022-07-31T22:33:23Z                                                                                   |
| disk_format      | raw                                                                                                    |
| file             | /v2/images/f61b244e-7d56-479d-b0e1-2cdb6c2d6196/file                                                   |
| id               | f61b244e-7d56-479d-b0e1-2cdb6c2d6196                                                                   |
| min_disk         | 0                                                                                                      |
| min_ram          | 0                                                                                                      |
| name             | fedora_img_download                                                                                    |
| owner            | 51238d9ffcd9416da13ff40dd6086292                                                                       |
| properties       | architecture='x86_64', description='Fedora Cloud Image', os_hidden='False', signature_verified='False' |
| protected        | False                                                                                                  |
| schema           | /v2/schemas/image                                                                                      |
| status           | saving                                                                                                 |
| tags             |                                                                                                        |
| updated_at       | 2022-07-31T22:33:24Z                                                                                   |
| visibility       | private                                                                                                |
+------------------+--------------------------------------------------------------------------------------------------------+
.
.
.
.
# openstack image show fedora_img_download
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                                                                                                                                                                                                                   |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | d46e68bad1a32825234d6faed3baeff3                                                                                                                                                                                                                                                                                                                                                        |
| container_format | bare                                                                                                                                                                                                                                                                                                                                                                                    |
| created_at       | 2022-07-31T22:33:23Z                                                                                                                                                                                                                                                                                                                                                                    |
| disk_format      | raw                                                                                                                                                                                                                                                                                                                                                                                     |
| file             | /v2/images/f61b244e-7d56-479d-b0e1-2cdb6c2d6196/file                                                                                                                                                                                                                                                                                                                                    |
| id               | f61b244e-7d56-479d-b0e1-2cdb6c2d6196                                                                                                                                                                                                                                                                                                                                                    |
| min_disk         | 0                                                                                                                                                                                                                                                                                                                                                                                       |
| min_ram          | 0                                                                                                                                                                                                                                                                                                                                                                                       |
| name             | fedora_img_download                                                                                                                                                                                                                                                                                                                                                                     |
| owner            | 51238d9ffcd9416da13ff40dd6086292                                                                                                                                                                                                                                                                                                                                                        |
| properties       | architecture='x86_64', description='Fedora Cloud Image', direct_url='swift+config://ref1/glance/f61b244e-7d56-479d-b0e1-2cdb6c2d6196', os_hash_algo='sha512', os_hash_value='8748e7fed4550c813c8159b00ce529d374157f680d85a8041c1ae2434d8bd05e538ea086afd741e56831b26565fd52f2c104fda50bf9647a3ad52bd24722e609', os_hidden='False', signature_verified='False', stores='default_backend' |
| protected        | False                                                                                                                                                                                                                                                                                                                                                                                   |
| schema           | /v2/schemas/image                                                                                                                                                                                                                                                                                                                                                                       |
| size             | 16106127360                                                                                                                                                                                                                                                                                                                                                                             |
| status           | active                                                                                                                                                                                                                                                                                                                                                                                  |
| tags             |                                                                                                                                                                                                                                                                                                                                                                                         |
| updated_at       | 2022-07-31T22:35:54Z                                                                                                                                                                                                                                                                                                                                                                    |
| visibility       | private                                                                                                                                                                                                                                                                                                                                                                                 |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Download the volume image created using the image ID.

```
# openstack image save --file fedora_img_download.raw f61b244e-7d56-479d-b0e1-2cdb6c2d6196
```

Convert the image from raw format to qcow2 format using [QEMU disk image utility](https://www.qemu.org/download/#linux). This conversion is required because images can be uploaded to OpenShift only if they are in *ISO*, *IMG* or *QCOW2* format.

```
# qemu-img convert -f raw -O qcow2 fedora_img_download.raw fedora_img_download.qcow2
```

The QEMU disk image utility can be used to get the expected virtual size of the image.

```
# qemu-img info fedora_img_download.qcow2 
image: fedora_img_download.qcow2
file format: qcow2
virtual size: 15 GiB (16106127360 bytes)
disk size: 1.96 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
```

While migrating a VM from a source to destination host, VM's storage and network configurations have to be created at the destination host, which are equivalent to the ones the VM had in the source host. Now that the VM's image has been downloaded from the source host, equivalent storage mapping can be created.

## Part 2 - Create Storage Mapping

In case of OpenStack to OpenShift VM migration, equivalent storage configuration for the VM is created in the OpenShift cluster by creating a PVC with the required image size for the VM to bind to.

OpenShift Virtualization uses [Containerized Data Importer (CDI)](https://docs.openshift.com/container-platform/4.7/virt/virtual_machines/virtual_disks/virt-uploading-local-disk-images-virtctl.html) to import a VM image into a persistent volume claim (PVC) by using a data volume.

Create a *DataVolume* object with *upload* data source to use for uploading the local VM image.

```
cat << EOF | oc create -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  annotations:
    cdi.kubevirt.io/storage.bind.immediate.requested: "true"
    kubevirt.ui/provider: local-provider
  name: fedora-dv
  namespace: migration
spec:
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 15Gi 
    storageClassName: lvstore
    volumeMode: Filesystem
  source:
    upload: {}
EOF
```

Make sure to change the ```specs```attributes as per the underlying persistent storage setup of your OpenShift cluster. In this post, the OpenShift cluster has been setup on a single node that uses local volumes for persistent storage. Also change the ```name``` and ```namespace``` attribute as required. 

While uploading images using CDI, its SSL certificates must be trusted by the browser. If a trusted CA is used for the OpenShift cluster certificates, this will be sufficient. If not, the cdi-uploadproxy-openshift-cnv application has to be browsed under the OpenShift cluster's ingress URL to accept its SSL certificate. Any 404 errors may be ignored during this process. The URL can be obtained using the following command.

```
oc whoami --show-console | sed 's/console-openshift-console/cdi-uploadproxy-openshift-cnv/'
```

The qcow2 image can be uploaded from jumpbox's local storage to OpenShift Virtualization using the [virtctl client](https://docs.openshift.com/container-platform/4.8/virt/install/virt-installing-virtctl.html) that specifies the parameters defined above.

```
# virtctl image-upload dv fedora-dv --size=15Gi --image-path=/root/fedora_img_download.qcow2  --insecure
Using existing PVC migration/fedora-dv
Uploading data to https://cdi-uploadproxy-openshift-cnv.apps.ocpd.example.com

 1.96 GiB / 1.96 GiB [=========================================================================================================================================================================] 100.00% 23s

Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
Processing completed successfully
Uploading /root/fedora_img_download.qcow2 completed successfully
```

Wait until the upload is completed. Ensure that the datavolume upload succeeded and the PVC was created successfully. 

```
# oc get dv -n migration
NAME        PHASE       PROGRESS   RESTARTS   AGE
fedora-dv   Succeeded   N/A                   2m26s
```

```
oc get pvc -n migration
NAME        STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
fedora-dv   Bound    local-pv-64fe52ca   372Gi      RWO            lvstore        3m28s
```

With this the required storage mapping for the VM is created. Now the network mapping can be created.

## Part 3 - Create Network Mapping


As the OpenStack environment had two networks in it, the two network interfaces enabled on the worker node(s) of the OpenShift cluster will be used for creating a network mapping. One thing to be noted in case of network mapping is that the network configurations of the VM like IP address, MAC addresses, etc. are preserved during migration. 

The *Network100* private network in OpenStack can be mapped in 2 ways - 
* It can be mapped to the default pod network of OpenShift cluster, in which case the IP address of the VM associated with this network will change as the CIDR ranges are different for the *Network100* private network and pod network. If the internal communication between VMs connected through this network are IP address independent, this mapping can be used. 
* It can be mapped to a new internal network by creating a bridge interface that is attached to a third network interface on the OpenShift worker node(s). The *Network100* private network IP addresses of the VM can be retained in this case.

The *ExternalNet* external network in OpenStack can be mapped to an equivalent network configuration in the following way -

Create a bridge interface which attaches to the second network interface enabled on the OpenShift cluster worker node(s) by creating a *Node Network Configuration Policy*.

```
cat << EOF | oc create -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: eno2-br-nncp
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with eno2 as a port
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
          dhcp: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eno2
EOF
```

Change the name of the second network interface in the ```port-name``` attribute if its not *eno2*.

The VMs in a specific namespace in the OpenShift Virtualization cluster can connect to additional networks using a custom resource that exposes layer-2 devices called *Network Attachment Definition*.

```
cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: tuning-bridge
  namespace: migration
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "tuning-bridge",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br1"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF
```

**TODO : EDIT TO MAKE IT CLEARER - As the OpenStack AIO host and the OpenShift cluster worker node(s) are on the same network subnet, the interface connected to *ExternalNet* external network and the secondary interface the bridge interface connects to are on the same subnet. Hence the VM's *ExternalNet*
network IP address in OpenStack can be retained in OpenShift Virtualization as well.** 

With this the required network configuration mapping for the VM is created. Now the migrated VM can be started by associating it with the created storage and network mappings.

*Note*: In case you want to follow the second method for mapping OpenStack's internal network *Network100*, you need to create another bridge interface and make it available in the *migration* namespace using another *Node Network Configuration Policy* and *Network Attachment Definition* combo.

## Part 4 - Start the migrated VM in OpenShift Virtualization

The VM can be created using the ```feodra-vm.yml``` file.

```
# oc apply -f fedora-vm.yml
```

To create a VM from the clone of the image loaded into the PVC created earlier, give the parameters of the PVC in the ```dataVolumeTemplates``` attribute. 

The current YAML creates a VM that maps the *Network100* private network in OpenStack to the pod network in OpenShift. 
To connect the VM to the secondary bridge interface, add the parameters of the *Network Attachment Definition* in the ```networks``` attribute. You can specify the static MAC address for the interface(s) in the ```interfaces``` attribute. The values given in the YAML are the ones the *FedoraServer1* VM had in OpenStack. You can also specify the static IP address that the VM's secondary interface is associated with in the ```cloudInitNoCloud-networkData```. This IP address value is again the one the *FedoraServer1* VM had in OpenStack. 

Additional configurations like adding ssh key-pairs, restarting NetworkManager, running certain commands on boot, etc. can be added in the ```cloudInitNoCloud-userData``` attribute. 

Start the VM.

```
# virtctl start fedoraserver1 -n migration
VM fedoraserver1 was scheduled to start
```

```
# oc get vm -n migration
NAME            AGE   STATUS    READY
fedoraserver1   29m   Running   True
```

Once the VM is up and running, reload the web page hosted in the httpd server of the VM, and the web page would display the server's IP address, which remains the same after migration.

![image info](./pictures/ocp-website.png)


In this way we can migrate a VM from OpenStack to OpenShift virtualization keeping the workloads in the VM intact. 


## Future Work

* Perform warm migration of VMs between OpenStack and OpenShift Virtualization using [Change Block Tracking](https://wiki.openstack.org/wiki/Raksha).
* **TODO - make it clearer** : Run an Ansible playbook post migration to configure network settings required by the VM

## References 

* [Creating image of a volume in OpenStack](https://access.redhat.com/solutions/2885001)
* [Convert format of an image](https://docs.openstack.org/image-guide/convert-images.html)
* [Creating a VM in OpenShift Virtualization via CLI](https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/virt-create-vms.html)
* [Create VM with a static IP address in OpenShift Virtualization](https://docs.openshift.com/container-platform/4.6/virt/virtual_machines/vm_networking/virt-configuring-ip-for-vms.html)
* [Migrating VMs from vSphere to OpenShift CNV using MTV](https://github.com/latouchek/mtv-vsphere)
* [Migrating VMs from Red Hat Virtualization to OpenShift CNV using MTV](https://www.youtube.com/watch?v=LWQazV4eVTo)