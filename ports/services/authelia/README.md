# Authelia
[Authelia](https://github.com/authelia/authelia) is a single sign-on multi-factor portal for web apps.

Below are:
- Instructions for setting up Authelia
- Example usage with Caddy reverse proxy
- Docker compose file
- A post about Authelia from Caddy forum

## 0. Project directory

My ***Authelia*** directory is located at ``/srv/server/services/authelia``. You can change it but remember to also change in inside ``docker-compose.yml``.

Inside, you need to create a directory (in my case ``config``) and make sure you have following files inside:

``config`` directory of *Authelia*:
├── configuration.yml
├── secrets
│   ├── JWT_SECRET
│   ├── REDIS_PASSWORD
│   ├── SESSION_SECRET
│   ├── SMTP_PASSWORD
│   ├── STORAGE_ENCRYPTION_KEY
│   └── STORAGE_PASSWORD
└── users_database.yml

Create ``config/secrets`` directory:
```bash
mkdir -p config/secrets
```

## 1. Secrets
Create your secrets:
```bash
tr -cd '[:alnum:]' < /dev/urandom | fold -w 64 | head -n 1 | tr -d '\n' > config/secrets/JWT_SECRET
tr -cd '[:alnum:]' < /dev/urandom | fold -w 64 | head -n 1 | tr -d '\n' > config/secrets/SESSION_SECRET
tr -cd '[:alnum:]' < /dev/urandom | fold -w 64 | head -n 1 | tr -d '\n' > config/secrets/STORAGE_PASSWORD
tr -cd '[:alnum:]' < /dev/urandom | fold -w 64 | head -n 1 | tr -d '\n' > config/secrets/STORAGE_ENCRYPTION_KEY
tr -cd '[:alnum:]' < /dev/urandom | fold -w 64 | head -n 1 | tr -d '\n' > config/secrets/REDIS_PASSWORD
```

Create ``SMTP_PASSWORD`` and paste your SMTP password here:
```bash
nano config/secrets/SMTP_PASSWORD
```

## 2. Authelia configuration
``config/configuration.yml``
```bash
# Miscellaneous https://www.authelia.com/configuration/miscellaneous/introduction/
# Set also AUTHELIA_JWT_SECRET_FILE
theme: auto

# First Factor https://www.authelia.com/configuration/first-factor/file/
authentication_backend:
  file:
    path: /config/users_database.yml

# Second Factor https://www.authelia.com/configuration/second-factor/introduction/
totp:
  issuer: yourdomain.tld # CHANGE ME

# Security https://www.authelia.com/configuration/security/access-control/
access_control:
  default_policy: deny
  rules:
  - domain: '*.yourdomain.tld' # CHANGE ME
    policy: 'two_factor'

# Session https://www.authelia.com/configuration/session/introduction/
# Set also AUTHELIA_SESSION_SECRET_FILE
session:
  cookies:
    - domain: 'yourdomain.tld' # CHANGE ME
      authelia_url: 'https://auth.yourdomain.tld'

  # https://www.authelia.com/configuration/session/redis/
  # Set also AUTHELIA_SESSION_REDIS_PASSWORD_FILE if appropriate
  redis:
    host: redis
    port: 6379

# Storage https://www.authelia.com/configuration/storage/postgres/
# Set also AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE
# Set also AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE
storage:
  postgres:
    address: 'tcp://database:5432'
    database: authelia
    username: authelia

# SMTP Notifier https://www.authelia.com/configuration/notifications/smtp/
# Set also AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE
notifier:
# I decided to use Gmail. You can choose any SMTP provider
  smtp:
    address: 'submissions://smtp.gmail.com:465'
    username: your-email@gmail.com
    sender: 'Authelia <authelia@yourdomain.tld>'
```

## 3. Authelia user database
``config/users_database.yml``
```bash
users:
    your_username:
        password: HASHED_PASSWORD
        displayname: "John Doe"
        email: your-email@gmail.com
```

Replace everything with your data and ``HASHED_PASSWORD`` with **argon2** hash like this:
```bash
docker run --rm -it authelia/authelia:latest authelia crypto hash generate argon2
```

...and copy that hash. It will look like ``$argon2id$v=19$m=65536,t=3,p=4$``

This service require more love than average but it's really worth it. I highly recommend following this [post from Caddy forum](https://caddy.community/t/securing-web-apps-with-caddy-and-authelia-in-docker-compose-an-opinionated-practical-and-minimal-production-ready-login-portal-guide/20465). It is more detailed, step-by-step guide how to setup 2FA with **T**ime-based **O**ne-**T**ime **P**asswords (TOTP). I based my setup on this.

## 4. Docker compose file

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

## 5. Usage with Caddy

To use *Authelia* with service of your choice, just add this to your ``Caddyfile``:
```python
(secure) {
    forward_auth {args[0]} authelia:9091 {
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