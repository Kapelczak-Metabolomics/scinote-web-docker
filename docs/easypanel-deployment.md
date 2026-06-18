# Deploy SciNote on EasyPanel

This guide deploys SciNote with the EasyPanel Compose service using
[`docker-compose.easypanel.yml`](../docker-compose.easypanel.yml).

The stack contains:

- `web`: Rails/Puma app, exposed internally on port `3000`
- `jobs`: delayed job worker for background jobs
- `db`: private PostgreSQL database
- `postgres_data`: persistent database volume
- `scinote_storage`: persistent Active Storage uploads volume

## 1. Prepare DNS

Create a DNS record for the domain you want to use, for example:

```text
scinote.example.com -> your EasyPanel server IP
```

## 2. Create the EasyPanel Compose service

1. Open EasyPanel.
2. Create a new **Compose** service.
3. Configure the source:
   - **Repository**: your SciNote repository URL
   - **Branch**: the branch you want to deploy
   - **Root path**: `/`
   - **Compose file**: `docker-compose.easypanel.yml`
4. Save the service, but add the environment variables below before the first
   deploy.

## 3. Set required environment variables

Add these variables in the EasyPanel service environment.

| Variable | Example | Notes |
| --- | --- | --- |
| `POSTGRES_PASSWORD` | generated password | Use a long random value. |
| `SECRET_KEY_BASE` | generated hex string | Generate with `openssl rand -hex 64`. Do not change after deploy. |
| `WEB_SERVER_URL` | `scinote.example.com` | Use the public SciNote host name. |
| `ADMIN_NAME` | `Admin` | Name for the first admin user. |
| `ADMIN_EMAIL` | `admin@example.com` | Email for the first admin user. |
| `ADMIN_PASSWORD` | generated password | Initial admin password. Change it after login. |

You can generate secrets locally with:

```bash
openssl rand -hex 32  # POSTGRES_PASSWORD
openssl rand -hex 64  # SECRET_KEY_BASE
```

Optional variables:

| Variable | Default | Notes |
| --- | --- | --- |
| `POSTGRES_DB` | `scinote_production` | Database name. |
| `POSTGRES_USER` | `scinote` | Database user. |
| `RAILS_LOG_LEVEL` | `info` | Rails log level. |
| `RAILS_FORCE_SSL` | empty | Set to `true` only if you want Rails to enforce HTTPS behind EasyPanel. |
| `ENABLE_USER_REGISTRATION` | `true` | Set to `false` to disable public signups. |
| `MAIL_FROM` / `MAIL_REPLYTO` | empty | Required for outbound email sender addresses. |
| `SMTP_ADDRESS` | empty | Set this plus the SMTP variables below to enable SMTP delivery. |
| `SMTP_PORT` | `587` | SMTP port. |
| `SMTP_DOMAIN` | empty | SMTP HELO/domain value. |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | empty | SMTP credentials. |

If you configure SMTP, set at least `MAIL_FROM`, `MAIL_REPLYTO`,
`SMTP_ADDRESS`, `SMTP_DOMAIN`, `SMTP_USERNAME`, and `SMTP_PASSWORD`.

## 4. Deploy

Deploy the Compose service from EasyPanel.

On first startup, the `web` service runs:

```bash
rails db:prepare
rails db:seed
rails server -b 0.0.0.0 -p 3000
```

The seed task creates the initial admin user only when the database has no
users. Later restarts keep existing users and data.

The first build can take several minutes because the production image installs
system dependencies such as Chromium, LibreOffice, Java, and PostgreSQL client
tools.

## 5. Attach your domain

1. In EasyPanel, open the deployed Compose service.
2. Add a domain for the `web` service.
3. Set the target/proxy port to `3000`.
4. Enable HTTPS in EasyPanel.

Do not expose the `db` service publicly.

## 6. Log in

Open your domain in a browser and sign in with:

```text
Email:    ADMIN_EMAIL
Password: ADMIN_PASSWORD
```

Change the admin password after the first login.

## 7. Backups and updates

- Back up both EasyPanel volumes:
  - `postgres_data`
  - `scinote_storage`
- Before changing `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, or
  `SECRET_KEY_BASE`, make a backup. Changing `SECRET_KEY_BASE` can invalidate
  existing signed/encrypted Rails data.
- To update SciNote, redeploy the Compose service from EasyPanel. Database
  migrations run automatically during `web` startup.
