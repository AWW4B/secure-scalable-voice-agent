# =============================================================================
# frontend/Dockerfile
# Multi-stage build:
#   Stage 1 (builder) — installs deps + builds static files with Vite
#   Stage 2 (runner)  — tiny nginx image serving the /dist output
#
# For local dev, use docker-compose "frontend-dev" service instead,
# which mounts src/ as a volume and runs `npm run dev` with HMR.
# =============================================================================

# ---- Stage 1: Build ----
FROM node:20-alpine AS builder

WORKDIR /app

# Copy manifests first for layer caching
COPY package*.json ./

# Install all dependencies (including devDependencies for the build)
RUN npm ci

# Copy source
COPY . .

# Build production bundle
RUN npm run build

# ---- Stage 2: Serve ----
FROM nginx:1.25-alpine AS runner

# Remove default nginx page
RUN rm -rf /usr/share/nginx/html/*

# Copy built assets from builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Serve SPA — redirect 404s to index.html (for React Router / hash routing)
RUN echo 'server { \
    listen 80; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { try_files $uri $uri/ /index.html; } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
