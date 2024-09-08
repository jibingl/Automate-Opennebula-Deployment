# Step-by-step Deployment
Prepare your environment by referencing to below table.
Server Roles | OS | OS Admin | Unit | IP | Notes |
-------------|----|------|----|----------|-------|
Management server | CentOS 9 | `admin` | 1 | DHCP | Running Ansible |
Opennebula front-end server | Ubuntu 22.04 | `sysadmin` | 1 | 172.31.11.100 | FQDN: one.felixliang.local <br> Admin account: `oneadmin`/`ciscopass` |
KVM Host servers | Ubuntu 22.04 | `sysadmin` | 2 | 172.31.11.101 <br> 172.31.11.102 | Hypervisor hosts |
NFS server | Ubuntu 22.04 | `sysadmin` | 1 | 172.31.11.200 | Network storage |
> 1. Set the same password for the `sysadmin` users of the front-end, KVM, and NFS servers. 

## P1 - Configuring the Front-end Server
The front-end server is for running the Opennebula all front-end services, https Apache proxy service, and a MySQL database.
1. Log into the management server with the `sysadmin` account.
2. Configure TLS certificates. Generate a self-signed certificate for the server.
   ```
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout "/etc/ssl/private/one.felixliang.local.key" -out "/etc/ssl/certs/one.felixliang.local.crt"
   ```
3. As prompted, input all required information. Use **one.felixliang.local** as the FQDN.

## P2 - Configuring the KVM Hosts
The two KVM servers are for running KVM hypervisor.
1. Log into the KVM servers with the `sysadmin` account.
2. Make sure those servers have the same name of their own network adpter. In our case, the name is **ens33**. 

## P3 - Configuring the NFS server
The NFS server is for shared network storage.
1. Login to the NFS server as `sysadmin`.
2. Install NFS service in the NFS server.
   ```
   apt update
   apt install nfs-kernel-server
   systemctl start nfs-kernel-server.service  
   ```
3. Create a directory **/storage/one_datastores** and set its ownership to be 9869 as this is the UID/GID assigned to the oneadmin account during the Opennebula installation.
   ```
   mkdir -p /storage/one_datastores
   chmod 9869 9869 /storage/one_datastores
   ```
4. Configure the **one_datestores** directory to be exported by adding them to the **/etc/exports** file.
   ```
   # /etc/exports
   #
   # See exports(5) for more information.
   #
   # Use exportfs -r to reread
   # /export	192.168.1.10(rw,no_root_squash)
   /storage/one_datastores 172.31.11.0/24(rw,async)
   ```
5. Apply the new config.
   ```
   exportfs -a
   ```

## P4 - Configuring the Management Server
The management server is for running Ansible.  
1. Log into the management server with the `admin` account.
2. Configure static IP addresses schema.
3. Generate a SSH key pairs for the `admin` user. 
   ```
   #Execute the command without sudo
   ssh-keygen
   ```
4. Copy the SSH public key into the front-end, KVM, and NFS servers to setup passwordless SSH for the management server. 
5. Install Ansible and clone deploy scripts from the **one-deploy** git repository.
    ```
    sudo apt install python3-pip
    sudo pip3 install 'ansible-core<2.16'
    git clone https://github.com/OpenNebula/one-deploy.git
    cd ./one-deploy/ && make requirements
    ```

## P5 - Running Scripts/Playbook to Deploy OpenNebula Services
1. Log into the management server as `admin`.
2. Open relative yaml file (**shared.yml**).
   ```
   cd one-deploy
   nano inventory/shared.yml
   ```
3. Configure the (**shared.yml**) to meet our invironment.
   ```
   ---
   all:
     vars:
       ansible_user: sysadmin
       one_version: '6.10'
       one_pass: ciscopass
       ds:
         mode: shared
       vn:
         admin_net:
           managed: true
           template:
             VN_MAD: bridge
             PHYDEV: ens33
             BRIDGE: br0
             AR:
               TYPE: IP4
               IP: 192.168.0.50
               SIZE: 100
             NETWORK_ADDRESS: 192.168.0.0
             NETWORK_MASK: 255.255.255.0
             GATEWAY: 192.168.0.1
             DNS: 1.1.1.1
       # Mount NFS share.
       fstab:
         - src: 172.31.11.200:/storage/one_datastores

       features:
         passenger: true
       apache2_http:
         managed: false
       apache2_https:
         managed: true
         key: /etc/ssl/private/one.felixliang.local.key
         certchain: /etc/ssl/certs/one.felixliang.local.crt
       one_fqdn: one.felixliang.local
   
   frontend:
     hosts:
       f1: { ansible_host: 172.31.11.100 }
   
   node:
     hosts:
       n1: { ansible_host: 172.31.11.101 }
       n2: { ansible_host: 172.31.11.102 }
   ```
4. Verify the connection to servers of front-end and KVM hosts. When prompt, enter the password for `sysadmin`. (You might be asked for the local `admin` password first, and then the password for `sysadmin` of remote servers.)
   ```
   sudo ansible -i inventory/shared.yml all -m ping -b -k
   ```
6. Now we can run the playbook that install and configure OpenNebula services.
   ```
   sudo ansible-playbook -i inventory/shared.yml opennebula.deploy.main -k
   ```
## P6 Verifying the Installation
By following the steps [here](https://github.com/OpenNebula/one-deploy/wiki/sys_verify), perform a quick check to verify your installation and your Opennebula system functions.


