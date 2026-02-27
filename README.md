h3po.proxmox_cloudimage
=========

This ansible role downloads for example a debian cloud image from https://cloud.debian.org and creates a proxmox template vm from it. It is meant to be run on the proxmox node that is going to store the vm; although we can do most things through the proxmox api it is currently not possible to attach the downloaded image to a vm without the local `qm importisk` command.

Role Variables
--------------

| variable | default value | description/alternatives
| --- | --- | --- |
| proxmox_template_pool | - | target pool for the template vm
| proxmox_template_storage | local | a storage pool that allows qcow2/raw image files
| proxmox_cloudimage_distro | debian | ubuntu, rocky |
| proxmox_cloudimage_repo_subdir | bullseye | bullseye/daily, buster, buster/daily, ...
| proxmox_cloudimage_release | latest | a specific version such as "20220328-962". the debian daily repository doesn't keep a very long history
| proxmox_cloudimage_downloaddir | /tmp | directory to store the downloaded file
| proxmox_cloudimage_keep | false | keep the file to avoid repeated downloads
| proxmox_cloudimage_qemuagent | false | install qemu-guest-agent in the image. needs libguestfs-tools installed on the proxmox node
| proxmox_cloudimage_diskparams | cache=writeback,discard=on | optimal settings for thin provisioned zfs

Example Playbook
----------------

The role sets some facts that are useful for using the template afterwards
* proxmox_cloudimage_template_name
* proxmox_cloudimage_template_vmid
* proxmox_cloudimage_template_created

This runs in the context of the host `debian-bullseye-daily' that is yet to be created. We're using delegate_to for the pve host, which needs to be defined in the inventory. 

```yaml
---
- name: create a debian vm
  hosts: debian-bullseye-daily
  become: true
  gather_facts: false
  vars:
    # add the template to the "template" pool
    proxmox_template_pool: template
    # the storage needs to support .qcow2 files, we'll migrate the disk to zfs_volumes afterwards
    proxmox_template_storage: zfs_files
    # download the daily image instead of the default stable release
    proxmox_cloudimage_repo_subdir: bullseye/daily
    # create the template on this host from the inventory
    proxmox_node: pve
  tasks:

    - name: create the debian template
      import_role:
        name: h3po.proxmox_cloudimage
      delegate_to: "{{ proxmox_node }}"
      become: true

    - name: move the template to zfs storage
      command: "qm move-disk {{ proxmox_cloudimage_template_vmid }} scsi0 zfs_volumes -delete 1"
      delegate_to: "{{ proxmox_node }}"
      become: true
      when: proxmox_cloudimage_template_created
  
    - name: now clone the vm some way or the other...
      debug:
        msg: "the template vm has id {{ proxmox_cloudimage_template_vmid }}"
      when: proxmox_cloudimage_template_existed or proxmox_cloudimage_template_created
```

License
-------

[GNU General Public License v3.0](LICENSE)
