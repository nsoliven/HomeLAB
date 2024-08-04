# Nextcloud All-in-One Docker Compose Configuration Notes

## Services Configuration

### `services`
Defines the services to run. Here, it runs `nextcloud-aio-mastercontainer`.

### `image: nextcloud/all-in-one:latest`
Uses the latest stable version of Nextcloud All-in-One. I prefer to use the AIO to allow NextCloud Documents (Collabora) and etc to fully work.

### `ports`
- **`8081:8080`**: Maps port 8081 on the host to port 8080 in the container for web access.

### `environment`
- **`AIO_DISABLE_BACKUP_SECTION=true`**: Hides the backup section in the interface. Backup is DISABLED due to using TrueNAS for data protection instead.
- **`NEXTCLOUD_DATADIR=/mnt/webnexus-apps/next-aio`**: Defines the host directory for data storage on my TrueNAS ISCSI shares. Used for thumbnails and configurations.
- **`NEXTCLOUD_MOUNT=/mnt/nfs/`**: Using NFS for general storage of mounts.
- **`NEXTCLOUD_MEMORY_LIMIT=2048M`**: Sets PHP memory limit to 2GB for better performance. Allows for bigger uploads.

<div align="center">
  <img src="nextcloud-login.png" width="98%"/>
</div>