# Pastefy
Pastefy is an Open Source self-hostable Pastebin.

> **Important!** Remember to change the passwords and provide your OAuth2 credentials before proceeding.

## Making yourself admin
To make yourself an admin, you need to login with your account and get inside your Docker container:
```bash
docker exec -it pastefy_db mysql -u pastefy -ppastefy pastefy
```
Remember to **not** put a space in ``-ppastefy``.

Then run following SQL:
```sql
SELECT id, unique_name, type FROM pastefy_users;
```

and:
```sql
UPDATE pastefy_users SET type='ADMIN' WHERE id='id_copied_from_above';
```

``docker-compose.yml``
```yaml
services:
   pastefy_db:
     container_name: pastefy_db
     image: mariadb:10.11
     networks:
       - pastefy
     volumes:
       - /srv/server/services/pastefy/db:/var/lib/mysql
     environment:
       MYSQL_ROOT_PASSWORD: pastefy
       MYSQL_DATABASE: pastefy
       MYSQL_USER: pastefy
       MYSQL_PASSWORD: pastefy
   pastefy:
     container_name: pastefy
     depends_on:
       - pastefy_db
     image: interaapps/pastefy:latest
     ports:
       - "9999:80"
     networks:
       - pastefy
       - caddy
     environment:
       HTTP_SERVER_PORT: 80
       HTTP_SERVER_CORS: "*"
       DATABASE_DRIVER: mysql
       DATABASE_NAME: pastefy
       DATABASE_USER: pastefy
       DATABASE_PASSWORD: pastefy
       DATABASE_HOST: pastefy_db
       DATABASE_PORT: 3306
       SERVER_NAME: "https://yourdomain.tld"
       # OPTIONAL
       PASTEFY_INFO_CUSTOM_NAME: CustomTitle
       PASTEFY_INFO_CUSTOM_FOOTER: https://yourdomain.tld
       # There is INTERAAPPS, GOOGLE, GITHUB, DISCORD, TWITCH; you can use multiple
       OAUTH2_GITHUB_CLIENT_ID:  
       OAUTH2_GITHUB_CLIENT_SECRET: 
       # Login-requirements for specific access types
       PASTEFY_LOGIN_REQUIRED_CREATE: true
       # This will disable the raw mode as well for browser users
       PASTEFY_LOGIN_REQUIRED_READ: false
       # Check the encryption checkbox by default
       PASTEFY_ENCRYPTION_DEFAULT: false
       # Requires every new account being accepted by an administrator
       PASTEFY_GRANT_ACCESS_REQUIRED: true
       # Allows /paste route listing all pastes
       PASTEFY_LIST_PASTES: true
       # Makes /app/stats public
       PASTEFY_PUBLIC_STATS: true
       # Disables public pastes section
       PASTEFY_PUBLIC_PASTES: true
       
networks:
  caddy:
    name: caddy
    external: true
  pastefy:
    name: pastefy
```