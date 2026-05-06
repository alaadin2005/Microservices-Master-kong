FROM node:18-alpine

WORKDIR /app

# Install required libs for Prisma
RUN apk add --no-cache libc6-compat openssl

# Copy dependencies first
COPY package.json yarn.lock ./

# Install dependencies (THIS creates prisma binary)
RUN yarn install

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

CMD ["yarn", "start"]
