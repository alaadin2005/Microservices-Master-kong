FROM node:22-alpine AS base

WORKDIR /app

RUN apk add --no-cache libc6-compat openssl

# ---------------------
# deps
# ---------------------
FROM base AS deps

WORKDIR /app

COPY package.json yarn.lock ./
COPY prisma ./prisma/

RUN yarn install --frozen-lockfile

# ---------------------
# builder
# ---------------------
FROM base AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV PRISMA_CLI_QUERY_ENGINE_TYPE=binary

RUN node node_modules/.bin/prisma generate
RUN yarn proto:generate
RUN yarn build
