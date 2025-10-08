28PELLETS
#!/usr/bin/env bash
set -euo pipefail

ROOT="28pellets"
ZIPNAME="${ROOT}.zip"

if [ -d "$ROOT" ]; then
  echo "Directory '$ROOT' exists. Remove it or choose another location."
  exit 1
fi

echo "Creating project folder: $ROOT"
mkdir -p "$ROOT"

# helper to write files
write() {
  mkdir -p "$(dirname "$ROOT/$1")"
  cat > "$ROOT/$1" <<'EOF'
$2
EOF
}

echo "Writing files..."

# package.json
cat > "$ROOT/package.json" <<'JSON'
{
  "name": "28pellets",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "db:generate": "drizzle-kit generate",
    "db:push": "npx tsx src/db/pushAndSeed.ts",
    "db:seed": "npx tsx src/db/seed.ts",
    "lint": "next lint"
  },
  "dependencies": {
    "drizzle-orm": "latest",
    "drizzle-zod": "latest",
    "pg": "latest",
    "zod": "latest",
    "next": "latest",
    "react": "latest",
    "react-dom": "latest",
    "resend": "latest",
    "pusher": "latest",
    "uploadthing": "latest",
    "@uploadthing/react": "latest",
    "@faker-js/faker": "latest",
    "stripe": "latest",
    "next-auth": "latest"
  },
  "devDependencies": {
    "drizzle-kit": "latest",
    "tsx": "latest",
    "typescript": "latest"
  }
}
JSON

# tsconfig.json
cat > "$ROOT/tsconfig.json" <<'TS'
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "ES2022"],
    "module": "ESNext",
    "moduleResolution": "Node",
    "jsx": "preserve",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "incremental": true,
    "resolveJsonModule": true
  },
  "exclude": ["node_modules", ".next"]
}
TS

# next.config.mjs
cat > "$ROOT/next.config.mjs" <<'NEXT'
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: {
    appDir: true
  },
};

export default nextConfig;
NEXT

# drizzle.config.ts
cat > "$ROOT/drizzle.config.ts" <<'DRIZZLE'
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  driver: "pg",
  dbCredentials: {
    connectionString: process.env.DATABASE_URL ?? ""
  },
  verbose: true,
  strict: true,
});
DRIZZLE

# .env.example
cat > "$ROOT/.env.example" <<'ENV'
DATABASE_URL=postgresql://user:pass@localhost:5432/28pellets
NODE_ENV=development
ADMIN_EMAIL=admin@example.com

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Resend
RESEND_API_KEY=re_...

# Pusher
PUSHER_APP_ID=...
PUSHER_KEY=...
PUSHER_SECRET=...
PUSHER_CLUSTER=us2
NEXT_PUBLIC_PUSHER_KEY=...

# UploadThing
UPLOADTHING_KEY=...

# Optional: Upstash redis, SENTRY etc.
ENV

# README.md
cat > "$ROOT/README.md" <<'MD'
# 28PELLETS — Super Mega App

Quick start:

1. Copy `.env.example` -> `.env` and fill secrets.
2. Install dependencies: