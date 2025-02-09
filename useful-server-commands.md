# Usefull commands for servers.

### Windows ssh-copy-id command alternative
```powershell
Get-Content $env:USERPROFILE\.ssh\*.pub | ssh user@ip_addr "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
