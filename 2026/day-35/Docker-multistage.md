FROM node:lts AS builder

WORKDIR /app

COPY package.json .

RUN npm install

FROM gcr.io/distroless/nodejs18 AS deployer

WORKDIR /app

COPY --from=builder /app /app
COPY . .
CMD ["app.js"]
