
# Multi-stage build for security and smaller image size
# Stage 1: Build stage
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
  adduser -S reactuser -u 1001


# Copy package files first for better caching
COPY package*.json ./

# Install dependencies with security optimizations
RUN npm install --no-audit --no-fund && \
  npm cache clean --force


# Copy source code
COPY . .

# Change ownership to non-root user
RUN chown -R reactuser:nodejs /app
USER reactuser

# Build the application
RUN npm run build

# Stage 2: Production stage
FROM nginx:1.25-alpine AS production

# Install security updates
RUN apk update && \
  apk upgrade && \
  apk add --no-cache ca-certificates && \
  rm -rf /var/cache/apk/*

# Create non-root user for nginx
RUN addgroup -g 1001 -S nginx_group && \
  adduser -S nginx_user -u 1001 -G nginx_group

# Remove default nginx configuration
RUN rm -rf /etc/nginx/conf.d/default.conf /var/www/html/*

# Create secure nginx configuration
COPY --chown=nginx_user:nginx_group <<EOF /etc/nginx/conf.d/default.conf
server {
  listen 8080;
  server_name localhost;
    
  # Security headers
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://fonts.googleapis.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self';" always;
    
  # Disable server tokens
  server_tokens off;
    
  # Set document root
  root /usr/share/nginx/html;
  index index.html;
    
  # Serve static files
  location / {
    try_files \$uri \$uri/ /index.html;
        
    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
      expires 1y;
      add_header Cache-Control "public, immutable";
    }
  }
    
  # Security: Deny access to hidden files
  location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
  }
    
  # Security: Deny access to backup files
  location ~ ~$ {
    deny all;
    access_log off;
    log_not_found off;
  }
    
  # Health check endpoint
  location /health {
    access_log off;
    return 200 "healthy\n";
    add_header Content-Type text/plain;
  }
}
EOF

# Copy built application from builder stage
COPY --from=builder --chown=nginx_user:nginx_group /app/build /usr/share/nginx/html

# Create nginx directories with correct permissions
RUN mkdir -p /var/cache/nginx /var/log/nginx /tmp/nginx && \
  chown -R nginx_user:nginx_group /var/cache/nginx /var/log/nginx /usr/share/nginx/html /tmp/nginx

# Create a custom nginx.conf that works with non-root user
COPY --chown=nginx_user:nginx_group <<EOF /etc/nginx/nginx.conf
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /tmp/nginx/nginx.pid;


  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  log_format main '\$remote_addr - \$remote_user [\$time_local] "\$request" '
          '\$status \$body_bytes_sent "\$http_referer" '
          '"\$http_user_agent" "\$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;

  sendfile on;
  tcp_nopush on;
  keepalive_timeout 65;
  types_hash_max_size 2048;

  include /etc/nginx/conf.d/*.conf;
}
EOF

# Switch to non-root user
USER nginx_user

# Expose non-privileged port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Start nginx with error handling
CMD ["nginx", "-g", "daemon off;"]
