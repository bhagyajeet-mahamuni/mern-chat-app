#FROM node:22-alpine
#
#WORKDIR /app
#
#COPY package*.json ./
#
#RUN npm install
#
#COPY . .
#
#RUN npm run build
#
#EXPOSE 4173
#
##CMD ["npm","run","preview"]
#CMD ["npm","run","preview","--","--host","0.0.0.0","--port","4173"]

##============================== Stage 1 starts

FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

##============================= Stage 2 starts

FROM nginx:alpine AS runner

COPY --from=builder /app/dist /usr/share/nginx/html

COPY nginx/default.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
