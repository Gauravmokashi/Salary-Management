# Deployment

## Live deployment

- **Frontend:** [https://salary-management-fe.onrender.com](https://salary-management-fe.onrender.com)
- **API health:** [https://salary-management-bs10.onrender.com/api/health](https://salary-management-bs10.onrender.com/api/health)

The browser calls the Next.js app on one origin. Next.js proxies `/api/*` to NestJS so the `SameSite=Strict` session cookie works across the separate services.

## Render configuration

Both services use the repository root as their working directory and deploy from `main`.

### API web service

| Setting | Value |
|---|---|
| Build | `npm ci --include=dev --legacy-peer-deps && npm run prisma:generate --workspace backend && npm run build --workspace backend` |
| Start | `npm run db:deploy --workspace backend && npm run start:prod --workspace backend` |
| Health check | `/api/health` |
| `DATABASE_URL` | Neon **direct** connection URI for the intended project, branch, and database |
| `JWT_SECRET` | Random secret of at least 32 bytes; store it only in Render's environment settings |
| `CORS_ORIGINS` | `https://salary-management-fe.onrender.com` |
| `NODE_ENV` | `production` |

The Prisma schema uses `DATABASE_URL` for the app and CLI. The start command applies committed migrations before starting NestJS, so the API uses a direct Neon connection rather than a pooled endpoint for this deployment. The API and seed must target the same Neon project, branch, and database.

Render's separate pre-deploy command is unavailable on its free web-service plan, so migrations run in the start command here. This is the configuration used for the current deployment.

### Frontend web service

| Setting | Value |
|---|---|
| Build | `npm ci --include=dev --legacy-peer-deps && npm run build --workspace frontend` |
| Start | `npm run start --workspace frontend` |
| `API_PROXY_TARGET` | `https://salary-management-bs10.onrender.com` (no `/api`) |

`API_PROXY_TARGET` is read by `next.config.mjs` at build time. Redeploy the frontend after changing it.

The free Render web-service plan spins services down after inactivity, so the first request after idle can be slow. It is suitable for a demo, not a production payroll system. See [Render's free service limitations](https://render.com/docs/free).

## Seed and bootstrap

Run one-off setup from the repository root with `DATABASE_URL` set to the Neon direct URI for the same project, branch, and database used by the API:

```bash
npm run db:seed --workspace backend
```

The deterministic seed creates 10,000 synthetic employee records in batches and is safe to rerun. Existing employee codes are skipped.

To create the first HR manager, put `INITIAL_HR_EMAIL` and a unique 12–72 byte `INITIAL_HR_PASSWORD` in the ignored local `backend/.env`, with `NODE_ENV=development` and the same `DATABASE_URL`, then run:

```bash
npm run auth:bootstrap --workspace backend
```

Remove the bootstrap email and password from the local environment file after the account is created. Never commit `.env` files or paste credentials into documentation.

## Before using real salary data

This is a synthetic-data assessment demo. Before real use, add operational controls such as login rate limiting, SSO or password recovery, backups and point-in-time recovery, and centralized logging. Review security and privacy requirements for the organization.
