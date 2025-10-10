#!/usr/bin/env bash
set -euo pipefail

ROOT="28pellets"
ZIP="${ROOT}.zip"

# Colors for output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}🚀 Starting 28pellets project scaffold...${NC}"

# Clean up previous run
if [ -d "$ROOT" ] || [ -f "$ZIP" ]; then
    echo -e "${YELLOW}⚠️  Cleaning up previous build...${NC}"
    rm -rf "$ROOT" "$ZIP"
fi

# Create core project directories
echo "📁 Creating directory structure..."
mkdir -p "$ROOT"
mkdir -p "$ROOT/src/app/api/"{users,verify,health}
mkdir -p "$ROOT/src/app/api/admin/verify/"{list,update}
mkdir -p "$ROOT/src/admin/verifications"
mkdir -p "$ROOT/src/"{db,lib,emails,components}
mkdir -p "$ROOT/public"

# Create package.json
cat > "$ROOT/package.json" <<'JSON'
{
  "name": "28pellets",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "db:generate": "drizzle-kit generate",
    "db:push": "npx tsx src/db/pushAndSeed.ts",
    "db:seed": "npx tsx src/db/seed.ts",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "drizzle-orm": "^0.33.0",
    "next-auth": "^5.0.0-beta.20",
    "@auth/drizzle-adapter": "^1.4.0",
    "postgres": "^3.4.0",
    "stripe": "^16.0.0",
    "zod": "^3.23.0",
    "resend": "^4.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "drizzle-kit": "^0.24.0",
    "tsx": "^4.16.0",
    "typescript": "^5.5.0",
    "eslint": "^8.57.0",
    "eslint-config-next": "^14.2.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0"
  }
}
JSON

# Create TypeScript config
cat > "$ROOT/tsconfig.json" <<'TS'
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
TS

# Create Next.js config
cat > "$ROOT/next.config.js" <<'NEXTJS'
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: {
      bodySizeLimit: '2mb',
    },
  },
}

module.exports = nextConfig
NEXTJS

# Create Tailwind config
cat > "$ROOT/tailwind.config.js" <<'TAILWIND'
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
TAILWIND

# Create PostCSS config
cat > "$ROOT/postcss.config.js" <<'POSTCSS'
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
POSTCSS

# Create environment files
cat > "$ROOT/.env.example" <<'ENV'
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/28pellets

# Auth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PUBLISHABLE_KEY=pk_test_xxx

# Email (Resend)
RESEND_API_KEY=re_xxx
ENV

cat > "$ROOT/.env.local" <<'ENV'
# Copy values from .env.example and fill in real values
DATABASE_URL=postgresql://user:pass@localhost:5432/28pellets
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=change-me-in-production
ENV

# Create gitignore
cat > "$ROOT/.gitignore" <<'GITIGNORE'
# Dependencies
node_modules
.pnp
.pnp.js

# Testing
coverage

# Next.js
.next
out
build
dist

# Misc
.DS_Store
*.pem

# Debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Local env files
.env*.local
.env

# Vercel
.vercel

# TypeScript
*.tsbuildinfo
next-env.d.ts

# Drizzle
drizzle
GITIGNORE

# Create Drizzle config
cat > "$ROOT/drizzle.config.ts" <<'DRIZZLE'
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
DRIZZLE

# Create database schema
cat > "$ROOT/src/db/schema.ts" <<'SCHEMA'
import { pgTable, varchar, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: varchar("id", { length: 255 }).primaryKey(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  name: varchar("name", { length: 255 }),
  emailVerified: timestamp("email_verified"),
  image: text("image"),
  role: varchar("role", { length: 50 }).default("user"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const verifications = pgTable("verifications", {
  id: varchar("id", { length: 255 }).primaryKey(),
  userId: varchar("user_id", { length: 255 }).references(() => users.id),
  status: varchar("status", { length: 50 }).default("pending"),
  submittedAt: timestamp("submitted_at").defaultNow(),
  reviewedAt: timestamp("reviewed_at"),
  notes: text("notes"),
});
SCHEMA

# Create database connection
cat > "$ROOT/src/db/index.ts" <<'DBINDEX'
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

const connectionString = process.env.DATABASE_URL!;
const client = postgres(connectionString);

export const db = drizzle(client, { schema });
DBINDEX

# Create health check API
cat > "$ROOT/src/app/api/health/route.ts" <<'HEALTH'
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({ 
    status: 'ok', 
    timestamp: new Date().toISOString() 
  });
}
HEALTH

# Create root layout
cat > "$ROOT/src/app/layout.tsx" <<'LAYOUT'
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: '28pellets',
  description: 'Your application description',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
LAYOUT

# Create global styles
cat > "$ROOT/src/app/globals.css" <<'GLOBALS'
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --foreground-rgb: 0, 0, 0;
  --background-rgb: 255, 255, 255;
}

body {
  color: rgb(var(--foreground-rgb));
  background: rgb(var(--background-rgb));
}
GLOBALS

# Create home page
cat > "$ROOT/src/app/page.tsx" <<'PAGE'
export default function Home() {
  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-24">
      <h1 className="text-4xl font-bold mb-4">Welcome to 28pellets</h1>
      <p className="text-lg text-gray-600">Your Next.js application is ready!</p>
    </main>
  )
}
PAGE

# Create README
cat > "$ROOT/README.md" <<'MD'
# 28PELLETS

A Next.js application with authentication, database, and payment processing.

## Getting Started

1. **Install dependencies:**
   ```bash
   npm install
   ```

2. **Set up environment variables:**
   ```bash
   cp .env.example .env.local
   # Edit .env.local with your actual values
   ```

3. **Set up the database:**
   ```bash
   npm run db:generate
   npm run db:push
   ```

4. **Run the development server:**
   ```bash
   npm run dev
   ```

5. **Open your browser:**
   Navigate to [http://localhost:3000](http://localhost:3000)

## Project Structure

```
28pellets/
├── src/
│   ├── app/              # Next.js app directory
│   │   ├── api/          # API routes
│   │   └── admin/        # Admin pages
│   ├── components/       # React components
│   ├── db/               # Database schema and queries
│   ├── lib/              # Utility functions
│   └── emails/           # Email templates
├── public/               # Static files
└── drizzle/              # Database migrations
```

## Available Scripts

- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm run start` - Start production server
- `npm run lint` - Run ESLint
- `npm run type-check` - Run TypeScript compiler check
- `npm run db:generate` - Generate database migrations
- `npm run db:push` - Push schema and seed database
- `npm run db:studio` - Open Drizzle Studio

## Tech Stack

- **Framework:** Next.js 14
- **Database:** PostgreSQL with Drizzle ORM
- **Authentication:** NextAuth.js
- **Payments:** Stripe
- **Email:** Resend
- **Styling:** Tailwind CSS
- **Language:** TypeScript

## License

MIT
MD

# Create placeholder files for key directories
touch "$ROOT/src/lib/.gitkeep"
touch "$ROOT/src/emails/.gitkeep"
touch "$ROOT/src/components/.gitkeep"
touch "$ROOT/public/.gitkeep"

# Create the zip file
echo "📦 Creating zip archive..."
(cd "$(dirname "$ROOT")" && zip -qr "$ZIP" "$(basename "$ROOT")")

echo ""
echo -e "${GREEN}✅ Project scaffolded successfully!${NC}"
echo ""
echo "📊 Summary:"
echo "  - Project: $ROOT"
echo "  - Archive: $ZIP"
echo "  - Size: $(du -h "$ZIP" | cut -f1)"
echo ""
echo "Next steps:"
echo "  1. unzip $ZIP"
echo "  2. cd $ROOT"
echo "  3. npm install"
echo "  4. Copy .env.example to .env.local and fill in values"

echo "  5. npm run dev"
echo ""
ls -lh "$ZIP"
