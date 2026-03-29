# ============================================
# How to use this template
# ============================================
#
# 1. Copy this entire folder for your new app:
#      cp -r templates/blog-example /root/my-new-app
#
# 2. Edit the .env file:
#      nano /root/my-new-app/.env
#
#    Change these values:
#      APP_DOMAIN    → Your app's domain (e.g., app.namani.in)
#      ROUTER_NAME   → A unique name (e.g., myapp)
#      APP_IMAGE     → The Docker image (e.g., nginx:latest)
#      APP_PORT      → The port your app listens on (e.g., 80)
#      CONTAINER_NAME → A friendly name (e.g., my-app)
#
# 3. Make sure the domain's DNS points to your server IP
#
# 4. Start it:
#      cd /root/my-new-app
#      docker compose up -d
#
# 5. Traefik will automatically:
#      ✅ Discover the container
#      ✅ Generate an SSL certificate
#      ✅ Start routing traffic
#
# ============================================
# Common APP_PORT values:
# ============================================
#   nginx         → 80
#   Node.js/Next  → 3000
#   Ghost Blog    → 2368
#   WordPress     → 80
#   Grafana       → 3000
#   Portainer     → 9000
#   Jellyfin      → 8096
#   Gitea         → 3000
#   Uptime Kuma   → 3001
#   MinIO Console → 9001
# ============================================
