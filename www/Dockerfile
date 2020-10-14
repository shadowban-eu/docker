FROM nginx:alpine

RUN mkdir /data/nginx/cache -p
RUN mkdir /var/sites -p

ADD nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
