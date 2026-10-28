---
type: resource
status: active
created: 2026-04-21
updated: '2026-04-21T11:15:25Z'
tags: [reference, automation, n8n, docker, docker-compose]
---

version: '3.8'

services:
  db:
    image: postgres:16-alpine
    restart: always
    environment:
      - POSTGRES_USER=n8n_user
      - POSTGRES_PASSWORD=replace_this_with_a_real_password
      - POSTGRES_DB=n8n
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n_user -d n8n"]
      interval: 5s
      timeout: 5s
      retries: 5

  caddy:
    image: caddy:2-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - n8n_app

  n8n_app:
    image: n8nio/n8n:1.45.1 # Hardcoded version for stability
    restart: always
    environment:
      - N8N_HOST=n8n.neurogenesis.dev
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_ENCRYPTION_KEY=4d@Dt-YC2dtyYRCAh3YP@HdX
      - WEBHOOK_URL=https://n8n.neurogenesis.dev/
      - N8N_SECURE_COOKIE=true
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_HOST=db
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=n8n_user
      - DB_POSTGRESDB_PASSWORD=replace_this_with_a_real_password
      - NODE_FUNCTION_ALLOW_EXTERNAL=moment,lodash # Useful for AI templates
    volumes:
      - ./n8n_data:/home/node/.n8n
    depends_on:
      db:
        condition: service_healthy

volumes:
  caddy_data:
  caddy_config:
