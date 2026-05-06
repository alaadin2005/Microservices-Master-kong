# =========================
# 1. BASE IMAGE (STABLE)
# =========================
FROM node:22-bullseye AS base

WORKDIR /app

# System dependencies (Prisma + SSL + grpc support)
RUN apt-get update && apt-get install -y \
    openssl \
    && rm -rf /var/lib/apt/lists/*

# Disable git hooks (important for Docker)
ENV HUSKY=0
ENV NODE_ENV=production

# =========================
# 2. DEPENDENCIES STAGE
# =========================
FROM base AS deps

WORKDIR /app

COPY package.json yarn.lock ./
COPY prisma ./prisma/

# Install dependencies safely (no scripts)
RUN yarn install --frozen-lockfile --ignore-scripts

# =========================
# 3. BUILD STAGE
# =========================
FROM base AS builder

WORKDIR /app

ENV HUSKY=0

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma client safely
RUN npx prisma generate

# Generate protobuf files
RUN yarn run proto:generate

# Build NestJS app
RUN yarn build

# =========================
# 4. RUNTIME STAGE
# =========================
FROM node:22-bullseye AS runner

WORKDIR /app

ENV NODE_ENV=production

# Create non-root user (security best practice)
RUN useradd -m appuser

# Copy only required artifacts
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/package.json ./package.json
COPY --from=deps /app/node_modules ./node_modules

# Fix permissions
RUN chown -R appuser:appuser /app
USER appuser

# Expose service port
EXPOSE 9001

# Start app
CMD ["node", "dist/main.js"]
