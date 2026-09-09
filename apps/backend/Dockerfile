# Backend image. Build context must be the repo root (monorepo workspace).
FROM node:22-slim AS base
WORKDIR /app

# ffmpeg is required to stitch template clips together + generate thumbnails.
RUN apt-get update \
  && apt-get install -y --no-install-recommends ffmpeg \
  && rm -rf /var/lib/apt/lists/*

# Install dependencies for the whole workspace (cached when manifests are unchanged).
COPY package.json package-lock.json ./
COPY apps/backend/package.json apps/backend/package.json
COPY packages/db/package.json packages/db/package.json
COPY packages/typescript-config/package.json packages/typescript-config/package.json
COPY packages/eslint-config/package.json packages/eslint-config/package.json
RUN npm ci

# Copy source.
COPY packages/db packages/db
COPY packages/typescript-config packages/typescript-config
COPY packages/eslint-config packages/eslint-config
COPY apps/backend apps/backend

# Ensure any local host .env files are not baked into the image
RUN rm -f /app/.env /app/packages/db/.env /app/apps/backend/.env /app/apps/frontend/.env

# Generate the Prisma client.
RUN cd packages/db && npx prisma generate

ENV NODE_ENV=production
EXPOSE 4000

# Sync the database schema, then start the server.
# `prisma db push` creates the schema directly from schema.prisma (no migration
# history needed at this stage). Switch to `prisma migrate deploy` once you commit
# migrations under packages/db/prisma/migrations.
#
# Run from packages/db so `npx prisma` resolves the project's pinned Prisma (it's a
# dependency there) rather than fetching a newer, incompatible Prisma from npm.
# DATABASE_URL is provided as a mounted `.env` file (k8s secret at
# /app/apps/backend/.env), so source it first; under docker-compose that file is
# absent (the `|| true` no-ops) and the real env vars are used instead.
CMD ["sh", "-c", "if [ -f /app/apps/backend/.env ]; then set -a; . /app/apps/backend/.env; set +a; fi; (cd packages/db && npx prisma db push --skip-generate) ; cd apps/backend && npm start"]
