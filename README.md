# Buzz on Coolify

Deploy [Buzz](https://github.com/block/buzz) on a self-hosted [Coolify](https://coolify.io/) server using Docker Compose.

> [!IMPORTANT]
> This is an **unofficial community deployment configuration**. It is not maintained or endorsed by Block, Inc. For Buzz issues, documentation, and releases, use the [official Buzz repository](https://github.com/block/buzz).

## Overview

Buzz is a self-hostable workspace where humans and AI agents collaborate through a relay you control.

This repository provides a small, Coolify-oriented deployment stack for the server-side components:

- Buzz relay and bundled web interface
- PostgreSQL
- Redis
- S3-compatible object storage (RustFS)
- Persistent Git data
- Automatic database migrations

The Buzz desktop application is **not** hosted inside Coolify. Install the desktop client on your computer and connect it to the relay deployed by this stack.

```text
Buzz desktop / agents / CLI
             |
             | HTTPS + WebSocket
             v
       Coolify proxy
             |
             v
        Buzz relay
       /     |      \
PostgreSQL  Redis  RustFS
             |
       Persistent volumes
```

## Repository structure

```text
.
├── compose.yml
└── README.md
```

Secrets are configured in the Coolify interface. Do not add a production `.env` file to this public repository.

## Prerequisites

Before deploying, you need:

- A server running Coolify
- A domain or subdomain, such as `buzz.example.com`
- A DNS record pointing the domain to your Coolify server
- A 64-character hexadecimal Nostr public key for the workspace owner
- Strong, stable passwords and application secrets

## Deploy with Coolify

### 1. Add the repository

In Coolify:

1. Open the appropriate project and environment.
2. Select **New Resource**.
3. Choose **Public Repository**.
4. Enter the URL of this repository.
5. Select **Docker Compose** as the build pack.
6. Set the Compose file path to:

```text
/compose.yml
```

### 2. Configure environment variables

Add the following variables in Coolify under **Environment Variables**.

Replace `buzz.example.com` with your actual domain.

```env
# Buzz image
# Use :main only for testing. Pin a release or SHA for production.
BUZZ_IMAGE=ghcr.io/block/buzz:main

# Public URLs
BUZZ_DOMAIN=buzz.example.com
RELAY_URL=wss://buzz.example.com
BUZZ_MEDIA_BASE_URL=https://buzz.example.com/media
BUZZ_MEDIA_SERVER_DOMAIN=buzz.example.com
BUZZ_CORS_ORIGINS=https://buzz.example.com

# Production behavior
BUZZ_REQUIRE_AUTH_TOKEN=true
BUZZ_REQUIRE_RELAY_MEMBERSHIP=true
BUZZ_ALLOW_NIP_OA_AUTH=true
BUZZ_AUTO_MIGRATE=true
BUZZ_GIT_CONFORMANCE_PROBE=true
RUST_LOG=buzz_relay=info,buzz_db=info,buzz_auth=info,buzz_pubsub=info,tower_http=info

# Workspace owner
RELAY_OWNER_PUBKEY=REPLACE_WITH_64_CHARACTER_HEX_NOSTR_PUBLIC_KEY

# Stable application secrets
BUZZ_RELAY_PRIVATE_KEY=REPLACE_WITH_64_CHARACTER_HEX_SECRET
BUZZ_GIT_HOOK_HMAC_SECRET=REPLACE_WITH_64_CHARACTER_HEX_SECRET

# PostgreSQL
POSTGRES_DB=buzz
POSTGRES_USER=buzz
POSTGRES_PASSWORD=REPLACE_WITH_STRONG_PASSWORD

# Redis
REDIS_PASSWORD=REPLACE_WITH_STRONG_PASSWORD

# S3 storage (bundled RustFS)
BUZZ_S3_ACCESS_KEY=REPLACE_WITH_RANDOM_ACCESS_KEY
BUZZ_S3_SECRET_KEY=REPLACE_WITH_STRONG_SECRET
BUZZ_S3_BUCKET=buzz-media
```

### 3. Configure S3 storage

The S3 bucket is required for the default Buzz deployment because the relay uses object storage for media uploads and related file handling. By default, you do not need AWS or any cloud storage service for this setup. This Compose stack includes [RustFS](https://github.com/rustfs/rustfs), which provides S3-compatible storage locally inside the deployment, and an `s3-init` job that creates the bucket automatically during startup using the `BUZZ_S3_BUCKET` value.

- `BUZZ_S3_BUCKET` is required for the default deployment and should be a stable bucket name such as `buzz-media`.
- The bundled `s3-init` job creates the bucket automatically, so you normally do not need to create it manually in Coolify.
- If you want to use an external S3-compatible service instead of the bundled RustFS, set the matching endpoint and credentials as well, but the bucket name remains required.

If you do not configure S3-compatible storage, media uploads and related object storage features will not work correctly.

#### Migrating from MinIO

Earlier versions of this stack bundled MinIO, which is no longer maintained and no longer publishes container images. It has been replaced by RustFS, which uses the same `BUZZ_S3_*` variables, so `.env` does not need to change.

RustFS stores data in a new `buzz-s3-data` volume. The old `buzz-minio-data` volume is left untouched. If you already have uploaded media, copy it into the new bucket (for example with `rclone sync` or `aws s3 sync` while both services run), verify the object counts, and only then remove the old volume.

### 4. Generate secrets

You can generate 64-character hexadecimal secrets with:

```bash
openssl rand -hex 32
```

Generate each secret independently. Do not reuse the same value for multiple variables.

A strong password can be generated with:

```bash
openssl rand -base64 36
```

`RELAY_OWNER_PUBKEY` is different from the relay signing secret. It must be the 64-character hexadecimal public key of the Nostr identity that will own and administer the Buzz workspace.

> [!WARNING]
> Keep `BUZZ_RELAY_PRIVATE_KEY`, `BUZZ_GIT_HOOK_HMAC_SECRET`, database credentials, Redis credentials, S3 credentials, and the owner identity credentials stable and securely backed up. Changing them after deployment can break identity, access, or stored-data behavior.

### 5. Configure the domain

Assign the following domain to the `relay` service:

```text
https://buzz.example.com:3000
```

Port `3000` is the internal application port. Users connect through the normal public URLs:

```text
https://buzz.example.com
wss://buzz.example.com
```

Coolify handles HTTPS certificates and reverse-proxy routing.

Only the Buzz relay should be publicly accessible. PostgreSQL, Redis, and RustFS should remain internal to the Compose stack.

### 6. Deploy

Save the environment variables and start the deployment.

On the first deployment, `BUZZ_AUTO_MIGRATE=true` allows Buzz to initialize and migrate the database automatically.

## Verify the installation

Open the application:

```text
https://buzz.example.com
```

Check the liveness endpoint:

```bash
curl -fsS https://buzz.example.com/_liveness
```

You can also inspect the service health and logs from the Coolify dashboard.

## Connect the desktop application

Download a packaged Buzz desktop application from the official [Buzz releases](https://github.com/block/buzz/releases).

Configure its relay URL as:

```text
wss://buzz.example.com
```

You can change the relay inside the desktop application or launch the client with the `BUZZ_RELAY_URL` environment variable.

Example on macOS or Linux:

```bash
BUZZ_RELAY_URL=wss://buzz.example.com open -a Buzz
```

The exact application launch command may vary by operating system and package format.

## Agents and CLI clients

Agents and command-line tools should use the same relay:

```env
BUZZ_RELAY_URL=wss://buzz.example.com
BUZZ_PRIVATE_KEY=YOUR_AGENT_OR_USER_PRIVATE_KEY
```

Each human or agent should have its own identity. Do not distribute the relay signing private key to clients or agents.

## Updating Buzz

This repository does not build Buzz from source. It pulls the image configured in `BUZZ_IMAGE`.

For testing:

```env
BUZZ_IMAGE=ghcr.io/block/buzz:main
```

For production, pin a release or immutable SHA tag when one is available:

```env
BUZZ_IMAGE=ghcr.io/block/buzz:sha-abcdef1
```

To update:

1. Change `BUZZ_IMAGE` in Coolify.
2. Review the upstream Buzz release notes.
3. Back up persistent data.
4. Redeploy the resource.
5. Verify the liveness endpoint and application logs.

Avoid relying permanently on the moving `main` tag in production.

## Persistent data and backups

The stack stores persistent data for:

- PostgreSQL
- Redis
- RustFS (S3 object storage)
- Buzz Git repositories

Back up all persistent volumes and the stable environment secrets.

At minimum, a recovery plan should include:

1. A PostgreSQL logical or volume-level backup
2. RustFS object data (`buzz-s3-data` volume)
3. Buzz Git data
4. Redis data, when needed for your recovery model
5. All stable secrets and owner identity information

Test restoration before relying on the deployment for important work.

## Security recommendations

- Never commit `.env`, passwords, private keys, API keys, or database dumps.
- Keep relay membership enforcement enabled for a private workspace.
- Use unique identities for humans, services, and agents.
- Pin immutable container image versions in production.
- Expose only the relay through Coolify.
- Keep PostgreSQL, Redis, RustFS, health, and metrics ports private unless you have a specific protected monitoring setup.
- Enable server firewalls and install Coolify and operating-system updates.
- Review upstream Buzz security notices before upgrades.
- Back up secrets in a password manager or secrets-management system.

## Coolify-specific notes

This deployment is intended to let Coolify manage:

- The Docker network
- HTTPS certificates
- Reverse-proxy routing
- Runtime environment variables
- Container restarts
- Persistent storage

A Coolify-adapted Compose file should avoid publishing database or storage ports directly on the host.

It should also avoid unnecessary custom Docker networks because Coolify creates an isolated network for the resource and attaches its proxy to that network.

## Troubleshooting

### Domain returns `502 Bad Gateway`

Confirm that the domain is assigned to the `relay` service and targets internal port `3000`:

```text
https://buzz.example.com:3000
```

Also confirm that the relay container is healthy and listening on `0.0.0.0:3000`.

### Desktop client cannot connect

Confirm that the client uses:

```text
wss://buzz.example.com
```

Do not use `ws://` for a public HTTPS deployment.

Check:

- DNS resolution
- TLS certificate status
- Relay logs
- Coolify proxy status
- Browser developer tools for WebSocket errors

### Database migration fails

Confirm:

```env
BUZZ_AUTO_MIGRATE=true
```

Then review the relay and PostgreSQL logs. Do not repeatedly delete volumes to fix migration problems unless you are intentionally discarding all data.

### Authentication or membership is denied

Verify that:

- `RELAY_OWNER_PUBKEY` is a valid 64-character hexadecimal Nostr public key
- The desktop client is using the corresponding owner identity
- `BUZZ_REQUIRE_RELAY_MEMBERSHIP` is configured as intended
- The owner and member records were initialized successfully
- The relay signing key has not changed unexpectedly

### Deployment works locally but not through Coolify

Look for Compose settings that publish ports directly or define custom networks. Coolify should normally provide external routing and resource networking.

## Upstream documentation

- [Buzz repository](https://github.com/block/buzz)
- [Buzz releases](https://github.com/block/buzz/releases)
- [Buzz production Compose deployment](https://github.com/block/buzz/tree/main/deploy/compose)
- [Coolify Docker Compose documentation](https://coolify.io/docs/applications/build-packs/docker-compose)
- [Coolify domain documentation](https://coolify.io/docs/knowledge-base/domains)
- [Coolify environment-variable documentation](https://coolify.io/docs/knowledge-base/environment-variables)

## Disclaimer

This repository is provided as-is. Buzz is under active development, and its container image, environment variables, database migrations, authentication behavior, and deployment requirements may change.

Review the upstream project before deploying updates or using the system for sensitive or business-critical workloads.

Buzz and its trademarks belong to their respective owners.
