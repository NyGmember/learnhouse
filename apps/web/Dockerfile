FROM oven/bun:1-slim AS base

# Stage 1: Install dependencies only when needed
FROM base AS deps
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
WORKDIR /app

COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile

# Stage 2: Build Next.js application
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED=1 \
    NODE_ENV=production

# Support build-time public arguments
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_COLLAB_WS_URL
ARG NEXT_PUBLIC_LEARNHOUSE_OSS=true
ARG LEARNHOUSE_PUBLIC=false

ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL} \
    NEXT_PUBLIC_COLLAB_WS_URL=${NEXT_PUBLIC_COLLAB_WS_URL} \
    NEXT_PUBLIC_LEARNHOUSE_OSS=${NEXT_PUBLIC_LEARNHOUSE_OSS}

RUN rm -f .env*

# Remove Enterprise Edition folder for public/OSS builds
RUN if [ "$LEARNHOUSE_PUBLIC" = "true" ]; then rm -rf /app/ee; fi

RUN bun run build

# Stage 3: Production runner using lightweight Node.js
FROM node:20-alpine AS runner
WORKDIR /app

RUN apk update && apk add --no-cache curl && rm -rf /var/cache/apk/*

ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    PORT=3000 \
    HOSTNAME=0.0.0.0

RUN addgroup --system --gid 1001 nodejs \
    && adduser --system --uid 1001 nextjs

COPY --from=builder --chown=nextjs:nodejs /app/public ./public

RUN mkdir .next && chown nextjs:nodejs .next

# Copy Next.js standalone output
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Server wrapper for runtime environment variable injection
COPY --chown=nextjs:nodejs server-wrapper.js ./
RUN chmod +x server-wrapper.js

COPY --chown=nextjs:nodejs docker-entrypoint.sh ./
RUN chmod +x docker-entrypoint.sh

RUN rm -f public/runtime-config.js && touch runtime-config.json && chown nextjs:nodejs runtime-config.json

USER nextjs

EXPOSE 3000

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:3000/ || exit 1

ENTRYPOINT ["./docker-entrypoint.sh"]
