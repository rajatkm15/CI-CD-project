FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/* 
COPY index.html styles.css /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
ENTRYPOINT ["nginx","-g","daemon off;"]
