# WordPress Traefik Template

This template provides a production-ready WordPress + MariaDB setup that works behind your existing Traefik proxy.

## 📁 Files
- `docker-compose.yml`: Defines the WordPress and Database services.
- `.env`: Contains all the configuration variables.

## 🚀 How to Deploy
1. **Prepare DNS:** Ensure your domain (e.g., `blog.example.com`) is pointing to your server's IP via an A-record.
2. **Copy Template:** Copy this `wordpress` folder to your target location on the server.
3. **Configure:** Update `.env` with your domain, unique router name, and strong passwords.
4. **Launch:**
   ```bash
   docker compose up -d
   ```

## 🔐 One-Area Configuration
You should only ever need to edit the `.env` file to deploy multiple instances of this. Just ensure the `ROUTER_NAME` and `CONTAINER_NAME` are unique for every new installation.
