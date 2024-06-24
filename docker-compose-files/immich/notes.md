## Data Storage Customization:

Immich stores various data types in separate locations on the host machine.

**With this docker compose file, you can setup different locations using the environment variables!**

These locations are defined using environment variables (LIBRARY_LOCATION, UPLOAD_LOCATION, THUMBS_LOCATION, PROFILE_LOCATION, VIDEO_LOCATION).
This allows you to choose the storage solution that best suits your needs (e.g., local disk, NAS, cloud storage).