#FROM node:22-alpine
#
#WORKDIR /app
#
#COPY package*.json ./
#
#RUN npm install
#
#COPY backend ./backend 
#
#EXPOSE 5000
#
#CMD ["npm","start"]

## ================================ Stage 1 

FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm install

## Frontend package files

COPY frontend/package*.json ./frontend/

RUN npm install --prefix frontend

COPY frontend ./frontend

COPY backend ./backend

RUN npm run build --prefix frontend

## ================================ Stage 2

FROM node:22-alpine

WORKDIR /app

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/frontend/dist ./frontend/dist
COPY --from=builder /app/backend ./backend

EXPOSE 5000

CMD ["npm","start"] 
