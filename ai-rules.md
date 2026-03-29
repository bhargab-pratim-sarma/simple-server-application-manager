# AI Agent Rules & Guidelines

This document provides instructions for AI agents working on the **Server Manager** repository. Follow these rules strictly to ensure security and consistency.

## 1. Branching & Secret Management (CRITICAL)

This repository uses a two-branch strategy to manage secrets:
- **`main` (Public)**: Contains only code and configuration. **NEVER** commit `.env` files or real secrets here.
- **`private` (Private)**: Tracks `.env` files and real credentials. Used for deployment and internal tracking.

### Rules for Agents:
- **ALWAYS** check which branch you are on before suggesting a commit.
- **NEVER** merge `private` into `main`. This is the #1 security risk.
- **Syncing**: If you update a configuration feature on `main`, merge `main` into `private` to sync, never the other way around.
- **Example Files**: When adding a new application, always create a `.env.example` in the `main` branch with placeholder values.

## 2. Directory Structure

- `traefik-setup/`: Core infrastructure (Traefik proxy, SSL, Dashboard).
- `applications/`: Individual Dockerized applications.
    - Each app must have its own directory.
    - Each app must have a `docker-compose.yml` and a `.env.example`.
    - Routing is handled via Traefik labels.

## 3. Docker & Traefik Standards

- **Networks**: Most apps should connect to the `traefik_public` network (ensure it exists).
- **Labels**: Follow the established pattern for Traefik labels in `docker-compose.yml`:
  ```yaml
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.${ROUTER_NAME}.rule=Host(`${APP_DOMAIN}`)"
    - "traefik.http.routers.${ROUTER_NAME}.entrypoints=websecure"
    - "traefik.http.routers.${ROUTER_NAME}.tls.certresolver=myresolver"
  ```
- **Environment Variables**: Use variables (e.g., `${APP_DOMAIN}`) in compose files and define them in `.env`.

## 4. Documentation

- Every new application must include a `README.md` explaining:
    - What the app does.
    - Required environment variables.
    - Specific setup steps.
- Add new apps to the main `README.md` list.

## 5. Security Precautions

- Do not expose the Traefik dashboard without `DASHBOARD_AUTH`.
- Ensure `.ssh/` and key files are **ALWAYS** ignored in `.gitignore`, even on the `private` branch.
