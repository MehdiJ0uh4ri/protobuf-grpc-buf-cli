FROM nginx:1.25-alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/nginx.conf

# Create user and fix permissions for nginx-specific folders
RUN addgroup -g 1001 -S sso && adduser -u 1001 -S sso -G sso \
    && chown -R 1001:1001 /var/cache/nginx /var/log/nginx

EXPOSE 8081 8080
USER 1001
CMD ["nginx", "-g", "daemon off;"]