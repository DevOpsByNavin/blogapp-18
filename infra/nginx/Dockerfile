FROM node:20-alpine As builder

WORKDIR /app
COPY services/frontend/package.json .
RUN npm install --legacy-peer-deps
COPY services/frontend .
RUN npm run builder


FROM node:20-alpine

WORKDIR /app
COPY infra/nginx/nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/dist /var/www/blog-app
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
