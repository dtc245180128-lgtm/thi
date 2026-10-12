FROM nginx:alpine
# Hardening: Chạy với user không phải root (non-root container)
RUN touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid /var/cache/nginx /var/log/nginx /usr/share/nginx/html
COPY index.html /usr/share/nginx/html/index.html
USER nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
