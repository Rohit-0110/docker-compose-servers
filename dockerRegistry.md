Using official Docker Registry with a Web-UI, using Docker Compose. You’ll get:

* Private registry with **htpasswd (basic auth)** handled by the registry itself
* **TLS** termination via Nginx (works with Docker CLI)
* A **web UI** (Joxit/docker-registry-ui) to browse repos/tags, delete images, etc.
* Persistent storage for images and credentials

---

# 1) File layout

```text
docker-registry/
├─ docker-compose.yml
├─ nginx.conf
├─ auth/
│  └─ htpasswd
└─ certs/
   ├─ tls.crt
   └─ tls.key
```

---

# 2) Create credentials

Generate an htpasswd file (bcrypt):

```bash
mkdir -p docker-registry/auth docker-registry/certs
cd docker-registry

# replace USERNAME with your desired username
docker run --rm httpd:2.4-alpine htpasswd -Bbn <username> <password> > auth/htpasswd
```

> You can change the password later by regenerating and replacing `auth/htpasswd`, then restarting.

---

# 3) TLS certificate

For testing, make a self-signed cert (use your real domain in `CN` and `subjectAltName`):

```bash
# Self-signed—replace myregistry.com with your domain
openssl req -x509 -nodes -days 365 \
  -newkey rsa:4096 \
  -keyout certs/tls.key -out certs/tls.crt \
  -subj "/CN=myregistry.com" \
  -addext "subjectAltName=DNS:myregistry.com"
```

> In production, use a real cert (e.g., via Let’s Encrypt). Just drop `fullchain.pem` as `tls.crt` and `privkey.pem` as `tls.key` in `certs/`.

---

# 4) `docker-compose.yml`

```yaml
version: "3.9"

services:
  registry:
    image: registry:2
    container_name: registry
    restart: unless-stopped
    environment:
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_HTTP_RELATIVEURLS: "true"
    volumes:
      - registry-data:/var/lib/registry
      - ./auth:/auth:ro
    networks:
      - net

  ui:
    image: joxit/docker-registry-ui:latest
    container_name: registry-ui
    restart: unless-stopped
    environment:
      REGISTRY_TITLE: Organisation
      SINGLE_REGISTRY: "true"
      NGINX_PROXY_PASS_URL: https://registry-proxy
      DELETE_IMAGES: "true"
      SHOW_CATALOG_NB_TAGS: "true"
    depends_on:
      - registry
    networks:
      - net

  nginx:
    image: nginx:alpine
    container_name: registry-proxy
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs/tls.crt:/etc/nginx/tls/tls.crt:ro
      - ./certs/tls.key:/etc/nginx/tls/tls.key:ro
    depends_on:
      - registry
    networks:
      - net

volumes:
  registry-data:

networks:
  net:
    driver: bridge

```

---

# 5) `nginx.conf`

This serves the UI at `/` and proxies the Docker Registry API at `/v2/` over TLS. Docker clients will talk to `https://myregistry.com` (replace with your domain).

```nginx
server {
  listen 443 ssl http2;
  server_name myregistry.com;

  ssl_certificate     /etc/nginx/tls/tls.crt;
  ssl_certificate_key /etc/nginx/tls/tls.key;

  # Hardening (good defaults)
  client_max_body_size 0;
  chunked_transfer_encoding on;

  # UI at /
  location / {
    proxy_pass http://registry-ui:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # Docker Registry v2 API at /v2/
  location /v2/ {
    proxy_pass http://registry:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Required for chunked uploads
    proxy_request_buffering off;
  }
}
```

> If you prefer **separate hostnames**, you can split this into two server blocks, e.g. `registry.example.com` for `/v2/` and `ui.example.com` for `/`.

---

# 6) Bring it up

```bash
docker compose up -d
```

* UI: `https://myregistry.com/`
* Registry endpoint (Docker CLI): `https://myregistry./v2/`

---

# 7) Use it

Login (Docker CLI will prompt for the password):

```bash
docker login myregistry.example.com
# Username: <username>
# Password: ****
```

Tag & push:

```bash
docker pull alpine:latest
docker tag alpine:latest myregistry.com/alpine:latest
docker push myregistry.com/alpine:latest
```

Browse/manage in the UI at `https://myregistry.com/`.

---

## Maintenance tips

* **Delete images**: enable via `REGISTRY_STORAGE_DELETE_ENABLED=true` (already set). After deleting tags, run GC to reclaim space:

  ```bash
  docker exec -it registry bin/registry garbage-collect /etc/docker/registry/config.yml
  ```

  (Stop pushes during GC to be safe.)

* **Backups**: back up the `registry-data` volume and `auth/htpasswd`.

* **Rotate passwords**: regenerate `auth/htpasswd` and restart the stack:

  ```bash
  docker compose restart registry
  ```

* **Certificates on clients** (self-signed only): you may need to add the CA/cert to Docker’s trust store on each client, or switch to a proper TLS cert to avoid this.
