# Migrating Virtual Machines from OpenStack to OpenShift Virtualization


## Table of Contents

<!-- TOC -->
  - [Introduction](#introduction)
  - [Environment](#environment)
  - [Prerequisites](#prerequisites)
  - [Your first VM](#your-first-vm)
    - [Using the OpenShift UI](#using-the-openshift-ui)
  - [Exposing a service from a VM](#exposing-a-service-from-a-vm)
    - [VMIRS and Liveness Probes](#vmirs-and-liveness-probes)
    - [Cleanup](#cleanup)
  - [Making a persistent Fedora VM](#making-a-persistent-fedora-vm)
    - [Migrating a vm from one host to another](#migrating-a-vm-from-one-host-to-another)
    - [Cleanup](#cleanup-1)
  - [Cloning a VM](#cloning-a-vm)
    - [Cleanup](#cleanup-2)
  - [Deploying a Windows VM from ISO](#deploying-a-windows-vm-from-iso)
    - [Create new VM](#create-new-vm)
    - [Accessing a Windows VM directly via RDP](#accessing-a-windows-vm-directly-via-rdp)
  - [Importing a VM from vSphere](#importing-a-vm-from-vsphere)
  - [Importing a VMDK](#importing-a-vmdk)
    - [Cleanup](#cleanup-3)
  - [Advanced networking examples](#advanced-networking-examples)
<!-- TOC -->


## Introduction

OpenShift Virtualization is a feature of OpenShift that allows users to run and manage virtual machine workloads alongside container workloads on the same platform. It supports various virtualization tasks like creating new VMs, importing existing VMs from other platforms, managing network controllers and storage disks attached to VMs, etc. [[1]](https://docs.openshift.com/container-platform/4.8/virt/about-virt.html)

A major attraction of OpenShift Virtualization was the feature to migrate existing VMs from other platforms like VMware vSphere and Red Hat Virtualization to OpenShift Virtualization using the [Migration Toolkit for Virtualization (MTV)](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/2.2/html/installing_and_using_the_migration_toolkit_for_virtualization/about-mtv) operator. But MTV operator does not support the migration of VMs (instances) from Red Hat OpenStack platform at the moment. This post will go through the steps to cold migrate a VM from Red Hat OpenStack platform to OpenShift Virtualization.

## Environment

* [Red Hat OpenStack Platform 16.2 All-In-One Installation](https://github.com/rh-telco-tigers/OSP16.2-AIO)
* [Red Hat OpenShift Container Platform 4.10 Single Node Installation](https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-installing-sno.html)
* JumpBox server to connect to OpenStack and OpenShift platforms 

There are two networks associated with OpenStack - an internal private network of type Geneve (Network100) and an external physical network of type Flat (ExternalNet).

![image info](./pictures/openstack-network.png)

Both these networks have to be mapped to equivalent networks in the OpenShift cluster. Hence, there are two network interfaces enabled on the worker nodes of the OpenShift cluster.

## Prerequisites
In the jumpbox, install -
 * [OpenStack command-line client](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html)
 * [OpenShift CLI](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html)

Login to both OpenStack and OpenShift via the command-line.

## Part 1 - Download the image of a running VM in OpenStack 

List VMs (instances) in OpenStack.

```
# nova list
+--------------------------------------+--------------------+--------+------------+-------------+-------------------------------------------------------+
| ID                                   | Name               | Status | Task State | Power State | Networks                                              |
+--------------------------------------+--------------------+--------+------------+-------------+-------------------------------------------------------+
| 6f0113f1-afff-4bc4-a59e-768e4e1332a1 | CirrOSInstance1    | ACTIVE | -          | Running     | Network100=192.168.100.110, 192.168.11.64             |
| 9e4e2e44-b54d-4ae8-9490-ff225305fdb1 | FedoraServer1      | ACTIVE | -          | Running     | ExternalNet=192.168.11.84; Network100=192.168.100.102 |
| a478c7a7-15c5-4cdd-a536-3f9f6030c0e2 | Windows2012Server1 | ACTIVE | -          | Running     | ExternalNet=192.168.11.82; Network100=192.168.100.114 |
+--------------------------------------+--------------------+--------+------------+-------------+-------------------------------------------------------+
```

From the list, get the name of the VM you want to migrate. 

In this post, I will be migrating the VM 'FedoraServer1'. There is a simple website running on an httpd server on the VM that displays the server's IP address. 

![image info](./pictures/osp-website.png)

The VM would have to be stopped in order to cold migrate it. 

```
# nova stop FedoraServer1
Request to stop server FedoraServer1 has been accepted.
```

Once the VM has been shut down, create a snapshot of the VM

```
# nova image-create --poll FedoraServer1 fedora_img_snap

Server snapshotting... 100% complete
Finished
```

Get the snapshot ID

```
# cinder snapshot-list
+--------------------------------------+--------------------------------------+-----------+------------------------------+------+----------------------------------+
| ID                                   | Volume ID                            | Status    | Name                         | Size | User ID                          |
+--------------------------------------+--------------------------------------+-----------+------------------------------+------+----------------------------------+
| f9db22c5-a490-438b-b1d9-a1ca7701d733 | e321fdb8-ba6a-4b07-97ea-86119b5b35ad | available | snapshot for fedora_img_snap | 15   | f8110b3150c84857b0ddf756afd0b137 |
+--------------------------------------+--------------------------------------+-----------+------------------------------+------+----------------------------------+
```

Create a volume from the snapshot by mentioning the snapshot ID and required size

```
# cinder create --snapshot-id f9db22c5-a490-438b-b1d9-a1ca7701d733 15
WARNING:cinderclient.shell:API version 3.68 requested, 
WARNING:cinderclient.shell:downgrading to 3.59 based on server support.
+--------------------------------+---------------------------------------+
| Property                       | Value                                 |
+--------------------------------+---------------------------------------+
| attachments                    | []                                    |
| availability_zone              | nova                                  |
| bootable                       | true                                  |
| consistencygroup_id            | None                                  |
| created_at                     | 2022-07-28T19:59:20.000000            |
| description                    | None                                  |
| encrypted                      | False                                 |
| group_id                       | None                                  |
| id                             | ae640e30-e2ad-4513-bd69-35946647b98a  |
| metadata                       | {}                                    |
| migration_status               | None                                  |
| multiattach                    | False                                 |
| name                           | None                                  |
| os-vol-host-attr:host          | hostgroup@tripleo_iscsi#tripleo_iscsi |
| os-vol-mig-status-attr:migstat | None                                  |
| os-vol-mig-status-attr:name_id | None                                  |
| os-vol-tenant-attr:tenant_id   | 51238d9ffcd9416da13ff40dd6086292      |
| provider_id                    | None                                  |
| replication_status             | None                                  |
| service_uuid                   | None                                  |
| shared_targets                 | True                                  |
| size                           | 15                                    |
| snapshot_id                    | f9db22c5-a490-438b-b1d9-a1ca7701d733  |
| source_volid                   | None                                  |
| status                         | creating                              |
| updated_at                     | 2022-07-28T19:59:21.000000            |
| user_id                        | f8110b3150c84857b0ddf756afd0b137      |
| volume_type                    | tripleo                               |
+--------------------------------+---------------------------------------+
```

Wait until the volume is created 

```
# openstack volume list
+--------------------------------------+------+-----------+------+---------------------------------------------+
| ID                                   | Name | Status    | Size | Attached to                                 |
+--------------------------------------+------+-----------+------+---------------------------------------------+
| ae640e30-e2ad-4513-bd69-35946647b98a | None | available |   15 |                                             |
| 7313a2d2-988c-4bb2-9799-f5b63582c70e |      | in-use    |   25 | Attached to Windows2012Server1 on /dev/vda  |
| e321fdb8-ba6a-4b07-97ea-86119b5b35ad |      | in-use    |   15 | Attached to FedoraServer1 on /dev/vda       |
| b4f6de2c-19bf-4b15-8e64-8e4da15e8417 |      | in-use    |   10 | Attached to CirrOSInstance1 on /dev/vda     |
+--------------------------------------+------+-----------+------+---------------------------------------------+
```

Attach the newly created volume to a new image that will be downloaded, using the volume ID

```
# cinder upload-to-image ae640e30-e2ad-4513-bd69-35946647b98a fedora_img_download
WARNING:cinderclient.shell:API version 3.68 requested, 
WARNING:cinderclient.shell:downgrading to 3.59 based on server support.
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| container_format    | bare                                 |
| disk_format         | raw                                  |
| display_description | None                                 |
| id                  | ae640e30-e2ad-4513-bd69-35946647b98a |
| image_id            | 0e953af4-093d-4dfc-a24d-a4959e7e816a |
| image_name          | fedora_img_download                   |
| protected           | False                                |
| size                | 15                                   |
| status              | uploading                            |
| updated_at          | 2022-07-28T19:59:21.000000           |
| visibility          | private                              |
| volume_type         | tripleo                              |
+---------------------+--------------------------------------+
```

Wait until the image is created and is in active state. You can check the status this way

```
# openstack image show fedora_img_download
+------------------+--------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                  |
+------------------+--------------------------------------------------------------------------------------------------------+
| container_format | bare                                                                                                   |
| created_at       | 2022-07-28T20:03:19Z                                                                                   |
| disk_format      | raw                                                                                                    |
| file             | /v2/images/0e953af4-093d-4dfc-a24d-a4959e7e816a/file                                                   |
| id               | 0e953af4-093d-4dfc-a24d-a4959e7e816a                                                                   |
| min_disk         | 0                                                                                                      |
| min_ram          | 0                                                                                                      |
| name             | fedora_img_download                                                                                     |
| owner            | 51238d9ffcd9416da13ff40dd6086292                                                                       |
| properties       | architecture='x86_64', description='Fedora Cloud Image', os_hidden='False', signature_verified='False' |
| protected        | False                                                                                                  |
| schema           | /v2/schemas/image                                                                                      |
| status           | saving                                                                                                 |
| tags             |                                                                                                        |
| updated_at       | 2022-07-28T20:03:20Z                                                                                   |
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
| created_at       | 2022-07-28T20:07:23Z                                                                                                                                                                                                                                                                                                                                                                    |
| disk_format      | raw                                                                                                                                                                                                                                                                                                                                                                                     |
| file             | /v2/images/070e9766-7a78-47dc-a02d-70531496624a/file                                                                                                                                                                                                                                                                                                                                    |
| id               | 070e9766-7a78-47dc-a02d-70531496624a                                                                                                                                                                                                                                                                                                                                                    |
| min_disk         | 0                                                                                                                                                                                                                                                                                                                                                                                       |
| min_ram          | 0                                                                                                                                                                                                                                                                                                                                                                                       |
| name             | fedora_img_download                                                                                                                                                                                                                                                                                                                                                                     |
| owner            | 51238d9ffcd9416da13ff40dd6086292                                                                                                                                                                                                                                                                                                                                                        |
| properties       | architecture='x86_64', description='Fedora Cloud Image', direct_url='swift+config://ref1/glance/070e9766-7a78-47dc-a02d-70531496624a', os_hash_algo='sha512', os_hash_value='8748e7fed4550c813c8159b00ce529d374157f680d85a8041c1ae2434d8bd05e538ea086afd741e56831b26565fd52f2c104fda50bf9647a3ad52bd24722e609', os_hidden='False', signature_verified='False', stores='default_backend' |
| protected        | False                                                                                                                                                                                                                                                                                                                                                                                   |
| schema           | /v2/schemas/image                                                                                                                                                                                                                                                                                                                                                                       |
| size             | 16106127360                                                                                                                                                                                                                                                                                                                                                                             |
| status           | active                                                                                                                                                                                                                                                                                                                                                                                  |
| tags             |                                                                                                                                                                                                                                                                                                                                                                                         |
| updated_at       | 2022-07-28T20:09:56Z                                                                                                                                                                                                                                                                                                                                                                    |
| visibility       | private                                                                                                                                                                                                                                                                                                                                                                                 |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Download the volume image created using the image ID

```
# glance image-download --file fedora_img_download.raw 070e9766-7a78-47dc-a02d-70531496624a
```

Convert the image from raw format to qcow2 format

```
# qemu-img convert -f raw -O qcow2 fedora_img_download.raw fedora_img_download.qcow2
```

Ensure that the qcow2 image was created correctly

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



