# ── Build stage ──────────────────────────────────────────────────────────────
FROM oven/bun:1-alpine AS builder

WORKDIR /app

COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile

COPY tsconfig.json ./
COPY src/ ./src/

RUN bun run build

# ── Production stage ────────────────────────────────────────────────────────
FROM node:20-alpine AS runner

WORKDIR /app

RUN apk update && apk add --no-cache curl && rm -rf /var/cache/apk/*

# Security: run as non-root
RUN addgroup -g 1001 -S collab && adduser -S collab -u 1001

COPY package.json bun.lock* ./
RUN npm install --omit=dev

COPY --from=builder /app/dist ./dist

ENV NODE_ENV=production \
    COLLAB_PORT=1234 \
    PORT=1234

EXPOSE 1234

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:1234/health || exit 1

USER collab

CMD ["node", "dist/index.js"]
