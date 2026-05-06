# -----------------
#   BASE IMAGE  
# -----------------
    # Multi-stage build for optimal image size
    FROM node:22.21.1-alpine AS base

RUN apk add --no-cache libc6-compat wget openssl

# ---------------------
#   Dependencies  IMAGE  
# ---------------------
# Install dependencies only when needed
FROM base AS deps

WORKDIR /app

# Copy package files
COPY package.json yarn.lock* ./
COPY prisma ./prisma/


# Install dependencies
RUN yarn install --frozen-lockfile --production=false





# -----------------
#   Builder IMAGE  
# -----------------
# Build the source code
FROM base AS builder

WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN ulimit -n 65536

# Generate Prisma client and build application
RUN yarn prisma:generate
RUN yarn proto:generate
RUN yarn build




# -------------------------------------
# Production runner (smallest possible)
# -------------------------------------
# Production image, copy all the files and run the app
FROM base AS runner

WORKDIR /app

# Create non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nestjs

# Copy necessary files from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/src/protos ./src/protos
COPY --from=builder /app/package.json ./package.json

# Copy node_modules from deps stage (production only)
COPY --from=deps /app/node_modules ./node_modules

# Change ownership to nestjs user
RUN chown -R nestjs:nodejs /app
USER nestjs

# Expose ports for HTTP and gRPC
EXPOSE 9001 50051

CMD ["yarn", "start"]
