# Briefkasten
This [bookmarking](https://github.com/ndom91/briefkasten) app lets you manage all your bookmarks, group them and do basically whatever you want with them. It supports many browsers through addons.

Unfortunately it's not simple *copy-paste*, because you need to grab your Github or Facebook credentials to use this app.

``docker-compose.yml``
```yaml
version: '3.8'

services:
  briefkasten-database:
    container_name: bk-postgres
    image: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=bkAdmin
      - POSTGRES_PASSWORD=briefkasten
      - POSTGRES_DB=briefkasten
    ports:
      - 5432:5432
    volumes:
      - briefkasten-db:/var/lib/postgresql/data
  app:
    container_name: bk-app
    image: ndom91/briefkasten
    restart: unless-stopped
    environment:
      - DATABASE_URL=postgres://bkAdmin:briefkasten@briefkasten-database:5432/briefkasten?sslmode=disable
      - NEXTAUTH_URL=https://briefkasten.YOURDOMAIN.COM
      - NEXTAUTH_SECRET=
      - GITHUB_ID=
      - GITHUB_SECRET=
      - GOOGLE_ID=
      - GOOGLE_SECRET=

    ports:
      - 3000:3000
    depends_on:
      - 'briefkasten-database'
volumes:
  briefkasten-db:
```

