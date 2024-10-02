# Cloudflare DDNS
[Cloudflare DDNS](https://github.com/favonia/cloudflare-ddns) is a small, feature-rich, and robust Cloudflare DDNS updater.

Configuration below creates three A DNS records. All except ``mc.your-domain.tld`` gets *Proxied*. What does it mean is that Cloudflare hides your real IP in HTTP and DNS requests and gives you better protection methods, mostly available in their generous Free Tier.

``docker-compose.yml``
```yaml
services:
  cloudflare_ddns:
    image: favonia/cloudflare-ddns:latest
    container_name: cloudflare_ddns
    network_mode: host
    restart: always
    user: "1000:1000"
    cap_drop:
      - all
    read_only: true
    security_opt:
      - no-new-privileges:true
    environment:
      - CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
      - IP6_PROVIDER=none
      - DOMAINS=your-domain.tld,*.your-domain.tld,mc.your-domain.tld
      - CACHE_EXPIRATION=3h
      - TZ=Europe/Warsaw
      # - PROXIED=true
      - PROXIED=!is(mc.your-domain.tld)
```