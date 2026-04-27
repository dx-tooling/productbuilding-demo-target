FROM nginx:1-alpine

COPY .productbuilding/preview/nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 8080
