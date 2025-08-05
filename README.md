# Build and Automate a Backup and Storage Server on Oracle Linux
This homelab project sets up a backup and storage server using Oracle Linux on VirtualBox. It features LVM, NFS, Restic, Netdata for monitoring, and Ansible for automation. It is optimized for low-resource environments, ideal for single-laptop homelabs and virtual setups.

1. [Virtual Machine Setup and Initial Configuration](#virtual-machine-setup-and-initial-configuration)
2. [Setup Oracle Storage and Backup Server](#setup-oracle-storage-and-backup-server)
3. [Setup Lubuntu NFS Client and Backup Target](#setup-lubuntu-nfs-client-and-backup-target)
4. [Setup Alpine Linux Ansible and Administration Tools](#setup-alpine-linux-ansible-and-administration-tools)
5. [Automating Backups Using Ansible](#automating-backups-using-ansible)
6. [Monitoring on Oracle Linux Using Cockpit](#monitoring-on-oracle-linux-using-cockpit)
7. [Version Control with Git on Alpine Linux](#version-control-with-git-on-alpine-linux)


## Virtual Machine Setup and Initial Configuration
- Visit [https://yum.oracle.com/oracle-linux-isos.html](https://yum.oracle.com/oracle-linux-isos.html) and download the Oracle Linux ISO image. In this homelab project, the full OracleLinux-R10-U0-x86_64-dvd.iso is downloaded <br />
  <img width="1209" height="851" alt="image" src="https://github.com/user-attachments/assets/1a4dc56f-34c0-41bb-aa5c-0d966ecec745" />

- In VirtualBox, create a new VM and add the downloaded ISO image <br />
  <img width="792" height="466" alt="image" src="https://github.com/user-attachments/assets/c438f4c8-2daf-4fa1-933e-3266e6c99f81" />

- Set 4GB of RAM and 2 CPUs <br />
  <img width="790" height="470" alt="image" src="https://github.com/user-attachments/assets/63502f91-9e82-4ea0-87a8-925d30a4e060" />

- 20GB of storage space is used <br />
  <img width="791" height="469" alt="image" src="https://github.com/user-attachments/assets/49c320b8-3c7e-40f8-bb68-04589cfb36d9" />

- Since Lubuntu and Alpine Linux VMs are already setup previously, their setup steps are not included
- On all the three VMs, set static IPs or define `/etc/hosts` mappings
  ```
  192.168.56.10 oracle1
  192.168.56.11 client1
  192.168.56.12 admin1
  ```

- Then, set hostnames using
  ```
  hostnamectl set-hostname <hostname>
  ```




## Setup Oracle Storage and Backup Server
- On Oracle Linux VM, install LVM, NFS, Restic, Netdata
  ``` 
  dnf update -y
  dnf install -y lvm2 nfs-utils restic xfsprogs
  ```
- Setup the LVM storage
  ```
  pvcreate /dev/sdb /dev/sdc
  vgcreate storage_vg /dev/sdb /dev/sdc
  lvcreate -L 18G -n storage_lv storage_vg
  mkfs.xfs /dev/storage_vg/storage_lv
  mkdir /mnt/storage
  mount /dev/storage_vg/storage_lv /mnt/storage
  echo '/mnt/storage *(rw,sync,no_root_squash)' >> /etc/exports
  systemctl enable --now nfs-server
  exportfs -rav
  ```

- Setup Restic backup repo
  ```
  mkdir /mnt/storage/restic_repo
  export RESTIC_REPOSITORY=/mnt/storage/restic_repo
  restic init
  ```




## Setup Lubuntu NFS Client and Backup Target
- On the Lubuntu VM, install NFS client and simulate data
  ```
  sudo apt update
  sudo apt install -y nfs-common
  sudo mkdir /mnt/nfs
  sudo mount oracle1:/mnt/storage /mnt/nfs
  ```

- Add to `/etc/fstab` for auto-mount
  ```
  oracle1:/mnt/storage /mnt/nfs nfs defaults 0 0
  ```

- Simulate data to backup
  ```
  mkdir -p ~/data/testfiles
  echo "dummy" > ~/data/testfiles/sample.txt
  ```



## Setup Alpine Linux Ansible and Administration Tools
- On the Alpine Linux VM, setup Ansible and admin tools
  ```
  apk update
  apk add ansible openssh git python3 py3-pip
  ```

- Create the Ansible inventory
  ```
  [storage]
  oracle1 ansible_host=192.168.56.10 ansible_user=root
  
  [clients]
  client1 ansible_host=192.168.56.11 ansible_user=ubuntu
  ```

- Generate the SSH key and copy to Oracle Linux VM and Lubuntu Client VM
  ```
  ssh-keygen
  ssh-copy-id root@oracle1
  ssh-copy-id ubuntu@client1
  ```

- Then test Ansible
  ```
  ansible all -m ping
  ```



## Automating Backups Using Ansible
- Write a playbook to automate setups using Ansible
  ```
  # backup.yml
  - hosts: storage
    tasks:
      - name: Run Restic to back up remote client
        command: >
          restic -r /mnt/storage/restic_repo backup --host client1 --tag daily ssh://ubuntu@client1/home/ubuntu/data
  ```

- Run the playbook
  ```
  ansible-playbook backup.yml
  ```



## Version Control with Git on Alpine Linux
- On Alpine Linux, use Git for version control
  ```
  mkdir -p ~/homelab-configs
  cd ~/homelab-configs
  git init
  cp ~/ansible/*.yml .
  git add .
  git commit -m "Initial commit: Ansible + backup playbook"
  ```

- Push to GitHub when ready






