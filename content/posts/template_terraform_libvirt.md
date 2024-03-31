+++
title = 'Template terraform libvirt'
date = 2024-03-04T17:19:34+05:00
tags = ["terraform", "libvirt", "debian", "alma", "template"]
+++

main.tf 
```tf
provider "libvirt" {
  uri = "qemu:///system"
  #uri = "qemu+ssh://root@ip/system"
}
```

providers.tf
```tf
terraform {
  required_providers {
    libvirt = {
      source  = "dmacvicar/libvirt"
    }
  }
}
```

network.tf 
```tf
resource "libvirt_network" "test_net" {
    name      = "networktest"
    mode      = "nat"
    domain    = "network_home.local"
    addresses = ["10.17.3.0/24"]
    dhcp {
        enabled = false
    }
    dns{
        enabled = true
      }
}
```

variables.tf 
```tf
resource "libvirt_volume" "debian_base" {
  name   = "debian_base"
  source = "https://cloud.debian.org/images/cloud/bookworm/daily/20240307-1679/debian-12-genericcloud-amd64-daily-20240307-1679.qcow2"
  format = "qcow2"
}

resource "libvirt_volume" "alma_base" {
  name   = "alma_base"
  source = "https://repo.almalinux.org/almalinux/8.9/cloud/x86_64/images/AlmaLinux-8-GenericCloud-8.9-20231128.x86_64.qcow2"
  format = "qcow2"
}

variable "debian_count" {
  type = number
  default = 1
}

variable "alma_count" {
  type = number
  default = 0
}
```

resourses.tf 
```tf
resource "libvirt_volume" "debian" {
  name           = "debian_tf_${count.index}.qcow2"
  base_volume_id = libvirt_volume.debian_base.id
  count          = var.debian_count
  pool = "default"
  size = 64424509440
}

resource "libvirt_volume" "alma" {
  name           = "alma_tf_${count.index}.qcow2"
  base_volume_id = libvirt_volume.alma_base.id
  count          = var.alma_count
  pool = "default"
  size = 64424509440
}
```

cloud_init.cfg 
```cfg
#cloud-config
ssh_pwauth: True

users:
  - name: q
    ssh-authorized-keys:
      - ${file("../keys/id_rsa.pub")}
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash

  - name: sa
    ssh-authorized-keys:
      - ${file("../sa_keys/sa_id_rsa.pub")}
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
```

cloudinit.tf 
```tf
resource "libvirt_cloudinit_disk" "commoninit" {
  name      = "commoninit.iso"
  user_data = data.template_file.user_data.rendered
}

data "template_file" "user_data" {
  template = file("${path.module}/cloud_init.cfg")
}
```

domain.tf 
```tf
resource "libvirt_domain" "domain_debian" {
    name = "debian_tf_${count.index}"
    vcpu = 1
    disk {
        volume_id = libvirt_volume.debian.*.id[count.index]
      }
    memory = "1024"
    cloudinit = libvirt_cloudinit_disk.commoninit.id
    network_interface {
      network_id = libvirt_network.test_net.id
      wait_for_lease = true
      hostname = "debian_${count.index}"
      addresses      = ["10.17.3.1${count.index+1}"] 
    }
    count = var.debian_count
  }

resource "libvirt_domain" "domain_alma" {
    name = "alma_tf_${count.index}"
    vcpu = 1
    disk {
        volume_id = libvirt_volume.alma.*.id[count.index]
      }
    memory = "2048"
    cloudinit = libvirt_cloudinit_disk.commoninit.id
    network_interface {
      network_id = libvirt_network.test_net.id
      wait_for_lease = true
      hostname = "alma_${count.index}"
      addresses      = ["10.17.3.2${count.index+1}"] 
    }
    count = var.alma_count
  }
```

output.tf 
```tf
output "debian_ip"{
  value = [for i in libvirt_domain.domain_debian: i.network_interface[0].addresses[0]]
}
output "alma_ip"{
  value = [for i in libvirt_domain.domain_alma: i.network_interface[0].addresses[0]]
}
```
