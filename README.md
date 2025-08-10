# 🚀 Menghubungkan n8n dengan Nginx Proxy Manager

> Panduan lengkap agar n8n bisa diakses via **domain HTTPS**, mendukung **WebSocket**, dan semua webhook **menggunakan URL eksternal yang benar**.

---

## 1. Siapkan Docker Network (jika belum ada)

```bash
docker network create external_net
```

---

## 2. Compose n8n

```yaml
# docker-compose.yml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    environment:
      # --- koneksi balik dari luar ---
      N8N_HOST: n8n.domainmu.com
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://n8n.domainmu.com/
      # --- reverse-proxy hacks ---
      N8N_PROXY_HOPS: 1
      # --- fitur baru (rekomen) ---
      N8N_RUNNERS_ENABLED: true
      # opsional
      GENERIC_TIMEZONE: Asia/Makassar
    networks:
      - external_net
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n

networks:
  external_net:
    external: true

volumes:
  n8n_data:
```

**Notes:**
- `WEBHOOK_URL` wajib agar semua link webhook eksternal di UI n8n **sudah menjadi HTTPS**.
- `N8N_PROXY_HOPS=1` memberitahu bahwa kita punya 1 layer proxy (NPM).
- `N8N_RUNNERS_ENABLED=true` adalah flag perubahan internal n8n terbaru.

---

## 3. Konfigurasi Nginx Proxy Manager

Trigger langkah berikut di NPM Dashboard:

1. **Proxy Hosts ▸ Add Proxy Host**
   * Domain Names: `n8n.domainmu.com`
   * Scheme: `http`
   * Forward Hostname/IP: `n8n` (nama container)
   * Forward Port: `5678`

2. **Advanced ▸ Centang:**
   ☑ Enable Websockets Support
   ☑ Block Common Exploits

3. **SSL ▸ Request a Let’s Encrypt Certificate**
   ☑ Force SSL
   ☑ HTTP/2 Support *(terserah)*

---

## 4. Jalankan & Uji

```bash
docker compose up -d
docker logs -f n8n
```

### Cek hasil
Buka di browser: [https://n8n.domainmu.com](https://n8n.domainmu.com)

---

## 5. Troubleshoot Cheat-Sheet

| Symptom                                | Solusi Cepat                                                                 |
|----------------------------------------|--------------------------------------------------------------------------------|
| “Connection lost” di editor            | Pastikan `N8N_PROXY_HOPS=1`<br>Pastikan **Websockets enabled** di NPM          |
| Webhook/callback pakai `localhost:5678`| Pastikan `WEBHOOK_URL` terisi dengan **domain HTTPS**                          |
| Cookie secure gagal (login loop)       | Pakai HTTPS saja. Bila terpaksa lokal HTTP, set `N8N_SECURE_COOKIE=false`      |
| NPM tidak bisa reach container         | Pastikan NPM container **join `external_net`**                               |
| WebSocket putus-nyambung               | Pastikan lagi checkbox **Websockets Support & Force SSL**                      |

<br>
```bash
# restart cepat setelah ubah
docker compose restart
```

---

Semoga membantu! ✨

