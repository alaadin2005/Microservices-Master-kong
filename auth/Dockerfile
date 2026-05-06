FROM base AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Fix Prisma on Alpine
ENV PRISMA_CLI_QUERY_ENGINE_TYPE=binary

# Optional debug
RUN ls -la node_modules/.bin/

# Generate Prisma client
RUN npx prisma generate

# Continue build
RUN yarn proto:generate
RUN yarn build
