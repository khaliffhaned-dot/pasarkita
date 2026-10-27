# guruku#!/bin/bash
set -e

# ============================================================
#  GURUKU — FULL FILE BUNDLE
# Creates the entire project structure in one shot
# ============================================================

REPO_NAME="guruku"
GITHUB_USER="khaliffhaned-dot"   # <-- CHANGE THIS

mkdir -p "$REPO_NAME" && cd "$REPO_NAME"
git init -b main

# ============================================================
# ROOT FILES
# ============================================================

cat > README.md << 'EOF'
# Guruku 🇲🇾

**Syok Sendiri, Syok Semua!**

A Malaysian social commerce app — TikTok Shop, but built for Malaysian culture.
Live selling, halal verification, Sabah & Sarawak spotlight, Manglish throughout.

## Stack
- **Frontend:** Flutter (mobile), React (seller portal + admin)
- **Backend:** Node.js + TypeScript on Alibaba Cloud ACK
- **Database:** PolarDB PostgreSQL + Tair Redis
- **Storage:** OSS + CDN
- **Live:** ApsaraVideo Live
- **AI:** PAI-EAS (Manglish NLP + recommendations)
- **Payments:** iPay88 (FPX, TNG, GrabPay, Boost, COD)

## Quick Start
See `docs/SETUP.md`

## Structure
- `services/` — backend microservices
- `frontend/` — Flutter + React apps
- `infrastructure/` — Terraform + Kubernetes
- `functions/` — Alibaba Function Compute
- `docs/` — setup, deployment, runbooks

## License
Proprietary — © PasarKita Sdn Bhd
EOF

cat > .gitignore << 'EOF'
# Node
node_modules/
dist/
*.log
.env
.env.*

# Flutter
.dart_tool/
.packages
build/
*.iml

# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfvars

# IDE
.vscode/
.idea/
.DS_Store

# Secrets
secrets/
*.pem
*.key
EOF

cat > LICENSE << 'EOF'
Copyright (c) 2025 PasarKita Sdn Bhd
All rights reserved.
EOF

# ============================================================
# DOCS
# ============================================================

mkdir -p docs

cat > docs/SETUP.md << 'EOF'
# PasarKita Setup Guide

## Prerequisites
- Alibaba Cloud account (international site)
- Terraform >= 1.5
- Docker
- Flutter SDK 3.2+
- Node.js 20+

## Steps
1. `cd infrastructure/terraform && terraform init && terraform apply`
2. `bash scripts/deploy.sh v1.0.0`
3. `cd frontend/mobile && flutter run`

Full guide in `docs/DEPLOYMENT.md`
EOF

cat > docs/ARCHITECTURE.md << 'EOF'
# Architecture

See the full architecture diagram in the project README.

Key services:
- API Gateway (Alibaba Cloud API Gateway)
- ACK cluster (Kubernetes) running Node.js microservices
- PolarDB PostgreSQL (primary + read replica)
- Tair Redis (cache + sessions)
- OSS + CDN (media)
- ApsaraVideo Live (streaming)
- Function Compute (image processing, notifications, halal verification)
- RocketMQ (event bus)
- PAI-EAS (AI recommendations + Manglish NLP)
EOF

cat > docs/DEPLOYMENT.md << 'EOF'
# Deployment

## Staging
Push to `develop` branch → auto-deploys to staging.

## Production
Tag a release: `git tag v1.0.0 && git push origin v1.0.0`
GitHub Actions deploys to production.

## Rollback
`kubectl rollout undo deployment/pasarkita-api -n pasarkita-prod`
EOF

cat > docs/MANGLISH-COPY.md << 'EOF'
# Manglish Copy Deck

## Pronouns
- **Default:** saya / awak (polite, safe)
- **Casual:** u / i (push notifications, live chat)
- **Never:** kau / aku (too informal for commerce)

## Key strings
- Add to cart: "Campak Dalam Beg"
- Buy now: "Beli Terus, Tak Payah Fikir"
- Sold out: "Alamak, Habis Sudah!"
- Empty cart: "Beg awak kosong lah. Jom pergi shopping!"
- Order placed: "Syok! Order dah masuk!"

See full deck in `frontend/mobile/lib/l10n/manglish.dart`
EOF

cat > docs/FREE-TIER.md << 'EOF'
# Alibaba Cloud Free Tier Plan

Month 1: RM 0 (all free trials)
Month 1.5-3: ~RM 120 (DB migration to serverless)
Month 3+: ~RM 250 (full paid)

See `infrastructure/terraform/` for free-tier-optimized configs.
EOF

# ============================================================
# BACKEND — services/api
# ============================================================

mkdir -p services/api/src/{config,routes,services,middleware,utils}
mkdir -p services/api/migrations

cat > services/api/package.json << 'EOF'
{
  "name": "pasarkita-api",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev src/index.ts",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3",
    "ioredis": "^5.3.2",
    "ali-oss": "^6.19.0",
    "@alicloud/fc2": "^2.6.0",
    "@alicloud/pop-core": "^1.7.12",
    "jsonwebtoken": "^9.0.2",
    "bcrypt": "^5.1.1",
    "zod": "^3.22.4",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "compression": "^1.7.4",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "@types/bcrypt": "^5.0.2",
    "@types/jsonwebtoken": "^9.0.5",
    "ts-node-dev": "^2.0.0"
  }
}
EOF

cat > services/api/tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}
EOF

cat > services/api/Dockerfile << 'EOF'
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
RUN apk add --no-cache tini
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs
EXPOSE 3000
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
EOF

cat > services/api/.env.example << 'EOF'
NODE_ENV=development
PORT=3000
ALIBABA_REGION=ap-southeast-3

POLARDB_ENDPOINT=
POLARDB_USER=
POLARDB_PASSWORD=

TAIR_HOST=
TAIR_PASSWORD=

OSS_REGION=oss-ap-southeast-3
OSS_MEDIA_BUCKET=pasarkita-media

JWT_SECRET=change-me-in-production

IPAY88_MERCHANT_CODE=
IPAY88_MERCHANT_KEY=

LIVE_PUSH_DOMAIN=push.pasarkita.my
LIVE_PULL_DOMAIN=pull.pasarkita.my
LIVE_AUTH_KEY=

PAI_EAS_ENDPOINT=
PAI_EAS_TOKEN=
EOF

cat > services/api/src/index.ts << 'EOF'
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import dotenv from 'dotenv';

dotenv.config();

import authRouter from './routes/auth';
import productRouter from './routes/products';
import orderRouter from './routes/orders';
import liveRouter from './routes/live';
import errorHandler from './middleware/error-handler';

const app = express();
const PORT = process.env.PORT || 3000;

app.use(helmet());
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') || '*' }));
app.use(compression());
app.use(express.json({ limit: '10mb' }));

app.get('/health', (req, res) => {
  res.json({ status: 'ok', service: 'pasarkita-api', timestamp: new Date().toISOString() });
});

app.use('/api/v1/auth', authRouter);
app.use('/api/v1/products', productRouter);
app.use('/api/v1/orders', orderRouter);
app.use('/api/v1/live', liveRouter);

app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`🚀 PasarKita API on port ${PORT}`);
});
EOF

cat > services/api/src/config/alibaba.ts << 'EOF'
import OSS from 'ali-oss';
import Redis from 'ioredis';
import { Pool } from 'pg';

export const ossClient = new OSS({
  region: process.env.OSS_REGION!,
  accessKeyId: process.env.ALIBABA_ACCESS_KEY_ID!,
  accessKeySecret: process.env.ALIBABA_ACCESS_KEY_SECRET!,
  bucket: process.env.OSS_MEDIA_BUCKET!,
});

export const polarDB = new Pool({
  host: process.env.POLARDB_ENDPOINT,
  port: 5432,
  database: 'pasarkita',
  user: process.env.POLARDB_USER,
  password: process.env.POLARDB_PASSWORD,
  ssl: { rejectUnauthorized: false },
  max: 20,
});

export const tair = new Redis({
  host: process.env.TAIR_HOST,
  port: 6379,
  password: process.env.TAIR_PASSWORD,
  keyPrefix: 'pk:',
});
EOF

cat > services/api/src/routes/auth.ts << 'EOF'
import { Router } from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import crypto from 'crypto';
import { polarDB, tair } from '../config/alibaba';

const router = Router();

router.post('/register', async (req, res, next) => {
  try {
    const { phone, full_name, pdpa_consent } = req.body;
    if (!phone || !full_name || !pdpa_consent) {
      return res.status(400).json({ error: 'Bagi nama, phone, dan setuju PDPA dulu' });
    }

    const existing = await polarDB.query('SELECT id FROM users WHERE phone = $1', [phone]);
    if (existing.rows.length > 0) {
      return res.status(409).json({ error: 'Nombor ni dah didaftar. Cuba login je.' });
    }

    const otpCode = crypto.randomInt(100000, 999999).toString();
    const otpHash = await bcrypt.hash(otpCode, 10);

    await polarDB.query(
      `INSERT INTO otp_codes (phone, code_hash, purpose, expires_at)
       VALUES ($1, $2, 'register', NOW() + INTERVAL '5 minutes')`,
      [phone, otpHash]
    );

    console.log(`[DEV] OTP for ${phone}: ${otpCode}`);

    res.json({ success: true, message: 'Kita dah hantar code ke phone awak. Check SMS ya!' });
  } catch (error) {
    next(error);
  }
});

router.post('/verify-otp', async (req, res, next) => {
  try {
    const { phone, code, full_name, pdpa_consent } = req.body;

    const otpResult = await polarDB.query(
      `SELECT * FROM otp_codes 
       WHERE phone = $1 AND used_at IS NULL AND expires_at > NOW()
       ORDER BY created_at DESC LIMIT 1`,
      [phone]
    );

    if (otpResult.rows.length === 0) {
      return res.status(400).json({ error: 'Code dah tamat tempoh. Minta code baru ya.' });
    }

    const otp = otpResult.rows[0];
    const isValid = await bcrypt.compare(code, otp.code_hash);

    if (!isValid) {
      await polarDB.query('UPDATE otp_codes SET attempts = attempts + 1 WHERE id = $1', [otp.id]);
      return res.status(400).json({ error: 'Code tak betul. Cuba lagi.' });
    }

    await polarDB.query('UPDATE otp_codes SET used_at = NOW() WHERE id = $1', [otp.id]);

    let userResult = await polarDB.query('SELECT * FROM users WHERE phone = $1', [phone]);
    let user;
    let isNewUser = false;

    if (userResult.rows.length === 0) {
      const referralCode = crypto.randomBytes(4).toString('hex').toUpperCase();
      userResult = await polarDB.query(
        `INSERT INTO users (phone, phone_verified, full_name, pdpa_consent, pdpa_consent_at, referral_code, last_login_at)
         VALUES ($1, TRUE, $2, TRUE, NOW(), $3, NOW()) RETURNING *`,
        [phone, full_name, referralCode]
      );
      user = userResult.rows[0];
      isNewUser = true;
    } else {
      user = userResult.rows[0];
      await polarDB.query('UPDATE users SET last_login_at = NOW() WHERE id = $1', [user.id]);
    }

    const accessToken = jwt.sign(
      { id: user.id, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: '15m' }
    );

    const refreshToken = crypto.randomBytes(64).toString('hex');
    const refreshHash = await bcrypt.hash(refreshToken, 10);

    await polarDB.query(
      `INSERT INTO refresh_tokens (user_id, token_hash, expires_at)
       VALUES ($1, $2, NOW() + INTERVAL '30 days')`,
      [user.id, refreshHash]
    );

    res.json({
      success: true,
      is_new_user: isNewUser,
      user: { id: user.id, phone: user.phone, full_name: user.full_name, role: user.role },
      access_token: accessToken,
      refresh_token: refreshToken,
      expires_in: 900,
    });
  } catch (error) {
    next(error);
  }
});

export default router;
EOF

cat > services/api/src/routes/products.ts << 'EOF'
import { Router } from 'express';
import { polarDB, tair } from '../config/alibaba';

const router = Router();

router.get('/', async (req, res, next) => {
  try {
    const { category, state, is_halal, is_borneo, page = 1, limit = 20 } = req.query;

    const cacheKey = `products:${JSON.stringify(req.query)}`;
    const cached = await tair.get(cacheKey);
    if (cached) return res.json(JSON.parse(cached));

    let query = `
      SELECT p.*, sp.shop_name, sp.halal_verified, sp.is_bumiputera
      FROM products p
      JOIN seller_profiles sp ON p.seller_id = sp.id
      WHERE p.status = 'active'
    `;
    const params: any[] = [];
    let idx = 1;

    if (category) { query += ` AND p.category_id = $${idx++}`; params.push(category); }
    if (state) { query += ` AND p.ships_from_state = $${idx++}`; params.push(state); }
    if (is_halal === 'true') { query += ` AND p.is_halal = TRUE`; }
    if (is_borneo === 'true') { query += ` AND p.is_borneo = TRUE`; }

    query += ` ORDER BY p.created_at DESC LIMIT $${idx++} OFFSET $${idx++}`;
    params.push(Number(limit), (Number(page) - 1) * Number(limit));

    const result = await polarDB.query(query, params);
    const response = { data: result.rows, page: Number(page), limit: Number(limit) };

    await tair.setex(cacheKey, 60, JSON.stringify(response));
    res.json(response);
  } catch (error) {
    next(error);
  }
});

router.get('/:id', async (req, res, next) => {
  try {
    const result = await polarDB.query(
      `SELECT p.*, sp.shop_name, sp.halal_verified FROM products p
       JOIN seller_profiles sp ON p.seller_id = sp.id
       WHERE p.id = $1 AND p.status = 'active'`,
      [req.params.id]
    );
    if (result.rows.length === 0) return res.status(404).json({ error: 'Barang tak jumpa' });
    res.json({ data: result.rows[0] });
  } catch (error) {
    next(error);
  }
});

export default router;
EOF

cat > services/api/src/routes/orders.ts << 'EOF'
import { Router } from 'express';
import { polarDB } from '../config/alibaba';

const router = Router();

router.get('/', async (req, res, next) => {
  try {
    const userId = (req as any).user.id;
    const result = await polarDB.query(
      `SELECT * FROM orders WHERE buyer_id = $1 ORDER BY created_at DESC LIMIT 50`,
      [userId]
    );
    res.json({ data: result.rows });
  } catch (error) {
    next(error);
  }
});

router.get('/:id', async (req, res, next) => {
  try {
    const userId = (req as any).user.id;
    const orderResult = await polarDB.query(
      'SELECT * FROM orders WHERE id = $1 AND buyer_id = $2',
      [req.params.id, userId]
    );
    if (orderResult.rows.length === 0) return res.status(404).json({ error: 'Order tak jumpa' });

    const itemsResult = await polarDB.query(
      'SELECT * FROM order_items WHERE order_id = $1',
      [req.params.id]
    );

    res.json({ data: { ...orderResult.rows[0], items: itemsResult.rows } });
  } catch (error) {
    next(error);
  }
});

export default router;
EOF

cat > services/api/src/routes/live.ts << 'EOF'
import { Router } from 'express';
import { polarDB } from '../config/alibaba';

const router = Router();

router.get('/', async (req, res, next) => {
  try {
    const result = await polarDB.query(
      `SELECT ls.*, sp.shop_name, sp.shop_logo_url
       FROM live_streams ls
       JOIN seller_profiles sp ON ls.seller_id = sp.id
       WHERE ls.status = 'live'
       ORDER BY ls.viewer_count DESC`
    );
    res.json({ data: result.rows });
  } catch (error) {
    next(error);
  }
});

router.get('/:id', async (req, res, next) => {
  try {
    const result = await polarDB.query(
      `SELECT ls.*, sp.shop_name FROM live_streams ls
       JOIN seller_profiles sp ON ls.seller_id = sp.id
       WHERE ls.id = $1`,
      [req.params.id]
    );
    if (result.rows.length === 0) return res.status(404).json({ error: 'Live tak jumpa' });
    res.json({ data: result.rows[0] });
  } catch (error) {
    next(error);
  }
});

export default router;
EOF

cat > services/api/src/middleware/error-handler.ts << 'EOF'
import { Request, Response, NextFunction } from 'express';

export default function errorHandler(err: any, req: Request, res: Response, next: NextFunction) {
  console.error('Error:', err);
  const status = err.status || 500;
  const message = status === 500
    ? 'Alamak, ada problem sikit. Cuba lagi sekali?'
    : err.message || 'Something went wrong';
  res.status(status).json({ error: message });
}
EOF

# ============================================================
# MIGRATIONS
# ============================================================

cat > services/api/migrations/001_initial.sql << 'EOF'
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    phone VARCHAR(15) UNIQUE NOT NULL,
    phone_verified BOOLEAN DEFAULT FALSE,
    email VARCHAR(255) UNIQUE,
    full_name VARCHAR(100),
    avatar_url TEXT,
    role VARCHAR(20) DEFAULT 'buyer' CHECK (role IN ('buyer','seller','admin','moderator')),
    preferred_lang VARCHAR(5) DEFAULT 'ms',
    state VARCHAR(50),
    pdpa_consent BOOLEAN DEFAULT FALSE,
    pdpa_consent_at TIMESTAMPTZ,
    marketing_consent BOOLEAN DEFAULT FALSE,
    last_login_at TIMESTAMPTZ,
    loyalty_points INT DEFAULT 0,
    loyalty_tier VARCHAR(20) DEFAULT 'bronze',
    referral_code VARCHAR(10) UNIQUE,
    is_active BOOLEAN DEFAULT TRUE,
    is_banned BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE otp_codes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    phone VARCHAR(15) NOT NULL,
    code_hash VARCHAR(255) NOT NULL,
    purpose VARCHAR(30) NOT NULL,
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 5,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE seller_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    shop_name VARCHAR(100) NOT NULL,
    shop_slug VARCHAR(100) UNIQUE NOT NULL,
    shop_description TEXT,
    shop_logo_url TEXT,
    halal_cert_url TEXT,
    halal_verified BOOLEAN DEFAULT FALSE,
    state VARCHAR(50) NOT NULL,
    is_east_malaysia BOOLEAN GENERATED ALWAYS AS (state IN ('Sabah','Sarawak','W.P. Labuan')) STORED,
    is_bumiputera BOOLEAN DEFAULT FALSE,
    tier VARCHAR(30) DEFAULT 'peniaga_kecil',
    rating DECIMAL(3,2) DEFAULT 0.00,
    total_sales INT DEFAULT 0,
    commission_rate DECIMAL(5,2) DEFAULT 5.00,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    slug VARCHAR(50) UNIQUE NOT NULL,
    name_ms VARCHAR(100) NOT NULL,
    parent_id UUID REFERENCES categories(id),
    icon VARCHAR(10),
    sort_order INT DEFAULT 0,
    is_borneo BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    seller_id UUID NOT NULL REFERENCES seller_profiles(id),
    category_id UUID REFERENCES categories(id),
    title VARCHAR(200) NOT NULL,
    description TEXT NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    original_price DECIMAL(10,2),
    stock INT NOT NULL DEFAULT 0,
    weight_grams INT,
    ships_from_state VARCHAR(50) NOT NULL,
    is_halal BOOLEAN DEFAULT FALSE,
    is_borneo BOOLEAN GENERATED ALWAYS AS (ships_from_state IN ('Sabah','Sarawak','W.P. Labuan')) STORED,
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft','active','sold_out','removed')),
    view_count INT DEFAULT 0,
    sold_count INT DEFAULT 0,
    rating DECIMAL(3,2) DEFAULT 0.00,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_status ON products(status) WHERE status = 'active';
CREATE INDEX idx_products_borneo ON products(is_borneo) WHERE is_borneo = TRUE;
CREATE INDEX idx_products_halal ON products(is_halal) WHERE is_halal = TRUE;

CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    image_url TEXT NOT NULL,
    sort_order INT DEFAULT 0,
    is_primary BOOLEAN DEFAULT FALSE
);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_number VARCHAR(30) UNIQUE NOT NULL,
    buyer_id UUID NOT NULL REFERENCES users(id),
    seller_id UUID NOT NULL REFERENCES seller_profiles(id),
    subtotal DECIMAL(10,2) NOT NULL,
    shipping_fee DECIMAL(10,2) DEFAULT 0,
    discount DECIMAL(10,2) DEFAULT 0,
    total DECIMAL(10,2) NOT NULL,
    shipping_address JSONB NOT NULL,
    status VARCHAR(30) DEFAULT 'pending_payment',
    payment_method VARCHAR(30),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE live_streams (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    seller_id UUID NOT NULL REFERENCES seller_profiles(id),
    title VARCHAR(200),
    stream_name VARCHAR(100) UNIQUE NOT NULL,
    playback_hls_url TEXT,
    status VARCHAR(20) DEFAULT 'scheduled',
    viewer_count INT DEFAULT 0,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id),
    method VARCHAR(30) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    gateway_ref VARCHAR(100),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id UUID,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
EOF

# ============================================================
# FRONTEND — Flutter
# ============================================================

mkdir -p frontend/mobile/lib/{core/theme,l10n,screens,screens/home,widgets,providers}

cat > frontend/mobile/pubspec.yaml << 'EOF'
name: pasarkita
description: "PasarKita - Syok Sendiri, Syok Semua!"
version: 1.0.0+1
environment:
  sdk: '>=3.2.0 <4.0.0'
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.1
  dio: ^5.4.0
  shared_preferences: ^2.2.2
  cached_network_image: ^3.3.1
  intl: ^0.19.0
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1
flutter:
  uses-material-design: true
EOF

cat > frontend/mobile/lib/main.dart << 'EOF'
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'app.dart';
import 'providers/cart_provider.dart';

void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => CartProvider()),
      ],
      child: const PasarKitaApp(),
    ),
  );
}
EOF

cat > frontend/mobile/lib/app.dart << 'EOF'
import 'package:flutter/material.dart';
import 'core/theme/app_theme.dart';
import 'screens/home/home_screen.dart';

class PasarKitaApp extends StatelessWidget {
  const PasarKitaApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'PasarKita',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      home: const HomeScreen(),
    );
  }
}
EOF

cat > frontend/mobile/lib/core/theme/app_colors.dart << 'EOF'
import 'package:flutter/material.dart';

class AppColors {
  static const Color primary = Color(0xFFFF6B35);
  static const Color primaryLight = Color(0xFFFFF3E0);
  static const Color secondary = Color(0xFF1B5E20);
  static const Color borneoGreen = Color(0xFF228B22);
  static const Color sabahBlue = Color(0xFF1565C0);
  static const Color sarawakRed = Color(0xFFDC143C);
  static const Color background = Color(0xFFF5F0E8);
  static const Color surface = Color(0xFFFFFFFF);
  static const Color textPrimary = Color(0xFF1A1A1A);
  static const Color textSecondary = Color(0xFF666666);
  static const Color textHint = Color(0xFF999999);
  static const Color border = Color(0xFFE0E0E0);
  static const Color success = Color(0xFF4CAF50);
  static const Color error = Color(0xFFE53935);
  static const Color halalGreen = Color(0xFF00695C);
  static const Color halalBg = Color(0xFFE0F2F1);
}
EOF

cat > frontend/mobile/lib/core/theme/app_theme.dart << 'EOF'
import 'package:flutter/material.dart';
import 'app_colors.dart';

class AppTheme {
  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      scaffoldBackgroundColor: AppColors.background,
      colorScheme: const ColorScheme.light(
        primary: AppColors.primary,
        secondary: AppColors.secondary,
        surface: AppColors.surface,
      ),
      appBarTheme: const AppBarTheme(
        backgroundColor: AppColors.background,
        elevation: 0,
        titleTextStyle: TextStyle(
          fontSize: 20,
          fontWeight: FontWeight.bold,
          color: AppColors.textPrimary,
        ),
      ),
    );
  }
}
EOF

cat > frontend/mobile/lib/l10n/manglish.dart << 'EOF'
class ManglishStrings {
  static const String appName = 'PasarKita';
  static const String tagline = 'Syok Sendiri, Syok Semua!';
  static const String navHome = 'Jom Tengok';
  static const String navLive = 'Live Lah!';
  static const String navCart = 'Beg Saya';
  static const String navOrders = 'Order Saya';
  static const String navProfile = 'Profil Saya';
  static const String searchPlaceholder = 'Cari apa hari ni? Cuba "sambal mak"';
  static const String addToCart = 'Campak Dalam Beg';
  static const String buyNow = 'Beli Terus, Tak Payah Fikir';
  static const String soldOut = 'Alamak, Habis Sudah!';
  static const String cartEmpty = 'Beg awak kosong lah. Jom pergi shopping!';
  static const String orderPlaced = 'Syok! Order dah masuk!';
  static const String eastMsiaTitle = 'Dari Borneo, Untuk Semua';
  static const String sabahTag = 'Sabah Bah!';
  static const String sarawakTag = 'Sarawak Tok!';
}
EOF

cat > frontend/mobile/lib/providers/cart_provider.dart << 'EOF'
import 'package:flutter/material.dart';

class CartProvider extends ChangeNotifier {
  final List<Map<String, dynamic>> _items = [];
  List<Map<String, dynamic>> get items => _items;
  int get itemCount => _items.length;
  double get total => _items.fold(0, (sum, item) =>
    sum + (item['price'] as double) * (item['quantity'] as int));

  void addItem(Map<String, dynamic> product) {
    final idx = _items.indexWhere((i) => i['id'] == product['id']);
    if (idx >= 0) {
      _items[idx]['quantity'] = (_items[idx]['quantity'] as int) + 1;
    } else {
      _items.add({...product, 'quantity': 1});
    }
    notifyListeners();
  }

  void removeItem(String id) {
    _items.removeWhere((i) => i['id'] == id);
    notifyListeners();
  }
}
EOF

cat > frontend/mobile/lib/screens/home/home_screen.dart << 'EOF'
import 'package:flutter/material.dart';
import '../../core/theme/app_colors.dart';
import '../../l10n/manglish.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: AppColors.background,
      appBar: AppBar(
        title: const Text(ManglishStrings.appName),
        actions: [
          IconButton(icon: const Icon(Icons.search), onPressed: () {}),
          IconButton(icon: const Icon(Icons.notifications_outlined), onPressed: () {}),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Container(
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color: AppColors.primaryLight,
                borderRadius: BorderRadius.circular(16),
              ),
              child: const Row(
                children: [
                  Text('🌙', style: TextStyle(fontSize: 32)),
                  SizedBox(width: 12),
                  Expanded(
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text('Selamat Hari Raya!',
                          style: TextStyle(font
