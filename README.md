
## 📂 PROJECT STRUCTURE

Create this exact folder structure:


```
ekpoma-artisan-connect/
├── server.js
├── package.json
├── .env
├── .env.example
├── .gitignore
├── README.md
├── Procfile                    (for Heroku)
├── vercel.json                 (for Vercel)
├── database-schema.sql
├── config/
│   ├── database.js
│   └── paystack.js
├── middleware/
│   ├── auth.js
│   └── errorHandler.js
├── routes/
│   ├── auth.js
│   ├── artisans.js
│   ├── bookings.js
│   └── payments.js
├── public/
│   ├── index.html
│   ├── manifest.json
│   ├── service-worker.js
│   ├── icons/
│   │   ├── icon-192x192.png
│   │   └── icon-512x512.png
│   └── css/
│       └── styles.css
└── scripts/
    └── seed.js
```

---

## 📄 FILE #1: .gitignore

```gitignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Environment variables
.env
.env.local
.env.production

# Build files
build/
dist/
*.tgz

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Logs
logs/
*.log

# Testing
coverage/

# Database
*.db
*.sqlite

# Uploads
uploads/
temp/
```

---

## 📄 FILE #2: README.md

```markdown
# Ekpoma Artisan Connect

Digital marketplace connecting verified artisans with clients in Ekpoma, Edo State, Nigeria.

## Features

- 🛠️ Browse 14 service categories
- 👨‍🔧 Verified artisan profiles
- 💳 Secure Paystack payments
- 🎁 Refer & earn ₦200 per referral
- 🚨 Emergency medical services
- 📱 Progressive Web App (iOS & Android)
- 👨‍🍳 Hire professional cooks (Local & International)
- 👔 Laundry services

## Quick Start

### Prerequisites

- Node.js v18+
- PostgreSQL 12+
- Paystack account

### Installation

```bash
# Clone repository
git clone https://github.com/yourusername/ekpoma-artisan-connect.git
cd ekpoma-artisan-connect

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your values

# Set up database
psql -U postgres -d ekpoma_artisan -f database-schema.sql

# Start development server
npm run dev
```

### Deployment

**Heroku:**
```bash
heroku create ekpoma-artisan
heroku addons:create heroku-postgresql:mini
git push heroku main
```

**Render/Railway:** Connect GitHub and deploy

## Tech Stack

- Backend: Node.js, Express.js
- Database: PostgreSQL
- Payment: Paystack
- Authentication: JWT
- Email: Nodemailer

## API Documentation

See `docs/api.md` for complete API documentation.

## Contact

- Email: artisan.connectekp@gmail.com
- Phone: 080-ARTISAN-1
- Facebook: https://www.facebook.com/profile.php?id=61583315283666
- Instagram: @artisanconnect__

## License

MIT License - See LICENSE file

## Contributing

Pull requests welcome! See CONTRIBUTING.md

---

© 2025 Ekpoma Artisan Connect. All rights reserved.
```

---

## 📄 FILE #3: Procfile (for Heroku)

```
web: node server.js
```

---

## 📄 FILE #4: vercel.json (for Vercel)

```json
{
  "version": 2,
  "builds": [
    {
      "src": "server.js",
      "use": "@vercel/node"
    },
    {
      "src": "public/**",
      "use": "@vercel/static"
    }
  ],
  "routes": [
    {
      "src": "/api/.*",
      "dest": "server.js"
    },
    {
      "src": "/(.*)",
      "dest": "public/$1"
    }
  ]
}
```

---

## 📄 FILE #5: config/database.js

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' 
    ? { rejectUnauthorized: false } 
    : false,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Test connection
pool.on('connect', () => {
  console.log('✅ Database connected');
});

pool.on('error', (err) => {
  console.error('❌ Database connection error:', err);
});

// Query helper
const query = async (text, params) => {
  const start = Date.now();
  try {
    const res = await pool.query(text, params);
    const duration = Date.now() - start;
    console.log('Executed query', { text, duration, rows: res.rowCount });
    return res;
  } catch (error) {
    console.error('Query error:', error);
    throw error;
  }
};

module.exports = {
  pool,
  query
};
```

---

## 📄 FILE #6: config/paystack.js

```javascript
const axios = require('axios');

const paystackClient = axios.create({
  baseURL: 'https://api.paystack.co',
  headers: {
    Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
    'Content-Type': 'application/json'
  }
});

const initializePayment = async (email, amount, reference, metadata) => {
  try {
    const response = await paystackClient.post('/transaction/initialize', {
      email,
      amount: amount * 100, // Convert to kobo
      currency: 'NGN',
      reference,
      callback_url: `${process.env.FRONTEND_URL}/payment/callback`,
      metadata
    });
    return response.data;
  } catch (error) {
    console.error('Paystack initialization error:', error.response?.data || error.message);
    throw error;
  }
};

const verifyPayment = async (reference) => {
  try {
    const response = await paystackClient.get(`/transaction/verify/${reference}`);
    return response.data;
  } catch (error) {
    console.error('Paystack verification error:', error.response?.data || error.message);
    throw error;
  }
};

const createTransferRecipient = async (accountName, accountNumber, bankCode) => {
  try {
    const response = await paystackClient.post('/transferrecipient', {
      type: 'nuban',
      name: accountName,
      account_number: accountNumber,
      bank_code: bankCode,
      currency: 'NGN'
    });
    return response.data;
  } catch (error) {
    console.error('Paystack recipient error:', error.response?.data || error.message);
    throw error;
  }
};

const initiateTransfer = async (recipientCode, amount, reason) => {
  try {
    const response = await paystackClient.post('/transfer', {
      source: 'balance',
      amount: amount * 100, // Convert to kobo
      recipient: recipientCode,
      reason
    });
    return response.data;
  } catch (error) {
    console.error('Paystack transfer error:', error.response?.data || error.message);
    throw error;
  }
};

module.exports = {
  initializePayment,
  verifyPayment,
  createTransferRecipient,
  initiateTransfer
};
```

---

## 📄 FILE #7: middleware/auth.js

```javascript
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  try {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
      return res.status(401).json({ 
        success: false,
        error: 'Access token required' 
      });
    }

    jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
      if (err) {
        return res.status(403).json({ 
          success: false,
          error: 'Invalid or expired token' 
        });
      }
      
      req.user = user;
      next();
    });
  } catch (error) {
    console.error('Authentication error:', error);
    res.status(500).json({ 
      success: false,
      error: 'Authentication failed' 
    });
  }
};

const isArtisan = (req, res, next) => {
  if (req.user.user_type !== 'artisan') {
    return res.status(403).json({ 
      success: false,
      error: 'Artisan access only' 
    });
  }
  next();
};

const isAdmin = (req, res, next) => {
  if (req.user.user_type !== 'admin') {
    return res.status(403).json({ 
      success: false,
      error: 'Admin access only' 
    });
  }
  next();
};

module.exports = {
  authenticateToken,
  isArtisan,
  isAdmin
};
```

---

## 📄 FILE #8: middleware/errorHandler.js

```javascript
const errorHandler = (err, req, res, next) => {
  console.error('Error:', err);

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      success: false,
      error: 'Validation failed',
      details: Object.values(err.errors).map(e => e.message)
    });
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      success: false,
      error: 'Invalid token'
    });
  }

  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      success: false,
      error: 'Token expired'
    });
  }

  // PostgreSQL errors
  if (err.code === '23505') { // Unique violation
    return res.status(409).json({
      success: false,
      error: 'Duplicate entry'
    });
  }

  if (err.code === '23503') { // Foreign key violation
    return res.status(400).json({
      success: false,
      error: 'Invalid reference'
    });
  }

  // Default error
  res.status(err.statusCode || 500).json({
    success: false,
    error: err.message || 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
};

const notFound = (req, res, next) => {
  res.status(404).json({
    success: false,
    error: `Route not found: ${req.originalUrl}`
  });
};

module.exports = {
  errorHandler,
  notFound
};
```

---

## 📄 FILE #9: scripts/seed.js

```javascript
const { pool } = require('../config/database');
const bcrypt = require('bcryptjs');

const seedData = async () => {
  try {
    console.log('🌱 Starting database seed...');

    // Create admin user
    const adminPassword = await bcrypt.hash('admin123', 10);
    await pool.query(
      `INSERT INTO users (email, phone, password_hash, full_name, user_type, is_verified)
       VALUES ($1, $2, $3, $4, $5, $6)
       ON CONFLICT (email) DO NOTHING`,
      ['admin@ekpomaartisan.com', '08000000000', adminPassword, 'Admin User', 'admin', true]
    );

    console.log('✅ Admin user created');

    // Create sample categories
    const categories = [
      ['Electrician', 'Electrical wiring, repairs, and installations', 'zap', 1],
      ['Generator Mechanic', 'Generator servicing and repairs', 'settings', 2],
      ['Plumber', 'Plumbing installations and repairs', 'droplets', 3],
      ['Hire A Cook', 'Professional cooking services', 'chef-hat', 4]
    ];

    for (const cat of categories) {
      await pool.query(
        `INSERT INTO service_categories (name, description, icon_name, display_order)
         VALUES ($1, $2, $3, $4)
         ON CONFLICT (name) DO NOTHING`,
        cat
      );
    }

    console.log('✅ Categories created');

    // Create sample emergency services
    const emergencyServices = [
      ['Ambrose Alli University Teaching Hospital', 'hospital', 'Ekpoma', 'AAU Campus', '08033445566', '112'],
      ['Edo State Ambulance Service', 'ambulance', 'Edo State', 'State-wide', '112', '112']
    ];

    for (const service of emergencyServices) {
      await pool.query(
        `INSERT INTO emergency_services (name, service_type, location, address, phone, emergency_line, is_active)
         VALUES ($1, $2, $3, $4, $5, $6, true)
         ON CONFLICT DO NOTHING`,
        service
      );
    }

    console.log('✅ Emergency services created');

    console.log('🎉 Database seeded successfully!');
    process.exit(0);
  } catch (error) {
    console.error('❌ Seed error:', error);
    process.exit(1);
  }
};

seedData();
```

---

## 📄 FILE #10: .env (PRODUCTION TEMPLATE)

```env
# IMPORTANT: Copy this file and rename to .env
# Then fill in your actual values

# Server Configuration
NODE_ENV=production
PORT=5000

# Database (Get from your hosting provider)
DATABASE_URL=postgresql://username:password@host:5432/database_name

# JWT Secret (Generate a strong random string)
# Use: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
JWT_SECRET=your_generated_secret_key_here_min_32_characters_long

# Frontend URL (Your actual domain)
FRONTEND_URL=https://ekpomaartisan.com

# Paystack (Get from https://dashboard.paystack.com/#/settings/developer)
PAYSTACK_PUBLIC_KEY=pk_live_your_actual_live_public_key
PAYSTACK_SECRET_KEY=sk_live_your_actual_live_secret_key

# Email Configuration (Gmail App Password)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=artisan.connectekp@gmail.com
EMAIL_PASSWORD=your_gmail_app_password_16_characters
EMAIL_FROM=Ekpoma Artisan Connect <artisan.connectekp@gmail.com>

# Social Media
FACEBOOK_URL=https://www.facebook.com/profile.php?id=61583315283666
INSTAGRAM_URL=https://www.instagram.com/artisanconnect__
SUPPORT_EMAIL=artisan.connectekp@gmail.com
SUPPORT_PHONE=080-ARTISAN-1

# Security
BCRYPT_ROUNDS=10
TOKEN_EXPIRY=30d
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# App Info
APP_NAME=Ekpoma Artisan Connect
APP_URL=https://ekpomaartisan.com
```

---

## 📄 FILE #11: Installation Script (install.sh)

```bash
#!/bin/bash

echo "🚀 Installing Ekpoma Artisan Connect..."

# Check Node.js
if ! command -v node &> /dev/null; then
    echo "❌ Node.js not found. Please install Node.js 18+ first."
    exit 1
fi

echo "✅ Node.js found: $(node --version)"

# Install dependencies
echo "📦 Installing dependencies..."
npm install

# Create .env file
if [ ! -f .env ]; then
    echo "📝 Creating .env file..."
    cp .env.example .env
    echo "⚠️  Please edit .env with your actual values"
else
    echo "✅ .env file already exists"
fi

# Create necessary directories
mkdir -p public/icons
mkdir -p logs
mkdir -p uploads

echo "✅ Installation complete!"
echo ""
echo "Next steps:"
echo "1. Edit .env with your database and API keys"
echo "2. Run database migrations: psql -U postgres -d ekpoma_artisan -f database-schema.sql"
echo "3. Seed database: npm run seed"
echo "4. Start server: npm run dev"
echo ""
echo "📧 Support: artisan.connectekp@gmail.com"
```

---

## 📄 FILE #12: Deployment Script (deploy.sh)

```bash
#!/bin/bash

echo "🚀 Deploying Ekpoma Artisan Connect..."

# Check if on main branch
BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$BRANCH" != "main" ]; then
    echo "⚠️  Not on main branch. Switch to main: git checkout main"
    exit 1
fi

# Run tests
echo "🧪 Running tests..."
npm test

if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Fix errors before deploying."
    exit 1
fi

# Build frontend (if applicable)
echo "🏗️  Building frontend..."
npm run build

# Commit and push
echo "📤 Pushing to repository..."
git add .
git commit -m "Deploy: $(date +%Y-%m-%d\ %H:%M:%S)"
git push origin main

# Deploy to Heroku
echo "🚀 Deploying to Heroku..."
git push heroku main

# Run migrations on Heroku
echo "📊 Running database migrations..."
heroku run psql $DATABASE_URL < database-schema.sql

echo "✅ Deployment complete!"
echo "🌐 App URL: $(heroku info -s | grep web_url | cut -d= -f2)"
```

---

## 🎯 DEPLOYMENT INSTRUCTIONS

### **STEP 1: Copy All Files**

1. Create a new folder: `ekpoma-artisan-connect`
2. Copy all files I provided into the folder
3. Make sure folder structure matches the one shown above

### **STEP 2: Install Dependencies**

```bash
cd ekpoma-artisan-connect
npm install
```

This will install all packages from package.json automatically.

### **STEP 3: Configure Environment**

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your actual values
nano .env  # or use any text editor
```

**MUST CHANGE:**
- DATABASE_URL (your PostgreSQL connection)
- JWT_SECRET (generate with: `node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"`)
- PAYSTACK keys (get from dashboard.paystack.com)
- EMAIL_PASSWORD (Gmail app password)

### **STEP 4: Set Up Database**

```bash
# Create database
createdb ekpoma_artisan

# Run migrations
psql -U postgres -d ekpoma_artisan -f database-schema.sql

# Seed initial data
npm run seed
```

### **STEP 5: Test Locally**

```bash
npm run dev
```

Visit: http://localhost:5000

### **STEP 6: Deploy to Heroku**

```bash
# Login to Heroku
heroku login

# Create app
heroku create ekpoma-artisan

# Add PostgreSQL
heroku addons:create heroku-postgresql:mini

# Set environment variables
heroku config:set NODE_ENV=production
heroku config:set JWT_SECRET=your_generated_secret
heroku config:set PAYSTACK_SECRET_KEY=sk_live_xxx
# ... set all other variables

# Deploy
git init
git add .
git commit -m "Initial commit"
git push heroku main

# Run migrations
heroku pg:psql < database-schema.sql

# Open app
heroku open
```

---
