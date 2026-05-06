FROM node:22-alpine

WORKDIR /app
COPY . .
RUN yarn install
RUN npx prisma generate
RUN yarn build
CMD ["yarn","start"]
