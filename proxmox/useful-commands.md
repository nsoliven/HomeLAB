Here are some useful commands for managing Proxmox, covering tasks such as managing VMs, containers, storage, backups, and networking:

### VM Management

1. **Create a VM**:
   ```bash
   qm create 100 --name myvm --memory 2048 --net0 virtio,bridge=vmbr0
   ```

2. **Start a VM**:
   ```bash
   qm start 100
   ```

3. **Stop a VM**:
   ```bash
   qm stop 100
   ```

4. **Delete a VM**:
   ```bash
   qm destroy 100
   ```

5. **List All VMs**:
   ```bash
   qm list
   ```

6. **Show VM Configuration**:
   ```bash
   qm config 100
   ```

### Container Management

1. **Create a Container**:
   ```bash
   pct create 101 local:vztmpl/debian-10-standard_10.0-1_amd64.tar.gz --rootfs local-lvm:8 --hostname mycontainer --net0 name=eth0,bridge=vmbr0,ip=dhcp --start 1
   ```

2. **Start a Container**:
   ```bash
   pct start 101
   ```

3. **Stop a Container**:
   ```bash
   pct stop 101
   ```

4. **Delete a Container**:
   ```bash
   pct destroy 101
   ```

5. **List All Containers**:
   ```bash
   pct list
   ```

6. **Show Container Configuration**:
   ```bash
   pct config 101
   ```

### Storage Management

1. **List Storage Devices**:
   ```bash
   pvesm status
   ```

2. **Add a Storage Device**:
   ```bash
   pvesm add dir local-backup --path /mnt/backup
   ```

3. **Remove a Storage Device**:
   ```bash
   pvesm remove local-backup
   ```

### Backup and Restore

1. **Backup a VM**:
   ```bash
   vzdump 100 --dumpdir /mnt/backup/
   ```

2. **Restore a VM from Backup**:
   ```bash
   qmrestore /mnt/backup/vzdump-qemu-100-2024_07_08-11_00_00.vma.lzo 100
   ```

### Network Management

1. **List Network Bridges**:
   ```bash
   brctl show
   ```

2. **Create a Network Bridge**:
   ```bash
   ip link add name vmbr1 type bridge
   ip link set dev vmbr1 up
   ```

3. **Add an Interface to a Bridge**:
   ```bash
   ip link set dev eth0 master vmbr1
   ```

### Node Management

1. **Show Node Status**:
   ```bash
   pveperf
   ```

2. **Update Proxmox Packages**:
   ```bash
   apt update && apt upgrade -y
   ```

3. **Reboot the Node**:
   ```bash
   reboot
   ```

### User and Permission Management

1. **Create a User**:
   ```bash
   pveum useradd user@pve -comment "Example User" -password
   ```

2. **Delete a User**:
   ```bash
   pveum userdel user@pve
   ```

3. **Assign a Role to a User**:
   ```bash
   pveum aclmod / -user user@pve -role Administrator
   ```

### Miscellaneous

1. **Check Proxmox Version**:
   ```bash
   pveversion
   ```

2. **Show Proxmox Cluster Status**:
   ```bash
   pvecm status
   ```