# Services - Port Forwarding
There are all services I made config for. All you need to do is follow instructions for particular service (such as change password) and run them either in ***Portainer*** (recommended) or just ``docker compose up -d``

In every case you need to run:
- **[Caddy](caddy)** - makes your sites more secure, more reliable, and more scalable than any other solution.
- **[WireGuard VPN](wg_easy)** - the easiest way to run WireGuard VPN + Web-based Admin UI.

Only required if you don't have static public IP:
- **[Cloudflare DDNS](cloudflare_ddns)** - small, feature-rich, and robust Cloudflare DDNS updater.