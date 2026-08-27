# valdoh.com

Static site. No build step, no dependencies — plain HTML and one stylesheet.

```
index.html          landing: hero + project grid
memodi/index.html   project detail page
style.css           shared styles
img/                assets
deploy/             systemd unit for the server
```

Adding a project is pasting one `<a class="card">` block into the grid in `index.html` and
creating `<project>/index.html`.

## Local preview

```bash
python3 -m http.server 8088
```

## Production

Served from the same home server as [memodi](https://github.com/iam-oov/memodi), behind the
existing Cloudflare Tunnel. Canonical host is `https://www.valdoh.com`; the apex redirects to it.

### One-time setup

**1. Confirm the tunnel is dashboard-managed**

```bash
systemctl cat cloudflared
```

`--token` in `ExecStart` means public hostnames are managed in the Zero Trust dashboard (step 3).
If it reads `--config /etc/cloudflared/config.yml` instead, the ingress rules go in that file
above the catch-all `service: http_status:404`.

**2. Clone and serve**

As user `memodi`:

```bash
git clone https://github.com/iam-oov/valdoh.com-page.git /home/memodi/valdoh.com-page
sudo install -m 644 /home/memodi/valdoh.com-page/deploy/valdoh-site.service \
  /etc/systemd/system/valdoh-site.service
sudo systemctl daemon-reload
sudo systemctl enable --now valdoh-site
curl -sI http://127.0.0.1:8088/ | head -1   # 200
```

Binds to `127.0.0.1` — cloudflared connects locally, so the LAN must not reach the port
directly. Same reason memodi binds to loopback.

**3. Cloudflare public hostname**

Zero Trust → Networks → Tunnels → the existing tunnel → Public Hostnames → Add:

| Hostname          | Service                 |
| ----------------- | ----------------------- |
| `www.valdoh.com`  | `http://localhost:8088` |
| `valdoh.com`      | `http://localhost:8088` |

Both DNS records are created automatically, proxied.

**4. Apex → www redirect**

Rules → Redirect Rules → Create rule:

- **When**: `Hostname` `equals` `valdoh.com`
- **Then**: Dynamic redirect, status `301`, preserve query string
- **Expression**: `concat("https://www.valdoh.com", http.request.uri.path)`

The redirect fires at Cloudflare's edge, so apex traffic never reaches the tunnel. The apex
hostname in step 3 exists only so the record resolves through Cloudflare — it is never served.

## Deploy

No restart needed: the file server reads from disk on every request.

```bash
ssh memodi@pi.valdoh.com 'cd ~/valdoh.com-page && git pull'
```

## Verify

```bash
curl -sI https://www.valdoh.com | head -1                                 # 200
curl -sI https://www.valdoh.com/memodi/ | head -1                         # 200
curl -sI https://valdoh.com | rg -i '^(HTTP|location)'                    # 301 -> www
curl -s -o /dev/null -w '%{http_code}\n' https://memodi.valdoh.com/login  # 302, memodi untouched
```
