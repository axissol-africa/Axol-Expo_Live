# Axsol Expo review deployment

This repository contains deployment configuration only. The application images are private. It does not contain application source, build output, database contents or credentials.

## Required before Deploy

1. The web and API images are published privately and pinned by digest in `compose.yaml`. Give the Coolify server permission to pull them. `WEB_IMAGE` and `API_IMAGE` are optional overrides for a different verified release. The supplied release is source commit `6647dabab01bdcb22f702a244445902af351afcd`; unfinished local work is excluded.
2. Prepare a separate synthetic-data staging PostgreSQL database with the reviewed schemas, restricted roles and accounts. The application does not initialize the database or run migrations. An empty PostgreSQL service is insufficient. The private project operator must complete its database rollout procedure before this deployment is useful for login and console review.
3. Obtain a working HTTPS hostname for the preview. An existing delegated subdomain is sufficient; a new purchased domain is not required. DNS must reach the Coolify proxy and certificate issuance must work.

## Coolify setup

1. Authenticate the target server to the private registry under **Servers → your server → Registries** (or **Registries → Log in**, depending on version). For GHCR use a package-reading account and classic token with `read:packages`, with access to both private packages. Keep this credential in Coolify, not Git. Reviewers receive no registry credentials.
2. Create a resource from **Public Repository**, enter this deployment repository URL, select **Docker Compose**, base directory `/`, Compose location `/compose.yaml`. The services pull images; this repository has no application build context.
3. Enter the required database, authentication and origin values from `.env.example` using Coolify's environment settings. Compose interpolation happens before image pulls, so these values must be available to Coolify during deployment as well as at runtime. `PUBLIC_ORIGIN` must be the exact browser origin, e.g. `https://preview.example.com`, with no path, port suffix for internal routing, or trailing slash. `BETTER_AUTH_URL` and `WEB_ORIGIN` are intentionally derived from the same value.
4. Assign an HTTPS domain to **web**, targeting its internal port **3000**. Coolify's domain field can use `https://preview.example.com:3000`; visitors use `https://preview.example.com`. Leave **api** without a public domain. There are no host port mappings. Coolify's proxy routes to web; web calls api on the Compose network.
5. Deploy, then perform the checks below. Retain the previously working image digests for rollback. Do not delete database volumes to troubleshoot.

The default images target `linux/amd64`. If the server is ARM64, build/publish ARM64 images and set `DEPLOY_PLATFORM=linux/arm64`.

## Configuration

| Variable | Required value |
| --- | --- |
| `WEB_IMAGE`, `API_IMAGE` | Optional overrides; default to the published private image digests |
| `PUBLIC_ORIGIN` | Exact public HTTPS origin |
| `AUTH_DATABASE_URL` | Staging PostgreSQL URL for the restricted authentication role, not database owner |
| `APP_DATABASE_URL` | Same staging database through the separate `axis_api` role |
| `BETTER_AUTH_SECRET` | Persistent random secret of at least 32 characters; must match any provisioned MFA/session data |
| `AUTH_ADMIN_EMAIL` | The already-provisioned, verified platform administrator email |
| `AUTH_PROVIDER_KEY` | Alias matching the staged identity-provider links |
| `DEFAULT_COUNTRY` | `ZM` or `ZW`; default `ZM` |

Do not use `localhost` for an external database: inside the API container it means that container itself. Use a database hostname reachable from the deployment network. Do not expose PostgreSQL publicly just to make the connection work.

MFA remains required. Redis and Meilisearch are disabled because they are not required for this read-only review. The API liveness check verifies the process only; it does not prove database configuration, account setup or tenant access. Its infrastructure readiness endpoint is not a deployment gate in this configuration.

## Reviewer acceptance

- Homepage and `/login` load through HTTPS.
- The designated reviewer signs in with an individually provisioned account and completes MFA. Do not publish or share the operator's existing credentials.
- An authorized platform account can open `/admin` and `/backoffice`; an ordinary tenant owner is not automatically a platform administrator.
- A fixture tenant owner can open `/account/workspaces`, switch between its authorized tenants and see only the corresponding events.
- Logged-out access and access to another tenant are denied. Sign-out prevents reuse of the old session.
- Draft creation/editing remain disabled. A failing `/bootstrap` without a selected catalog is a known release limitation, not a reason to bypass access controls.

The backoffice's attendee, registration, inventory and printing operations remain browser-local demonstrations. They are not shared between reviewers' computers. This is a synthetic-data review deployment, not production acceptance.

## Visibility

Only the files in this repository are intended to be public. Keep private images private. A person with image-pull or server access can inspect executable JavaScript inside the images. Frontend JavaScript delivered to a browser is always inspectable.

## References

- [Coolify Compose deployments](https://coolify.io/docs/applications/builds/docker-compose)
- [Coolify registry authentication](https://coolify.io/docs/applications/builds/docker-registries)
- [Coolify domains and service ports](https://coolify.io/docs/core/networking/domains)
- [GitHub Container Registry permissions](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
