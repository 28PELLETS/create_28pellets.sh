chmod +x create_28pellets.sh
./create_28pellets.sh
cd 28pellets
cp .env.example .env
# Edit .env to fill secrets
chmod +x launch-final.sh
./launch-final.sh --mode local --yes
cd 28pellets
cp .env.example .env
# Edit .env to fill secrets
chmod +x launch-final.sh
./launch-final.sh --mode local --yes
#!/usr/bin/env bash
set -euo pipefail

# create_28pellets.sh
# Creates a ready-to-run 28pellets project scaffold and zips it.
# Usage: ./create_28pellets.sh

ROOT="28pellets"
ZIP="${ROOT}.zip"

if [ -d "$ROOT" ]; then
  echo "Directory '$ROOT' already exists. Remove or move it before running this script."
  exit 1
fi

echo "Creating project directory: $ROOT"
mkdir -p "$ROOT"

write_file() {
  # $1 = path relative to ROOT, $2 = heredoc content
  mkdir -p "$(dirname "$ROOT/$1")"
  cat > "$ROOT/$1" <<'EOF'
'"$2"' EOF
}

# ---------- package.json ----------
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
    "db:seed": "npx tsx src/db/seed.ts"
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

# ---------- tsconfig.json ----------
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
    "resolveJsonModule": true
  },
  "exclude": ["node_modules", ".next"]
}
TS

# ---------- next.config.mjs ----------
cat > "$ROOT/next.config.mjs" <<'NEXT'
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: { appDir: true },
};
export default nextConfig;
NEXT

# ---------- drizzle.config.ts ----------
cat > "$ROOT/drizzle.config.ts" <<'DRIZZLE'
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  driver: "pg",
  dbCredentials: { connectionString: process.env.DATABASE_URL ?? "" },
  verbose: true,
  strict: true,
});
DRIZZLE

# ---------- .env.example ----------
cat > "$ROOT/.env.example" <<'ENV'
# 28PELLETS example env file - copy to .env and fill values
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

# Optional monitoring
SENTRY_DSN=
REDIS_URL=
ENV

# ---------- README.md ----------
cat > "$ROOT/README.md" <<'MD'
# 28PELLETS — Starter App

Quick start:

1. Copy `.env.example` to `.env` and fill values.
2. Install deps:
3. Generate & apply migrations:
4. Seed (dev):
5. Run:
Or use helper:
Notes:
- Fill external service keys (Stripe, Resend, Pusher, UploadThing).
- For money handling in prod use NUMERIC rather than real.
MD

# ---------- create directories ----------
mkdir -p "$ROOT/src/app/api/users"
mkdir -p "$ROOT/src/app/api/verify"
mkdir -p "$ROOT/src/app/api/admin/verify/list"
mkdir -p "$ROOT/src/app/api/admin/verify/update"
mkdir -p "$ROOT/src/app/api/health"
mkdir -p "$ROOT/src/app/admin/verifications"
mkdir -p "$ROOT/src/db"
mkdir -p "$ROOT/src/lib"
mkdir -p "$ROOT/src/emails"
mkdir -p "$ROOT/src/components"

# ---------- src/db/schema.ts ----------
cat > "$ROOT/src/db/schema.ts" <<'SCHEMA'
import { sql, index } from "drizzle-orm";
import {
  pgTable, text, varchar, integer, timestamp, boolean, jsonb, real, pgEnum
} from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

/* Enums */
export const verificationStatusEnum = pgEnum("verification_status", ["pending","verified","rejected"]);
export const tipStatusEnum = pgEnum("tip_status", ["pending","completed","failed"]);

/* Users */
export const users = pgTable("users", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  username: text("username").notNull().unique(),
  email: text("email").notNull().unique(),
  avatar: text("avatar"),
  bio: text("bio"),
  followerCount: integer("follower_count").notNull().default(0),
  dateOfBirth: timestamp("date_of_birth").notNull(),
  verificationDocument: text("verification_document"),
  verificationStatus: verificationStatusEnum("verification_status").notNull().default("pending"),
  isVerified: boolean("is_verified").notNull().default(false),
  verifiedAt: timestamp("verified_at"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
}, (t) => ({
  usernameIdx: index("users_username_idx").on(t.username),
  verificationStatusIdx: index("users_verification_status_idx").on(t.verificationStatus),
}));

/* Streams */
export const streams = pgTable("streams", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  userId: varchar("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  category: text("category"),
  thumbnail: text("thumbnail"),
  isLive: boolean("is_live").notNull().default(false),
  viewerCount: integer("viewer_count").notNull().default(0),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (t) => ({ userIdx: index("streams_user_id_idx").on(t.userId) }));

/* Chat messages */
export const chatMessages = pgTable("chat_messages", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  streamId: varchar("stream_id").notNull().references(() => streams.id, { onDelete: "cascade" }),
  userId: varchar("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  username: text("username").notNull(),
  message: text("message").notNull(),
  isModerator: boolean("is_moderator").notNull().default(false),
  isSubscriber: boolean("is_subscriber").notNull().default(false),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (t) => ({ streamIdx: index("chat_messages_stream_id_idx").on(t.streamId) }));

/* Tips */
export const tips = pgTable("tips", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  streamId: varchar("stream_id").notNull().references(() => streams.id, { onDelete: "cascade" }),
  fromUserId: varchar("from_user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  toUserId: varchar("to_user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  amount: real("amount").notNull(),
  message: text("message"),
  stripePaymentIntentId: text("stripe_payment_intent_id"),
  status: tipStatusEnum("status").notNull().default("pending"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (t) => ({ tipsStreamIdx: index("tips_stream_id_idx").on(t.streamId) }));

/* Verification logs */
export const verificationLogs = pgTable("verification_logs", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  adminId: varchar("admin_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  userId: varchar("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  action: text("action").notNull(),
  details: jsonb("details"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (t) => ({ userIdx: index("verification_logs_user_idx").on(t.userId) }));

/* Helpers and Zod schemas */
export function is18OrOlder(date: Date) {
  const today = new Date();
  const age = today.getFullYear() - date.getFullYear() - (today < new Date(today.getFullYear(), date.getMonth(), date.getDate()) ? 1 : 0);
  return age >= 18;
}

export const insertUserSchema = createInsertSchema(users).omit({
  id: true, followerCount: true, isVerified: true, verifiedAt: true, verificationStatus: true, createdAt: true, updatedAt: true,
}).extend({
  username: z.string().min(3).max(20).regex(/^[a-zA-Z0-9_]+$/),
  email: z.string().email(),
  dateOfBirth: z.union([z.string(), z.date()]).transform(v => typeof v === "string" ? new Date(v) : v).refine(d => !isNaN(d.getTime())).refine(is18OrOlder, { message: "You must be 18 or older" })
});
SCHEMA

# ---------- src/db/index.ts ----------
cat > "$ROOT/src/db/index.ts" <<'DBINDEX'
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { logger: process.env.NODE_ENV === "development", schema: { ...schema } });
DBINDEX

# ---------- src/db/seed.ts ----------
cat > "$ROOT/src/db/seed.ts" <<'SEED'
import { db } from "./index";
import { users } from "./schema";

export default async function seed() {
  console.log("Seeding users...");
  await db.delete(users);
  await db.insert(users).values({
    username: "admin",
    email: "admin@example.com",
    bio: "Admin user",
    dateOfBirth: new Date("1990-01-01"),
    isVerified: true,
    verificationStatus: "verified",
  });
  console.log("Seed complete.");
}

if (require.main === module) seed();
SEED

# ---------- src/lib/pusher.ts ----------
cat > "$ROOT/src/lib/pusher.ts" <<'PUSHER'
import Pusher from "pusher";
export const pusherServer = new Pusher({
  appId: process.env.PUSHER_APP_ID!,
  key: process.env.PUSHER_KEY!,
  secret: process.env.PUSHER_SECRET!,
  cluster: process.env.PUSHER_CLUSTER ?? "us2",
  useTLS: true,
});
PUSHER

# ---------- src/lib/resend.ts ----------
cat > "$ROOT/src/lib/resend.ts" <<'RESEND'
import { Resend } from "resend";
export const resend = new Resend(process.env.RESEND_API_KEY!);
RESEND

# ---------- src/emails/VerificationEmail.tsx ----------
cat > "$ROOT/src/emails/VerificationEmail.tsx" <<'EMAIL'
export default function VerificationEmail({ status }: { status: "verified" | "rejected" }) {
  return (
    <div style={{ fontFamily: "sans-serif", padding: 20 }}>
      <h2>{status === "verified" ? "✅ Verification Approved" : "❌ Verification Rejected"}</h2>
      <p>
        {status === "verified"
          ? "Your ID verification has been approved. Welcome!"
          : "Your ID verification was rejected. Please resubmit a clear ID."}
      </p>
    </div>
  );
}
EMAIL

# ---------- API: users register ----------
cat > "$ROOT/src/app/api/users/route.ts" <<'USERSAPI'
import { NextResponse } from "next/server";
import { db } from "@/db";
import { insertUserSchema, users, is18OrOlder } from "@/db/schema";
import { or, eq } from "drizzle-orm";

export async function POST(req: Request) {
  try {
    const body = await req.json();
    const data = insertUserSchema.parse(body);

    const existing = await db.query.users.findFirst({
      where: or(eq(users.username, data.username), eq(users.email, data.email)),
    });
    if (existing) return NextResponse.json({ error: "Username or email already exists" }, { status: 400 });

    if (!is18OrOlder(new Date(data.dateOfBirth))) {
      return NextResponse.json({ error: "You must be 18 or older" }, { status: 400 });
    }

    const [user] = await db.insert(users).values(data).returning();
    return NextResponse.json({ success: true, userId: user.id });
  } catch (err: any) {
    console.error(err);
    return NextResponse.json({ error: err?.issues?.[0]?.message ?? err.message ?? "Invalid input" }, { status: 400 });
  }
}
USERSAPI

# ---------- API: verify start ----------
cat > "$ROOT/src/app/api/verify/route.ts" <<'VERIFYAPI'
import { NextResponse } from "next/server";
import { db } from "@/db";
import { users, is18OrOlder } from "@/db/schema";
import { eq } from "drizzle-orm";
import { pusherServer } from "@/lib/pusher";

export async function POST(req: Request) {
  try {
    const { userId, dateOfBirth, documentUrl } = await req.json();
    if (!userId || !dateOfBirth || !documentUrl) return NextResponse.json({ error: "Missing fields" }, { status: 400 });

    const dob = new Date(dateOfBirth);
    if (!is18OrOlder(dob)) return NextResponse.json({ error: "Must be 18+" }, { status: 403 });

    const [updated] = await db.update(users).set({
      dateOfBirth: dob,
      verificationDocument: documentUrl,
      verificationStatus: "pending",
      isVerified: false,
      updatedAt: new Date(),
    }).where(eq(users.id, userId)).returning();

    try { await pusherServer.trigger("verifications", "new", { userId }); } catch (e) { console.warn("pusher fail", e); }

    return NextResponse.json({ success: true, user: updated });
  } catch (e: any) {
    console.error(e);
    return NextResponse.json({ error: "Failed to start verification" }, { status: 500 });
  }
}
VERIFYAPI

# ---------- API: admin list ----------
cat > "$ROOT/src/app/api/admin/verify/list/route.ts" <<'ADMINLISTAPI'
import { NextResponse } from "next/server";
import { db } from "@/db";
import { users } from "@/db/schema";
import { eq } from "drizzle-orm";

export async function GET() {
  // NOTE: production must check session/role
  const pending = await db.select().from(users).where(eq(users.verificationStatus, "pending"));
  return NextResponse.json(pending);
}
ADMINLISTAPI

# ---------- API: admin update ----------
cat > "$ROOT/src/app/api/admin/verify/update/route.ts" <<'ADMINUPDATEAPI'
import { NextResponse } from "next/server";
import { db } from "@/db";
import { users, verificationLogs } from "@/db/schema";
import { eq } from "drizzle-orm";
import { resend } from "@/lib/resend";
import VerificationEmail from "@/emails/VerificationEmail";
import { pusherServer } from "@/lib/pusher";

export async function POST(req: Request) {
  try {
    const { userId, action, reason } = await req.json();
    if (!userId || !["approve","reject"].includes(action)) return NextResponse.json({ error: "Invalid" }, { status: 400 });

    const status = action === "approve" ? "verified" : "rejected";
    const isVerified = action === "approve";

    const [user] = await db.select().from(users).where(eq(users.id, userId)).limit(1);
    if (!user) return NextResponse.json({ error: "User not found" }, { status: 404 });

    await db.transaction(async (tx) => {
      await tx.update(users).set({
        verificationStatus: status,
        isVerified,
        verifiedAt: isVerified ? new Date() : null,
        updatedAt: new Date(),
      }).where(eq(users.id, userId));

      await tx.insert(verificationLogs).values({
        adminId: "system",
        userId,
        action,
        details: reason ? { reason } : null,
      });
    });

    if (user.email) {
      try {
        await resend.emails.send({
          from: "no-reply@28pellets.app",
          to: user.email,
          subject: action === "approve" ? "✅ Verification Approved" : "❌ Verification Rejected",
          react: VerificationEmail({ status: action === "approve" ? "verified" : "rejected" }),
        });
      } catch (e) { console.warn("email fail", e); }
    }

    try { await pusherServer.trigger("verifications", "updated", { userId, action }); } catch (e) { console.warn("pusher fail", e); }

    return NextResponse.json({ success: true });
  } catch (e: any) {
    console.error(e);
    return NextResponse.json({ error: "Failed to update" }, { status: 500 });
  }
}
ADMINUPDATEAPI

# ---------- API: health ----------
cat > "$ROOT/src/app/api/health/route.ts" <<'HEALTH'
export async function GET() {
  return new Response(JSON.stringify({ ok: true }), { status: 200 });
}
HEALTH

# ---------- admin page ----------
cat > "$ROOT/src/app/admin/verifications/page.tsx" <<'ADMINTX'
'use client';
import { useEffect, useState } from "react";

export default function Page() {
  const [pending, setPending] = useState<any[]>([]);
  useEffect(()=>{ fetch("/api/admin/verify/list").then(r=>r.json()).then(setPending); }, []);
  return (
    <div style={{padding:20}}>
      <h1>Pending Verifications</h1>
      <ul>
        {pending.map(u => (<li key={u.id}>{u.username} - {u.email} - <a href={u.verificationDocument} target="_blank" rel="noreferrer">doc</a></li>))}
      </ul>
    </div>
  );
}
ADMINTX

# ---------- component: VerificationForm ----------
cat > "$ROOT/src/components/VerificationForm.tsx" <<'VFORM'
"use client";
import { useState } from "react";

export default function VerificationForm({ userId }: { userId: string }) {
  const [dob, setDob] = useState("");
  const [doc, setDoc] = useState("");
  const [loading, setLoading] = useState(false);
  async function submit(e: any) {
    e.preventDefault();
    if (!dob || !doc) return alert("Date and doc required");
    setLoading(true);
    const res = await fetch("/api/verify", { method: "POST", headers: { "Content-Type":"application/json" }, body: JSON.stringify({ userId, dateOfBirth: dob, documentUrl: doc }) });
    setLoading(false);
    const data = await res.json();
    if (!res.ok) return alert(data.error || "Failed");
    alert("Verification started");
  }
  return (
    <form onSubmit={submit}>
      <label>Date of birth <input type="date" value={dob} onChange={e=>setDob(e.target.value)} /></label>
      <label>Document URL <input value={doc} onChange={e=>setDoc(e.target.value)} /></label>
      <button disabled={loading}>{loading ? "..." : "Submit"}</button>
    </form>
  );
}
VFORM

# ---------- launch-final.sh ----------
cat > "$ROOT/launch-final.sh" <<'LAUNCH'
#!/usr/bin/env bash
set -euo pipefail
MODE="local"
SKIP_SEED=false
ASSUME_YES=false
HEALTH_PATH="/api/health"
if [ ! -f .env ]; then
  if [ -f .env.example ]; then cp .env.example .env; echo ".env created from example; edit it before running."; fi
fi
echo "Installing deps..."
npm ci
echo "Running drizzle migrations..."
npx drizzle-kit generate || true
npx drizzle-kit push || true
if [ "$SKIP_SEED" = false ]; then
  echo "Seeding database..."
  npx tsx src/db/seed.ts || true
fi
echo "Building..."
npm run build || true
if [ "$MODE" = "local" ]; then
  echo "Starting Next.js (dev)..."
  npm run dev &
  APP_PID=$!
  echo "App PID: $APP_PID"
  echo "Waiting for health..."
  for i in {1..30}; do
    if curl -sSf http://localhost:3000${HEALTH_PATH} >/dev/null 2>&1; then
      echo "Health ok"
      break
    fi
    sleep 1
  done
fi
echo "Launch script finished."
LAUNCH

chmod +x "$ROOT/launch-final.sh"

# ---------- pushAndSeed (helper) ----------
cat > "$ROOT/src/db/pushAndSeed.ts" <<'PUSHSEED'
import { execSync } from "child_process";
import seed from "./seed";
try {
  execSync("npx drizzle-kit generate", { stdio: "inherit" });
  execSync("npx drizzle-kit push", { stdio: "inherit" });
} catch(e) {
  console.warn("Drizzle push/generate failed or skipped", e);
}
seed().then(()=>console.log("Seeded")).catch(e=>console.error(e));
PUSHSEED

# ---------- Make zip ----------
echo "Creating zip archive: $ZIP"
if command -v zip >/dev/null 2>&1; then
  zip -qr "$ZIP" "$ROOT"
  echo "Created $ZIP"
else
  echo "zip not available; skipping archive creation."
fi

echo
echo "Done. Project created in: $ROOT"
echo "Next steps:"
echo "  cd $ROOT"
echo "  cp .env.example .env"
echo "  Edit .env with your services (DATABASE_URL, STRIPE, RESEND, PUSHER, etc.)"
echo "  chmod +x launch-final.sh"
echo "  ./launch-final.sh --mode local --yes"
echo
echo "If you want, I can now:"
echo " - Add Sentry and Upstash rate limiter wiring"
echo " - Replace 'real' money type with NUMERIC in schema"
echo " - Create a GitHub repo and push the generated project"
echo
exit 0
# 1️⃣ Give script permission to run
chmod +x create_28pellets.sh

# 2️⃣ Run it to create your full project
./create_28pellets.sh

# 3️⃣ Move into your new app folder
cd 28pellets

# 4️⃣ Create your env file and fill in secrets
cp .env.example .env
# Edit .env — add your DATABASE_URL, Stripe, Pusher, Resend, etc.

# 5️⃣ Give launch script permission and run
chmod +x launch-final.sh
./launch-final.sh --mode local --yes

