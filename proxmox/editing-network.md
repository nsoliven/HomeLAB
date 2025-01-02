### **Editing Network Configuration**

1. **Locate Netplan configuration files:**
    
    ```bash
    ls /etc/netplan/
    ```
    
    Files typically have names like `01-netcfg.yaml` or `50-cloud-init.yaml`.
    
2. **Edit the configuration file:**
    
    ```bash
    sudo nano /etc/netplan/<filename>.yaml
    ```
    
    Replace `<filename>` with the actual file name.
    

---

### **Sample Configurations**

#### Static IP Configuration:

```yaml
network:
  version: 2
  ethernets:
    ens33:  # Replace with your interface name
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # Replace with your static IP
      gateway4: 192.168.1.1  # Replace with your gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

#### DHCP Configuration:

```yaml
network:
  version: 2
  ethernets:
    ens33:  # Replace with your interface name
      dhcp4: yes
```

---

### **Apply Configuration**

- Apply the changes:
    
    ```bash
    sudo netplan apply
    ```
    

---

### **Verify Network Settings**

- Check the IP address:
    
    ```bash
    ip addr show
    ```
    
- Test connectivity:
    
    ```bash
    ping -c 4 8.8.8.8
    ping -c 4 google.com
    ```
    

---

### **Other Useful Commands**

- **Restart networking services:**
    
    ```bash
    sudo systemctl restart systemd-networkd
    ```
    
- **Backup Netplan configuration:**
    
    ```bash
    sudo cp /etc/netplan/<filename>.yaml /etc/netplan/<filename>.backup
    ```
    
- **Restore from backup:**
    
    ```bash
    sudo mv /etc/netplan/<filename>.backup /etc/netplan/<filename>.yaml
    sudo netplan apply
    ```
    