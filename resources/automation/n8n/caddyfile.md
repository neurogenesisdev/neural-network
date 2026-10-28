---
type: resource
status: active
created: 2026-04-21
updated: '2026-04-21T11:15:25Z'
tags: [reference, automation, n8n, caddy, reverse-proxy]
---

n8n.neurogenesis.dev {
    reverse_proxy n8n_app:5678
    
    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000;"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy no-referrer-when-downgrade
    }
}
