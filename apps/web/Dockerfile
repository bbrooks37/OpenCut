# apps/web/Dockerfile

FROM oven/bun:alpine AS base

# Install curl for health checks
RUN apk add --no-cache curl

# Declare DATABASE_URL as a build argument
ARG DATABASE_URL

# Install dependencies and build the application
FROM base AS builder

WORKDIR /app

# Set DATABASE_URL as an environment variable for the build stage
# This is crucial for Next.js's 'bun run build' to recognize the DB URL
ENV DATABASE_URL=${DATABASE_URL}
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Copy all package.json, bun.lock, turbo.json from the root
# and then copy all apps/ and packages/ directories.
# This ensures the full monorepo context is available for bun install and build.
COPY package.json bun.lock turbo.json ./
COPY apps/ apps/
COPY packages/ packages/

# Run bun install at the monorepo root to link workspace dependencies
RUN bun install

# Run the build command for the web application from the monorepo root.
# TurboRepo will handle building dependencies like @opencut/db and @opencut/auth
# as part of the web app's build process.
# We explicitly filter for 'opencut' (the actual package name for apps/web)
WORKDIR /app
RUN bun run build --filter=opencut # <--- CHANGED: Filter by 'opencut' instead of 'web'

# Production image
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Make sure DATABASE_URL is also available at runtime
ENV DATABASE_URL=${DATABASE_URL}
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Add the nextjs and nodejs user/group
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy the built standalone Next.js application from the builder stage
# The .next/standalone directory contains all the necessary files, including
# compiled versions of your @opencut/db and @opencut/auth packages.
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/public ./apps/web/public
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static

# Ensure correct ownership for the entire application directory
# The standalone output copies everything into the root of the runner's WORKDIR (/app)
RUN chown -R nextjs:nodejs /app

USER nextjs

EXPOSE 3000

CMD ["bun", "apps/web/server.js"]
