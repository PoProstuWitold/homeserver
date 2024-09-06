# Authelia
[Authelia](https://github.com/authelia/authelia) is a single sign-on multi-factor portal for web apps.

This service require more love than average but it's really worth it. I highly recommend following this [post from Caddy forum](https://caddy.community/t/securing-web-apps-with-caddy-and-authelia-in-docker-compose-an-opinionated-practical-and-minimal-production-ready-login-portal-guide/20465). It is step-by-step guide how to setup 2FA with **T**ime-based **O**ne-**T**ime **P**asswords (TOTP).

Initial guide is kinda outdated and likely to change in Authelia v5.0.0, but [following comment](https://caddy.community/t/securing-web-apps-with-caddy-and-authelia-in-docker-compose-an-opinionated-practical-and-minimal-production-ready-login-portal-guide/20465/7) corrects parts that probably will not be supported anymore.

``docker-compose.yml``
```yaml
services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    depends_on:
      - database
      - redis
    volumes:
      - /srv/server/services/authelia/config:/config
    environment:
      AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE: /config/secrets/JWT_SECRET
      AUTHELIA_SESSION_SECRET_FILE: /config/secrets/SESSION_SECRET
      AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE: /config/secrets/SMTP_PASSWORD
      AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE: /config/secrets/STORAGE_ENCRYPTION_KEY
      AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE: /config/secrets/STORAGE_PASSWORD
      AUTHELIA_SESSION_REDIS_PASSWORD_FILE: /config/secrets/REDIS_PASSWORD
    networks:
      - authelia
      - caddy
  database:
    image: postgres:15
    container_name: authelia_db
    restart: unless-stopped
    volumes:
      - /srv/server/services/authelia/postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: "authelia"
      POSTGRES_PASSWORD: "STORAGE_PASSWORD"
    networks:
      - authelia
  redis:
    image: redis:7
    container_name: authelia_redis
    restart: unless-stopped
    command: "redis-server --save 60 1 --loglevel warning --requirepass REDIS_PASSWORD"
    volumes:
      - /srv/server/services/authelia/redis:/data
    networks:
      - authelia

networks:
  caddy:
    name: caddy
    external: true
  authelia:
    name: authelia
```

To use *Authelia* with service of your choice, just add this to your ``Caddyfile``:
```python
(secure) {
    forward_auth {args.0} authelia:9091 {
        uri /api/verify?rd=https://auth.{env.BASE_URL}
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
}
```

```python
@authelia host auth.{env.BASE_URL}
handle @authelia {
    reverse_proxy authelia:9091
}
```

and to service you want to secure:
```python
@mealie host mealie.{env.BASE_URL}
handle @mealie {
    import secure *
    reverse_proxy mealie:9000
}
```