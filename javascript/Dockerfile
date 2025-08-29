# See https://docs.docker.com/engine/reference/builder/
# Use a specific Node.js LTS version as base image
FROM node:18.20.3-alpine3.19

# Set working directory
WORKDIR /app

# Copy only package.json and package-lock.json first for dependency install
COPY package.json package-lock.json ./

# Install dependencies with audit fix for vulnerabilities
RUN npm install --omit=dev && npm audit fix --force

# Copy the rest of the application source code
COPY src ./src
COPY .env* ./

# Create a non-root user and switch to it
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Expose the port the app runs on
EXPOSE 3000

# Healthcheck to ensure the app is running
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

# Start the application
CMD ["npm", "start"]
