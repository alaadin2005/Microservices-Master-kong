FROM node:18-alpine

WORKDIR /app

RUN apk add --no-cache libc6-compat openssl

COPY package.json yarn.lock ./

RUN yarn install

COPY . .

RUN npx prisma generate

RUN yarn build

CMD ["node", "dist/main.js"]
