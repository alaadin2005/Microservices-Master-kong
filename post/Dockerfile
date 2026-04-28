# -----------------
#   BASE IMAGE  
# -----------------
FROM node:22.21.1-alpine AS base

RUN apk add --no-cache libc6-compat wget openssl

# ---------------------
#   Dependencies IMAGE
# ---------------------
FROM base AS deps

WORKDIR /app

# Copy package files first (for caching)
COPY package.json yarn.lock* ./
COPY prisma ./prisma/

# Install missing types explicitly
RUN yarn add -D @types/mime

# Install dependencies
RUN yarn install --frozen-lockfile --production=false


# -----------------
#   Builder IMAGE
# -----------------
FROM base AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN yarn prisma:generate
RUN yarn proto:generate
RUN yarn build


# ----------------------
#   Production runner
# ----------------------
FROM base AS runner

WORKDIR /app

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nestjs

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/src/protos ./src/protos
COPY --from=builder /app/package.json ./package.json

COPY --from=deps /app/node_modules ./node_modules

RUN chown -R nestjs:nodejs /app
USER nestjs

EXPOSE 9002 50052

CMD ["yarn", "start"]

