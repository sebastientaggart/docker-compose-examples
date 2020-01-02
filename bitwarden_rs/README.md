# Bitwarden Password Manager

Bitwarden is an open source alternative to LastPass and similar services.  They provide a hosted service at https://bitwarden.com/  They also make their software available as a self-hosted package.  While their [on-premises hosting instructions](https://help.bitwarden.com/article/install-on-premise/) are very good, I found them too complex for the limited, personal use I had planned for me and my close friends and family.

Luckily there is an unofficial Bitwarden-compatible server that ships as a single docker container with all the necessary features: https://github.com/dani-garcia/bitwarden_rs

The `docker-compose.yml` in this project is based on `bitwarden_rs`, and is configured here to run behind a proxy.  In my case, a Caddy proxy.

NOTE: be sure to enable `/admin`, log in, and set the `Domain URL` under `General Settings`, otherwise emails will send with incorrect URL paths and verifications will fail.

## Caddy Proxy Configuration

Assumptions (see [global assumptions](../README.md) for more details):

- We are running multiple services under one domain, accessed via `XXX.mydomain.com`
- We share a wildcard cert across all the domains (to avoid flood limits)
- `8082` is used for web, and `3012` for the Bitwarden notifications hub

```
# Bitwarden Password Manager
https://bitwarden.MYDOMAIN.com {
  proxy / MYIPADDRESS:8082 {
    transparent
    header_upstream X-Forwarded-Ssl on
  }
  # The negotiation endpoint is also proxied to Rocket
  proxy /notifications/hub/negotiate MYIPADDRESS:80 {
      transparent
  }
  # Notifications redirected to the websockets server
  proxy /notifications/hub MYIPADDRESS:3012 {
      websocket
  }
  tls {
    max_certs 1
    wildcard
  }
  gzip
}
```

## docker-compose.yml notes

```
version: "3"

services:
  bitwarden:
    image: bitwardenrs/server:latest
    container_name: bitwarden
    environment:
      - SIGNUPS_ALLOWED=false
      - INVITATIONS_ALLOWED=false
      - ADMIN_TOKEN=f8G62pH5rT5tzVMq6cbMW72g9+i/qkwcoVGNm/pIbmxdTLQt+BzJ+e7jGqPgU/wp
      - WEBSOCKET_ENABLED=true
      - SMTP_HOST=smtp.gmail.com
      - SMTP_FROM=<bitwarden@domain.tld>
      - SMTP_PORT=465
      - SMTP_SSL=true
      - SMTP_USERNAME=<username>
      - SMTP_PASSWORD=<password>
    ports:
      - 8082:80
      - 3012:3012
    volumes:
      - ./data:/data/
    restart: always
```

- Consider using `image: bitwardenrs/server:1.XX.0` instead of `latest` because latest is evil
- `8082` and `3012` can be any other port that suits your environment
- Data in in `./data` but see https://github.com/dani-garcia/bitwarden_rs/wiki/Backing-up-your-vault for backup instructions
- `SIGNUPS_ALLOWED=false` because this is for personal use only.
- `INVITATIONS_ALLOWED=false` to disallow users from inviting others
- `ADMIN_TOKEN` activates access to the admin area, use `openssl rand -base64 48` to generate a good password.  With this `/admin` is available.
- `WEBSOCKET_ENABLED=true` to enable notifications at the `/notifications/hub` endpoint
- `SMTP_*` to allow sending of email, comment out these entries to disable
