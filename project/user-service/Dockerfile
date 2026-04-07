FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev

FROM node:22-alpine
RUN apk upgrade --no-cache zlib && \
    rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/bin/npm \
           /usr/local/bin/npx \
           /opt/yarn*
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3001
CMD ["node", "app.js"]
