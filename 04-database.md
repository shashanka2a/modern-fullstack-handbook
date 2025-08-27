# Chapter 4: Database - PostgreSQL and Prisma

## 4.1 Concepts

PostgreSQL is a powerful open-source relational database. It uses tables, rows, and SQL queries to manage structured data. While SQL is expressive, writing raw queries in application code can be verbose and error-prone.

Prisma solves this by providing:

- **A schema definition file** (`schema.prisma`) to model tables
- **A type-safe client** with autocomplete for Node.js/TypeScript
- **Migrations** to version and update the database schema
- **Prisma Studio**, a UI to browse and edit your data

## 4.2 Example Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  transactions Transaction[]
}

model Transaction {
  id               String   @id @default(cuid())
  userId           String
  beneficiaryId    String
  sourceAmount     Decimal  @db.Decimal(18,2)
  sourceCurrency   String
  targetAmount     Decimal  @db.Decimal(18,2)
  targetCurrency   String
  fxRate           Decimal  @db.Decimal(18,6)
  feeFixed         Decimal  @db.Decimal(18,2)
  feePct           Decimal  @db.Decimal(5,4)
  status           String
  highRisk         Boolean  @default(false)
  createdAt        DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id])
}
```

## 4.3 Workflow

1. **Define models** in `prisma/schema.prisma`
2. **Run migration**: `npx prisma migrate dev --name init` to create tables
3. **Use the generated client** in your code:

```javascript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Get user transactions
const txns = await prisma.transaction.findMany({
  where: { userId },
  orderBy: { createdAt: 'desc' },
  take: 50,
  include: {
    user: {
      select: { email: true }
    }
  }
});
```

## 4.4 Hands-On Exercise

### Step 1: Install Prisma

```bash
npm install @prisma/client
npm install -D prisma
```

### Step 2: Initialize Prisma

```bash
npx prisma init
```

This creates:
- `prisma/schema.prisma` - Your database schema
- `.env` - Environment variables (including DATABASE_URL)

### Step 3: Configure Database Connection

Update your `.env` file:

```env
# Database
DATABASE_URL="postgresql://username:password@localhost:5432/mydb?schema=public"

# JWT Secrets
ACCESS_TOKEN_SECRET="your-access-token-secret"
REFRESH_TOKEN_SECRET="your-refresh-token-secret"
```

### Step 4: Define Schema

Update `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  transactions Transaction[]
}

model Transaction {
  id               String   @id @default(cuid())
  userId           String
  beneficiaryId    String
  sourceAmount     Decimal  @db.Decimal(18,2)
  sourceCurrency   String
  targetAmount     Decimal  @db.Decimal(18,2)
  targetCurrency   String
  fxRate           Decimal  @db.Decimal(18,6)
  feeFixed         Decimal  @db.Decimal(18,2)
  feePct           Decimal  @db.Decimal(5,4)
  status           String   @default("pending")
  highRisk         Boolean  @default(false)
  createdAt        DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id])
}
```

### Step 5: Run Migration

```bash
npx prisma migrate dev --name init
```

This will:
- Create the database tables
- Generate the Prisma client

### Step 6: Generate Client

```bash
npx prisma generate
```

### Step 7: Test Database Operations

Create a test script `test-db.js`:

```javascript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  try {
    // Create a user
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        password: 'hashed_password_here'
      }
    });
    
    console.log('Created user:', user);
    
    // Create a transaction
    const transaction = await prisma.transaction.create({
      data: {
        userId: user.id,
        beneficiaryId: 'beneficiary_123',
        sourceAmount: 100.00,
        sourceCurrency: 'USD',
        targetAmount: 85.50,
        targetCurrency: 'EUR',
        fxRate: 0.855,
        feeFixed: 2.50,
        feePct: 0.015,
        status: 'completed'
      }
    });
    
    console.log('Created transaction:', transaction);
    
    // Query transactions with user info
    const transactions = await prisma.transaction.findMany({
      where: { userId: user.id },
      include: {
        user: {
          select: { email: true }
        }
      }
    });
    
    console.log('User transactions:', transactions);
    
  } catch (error) {
    console.error('Error:', error);
  } finally {
    await prisma.$disconnect();
  }
}

main();
```

Run the test:
```bash
node test-db.js
```

## 4.5 Mini Project

Build a database-backed service that integrates with the Express API from Chapter 3.

### Updated Express Server with Prisma

```javascript
import express from 'express';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import cookieParser from 'cookie-parser';
import cors from 'cors';
import { PrismaClient } from '@prisma/client';

const app = express();
const prisma = new PrismaClient();

// Middleware
app.use(express.json());
app.use(cookieParser());
app.use(cors({ credentials: true, origin: 'http://localhost:3000' }));

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET || 'dev-secret';
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET || 'dev-refresh-secret';

// Routes
app.get('/health', (req, res) => {
  res.json({ ok: true, timestamp: new Date().toISOString() });
});

// Create user endpoint
app.post('/users', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = await prisma.user.create({
      data: {
        email,
        password: hashedPassword
      }
    });
    
    // Don't return password
    const { password: _, ...userWithoutPassword } = user;
    res.status(201).json(userWithoutPassword);
    
  } catch (error) {
    if (error.code === 'P2002') {
      return res.status(400).json({ error: 'Email already exists' });
    }
    console.error('Create user error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Login endpoint
app.post('/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }
    
    // Find user
    const user = await prisma.user.findUnique({
      where: { email }
    });
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Verify password
    const validPassword = await bcrypt.compare(password, user.password);
    if (!validPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate tokens
    const accessToken = jwt.sign(
      { userId: user.id, email: user.email },
      ACCESS_TOKEN_SECRET,
      { expiresIn: '15m' }
    );
    
    const refreshToken = jwt.sign(
      { userId: user.id },
      REFRESH_TOKEN_SECRET,
      { expiresIn: '7d' }
    );
    
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: false,
      sameSite: 'lax',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });
    
    res.json({ 
      accessToken, 
      user: { id: user.id, email: user.email } 
    });
    
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Authentication middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }
  
  jwt.verify(token, ACCESS_TOKEN_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

// Get user transactions
app.get('/transactions', authenticateToken, async (req, res) => {
  try {
    const transactions = await prisma.transaction.findMany({
      where: { userId: req.user.userId },
      orderBy: { createdAt: 'desc' },
      take: 50
    });
    
    res.json(transactions);
    
  } catch (error) {
    console.error('Get transactions error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create transaction
app.post('/transactions', authenticateToken, async (req, res) => {
  try {
    const {
      beneficiaryId,
      sourceAmount,
      sourceCurrency,
      targetAmount,
      targetCurrency,
      fxRate
    } = req.body;
    
    const transaction = await prisma.transaction.create({
      data: {
        userId: req.user.userId,
        beneficiaryId,
        sourceAmount: parseFloat(sourceAmount),
        sourceCurrency,
        targetAmount: parseFloat(targetAmount),
        targetCurrency,
        fxRate: parseFloat(fxRate),
        feeFixed: 2.50, // Fixed fee
        feePct: 0.015,  // 1.5% fee
        status: 'pending'
      }
    });
    
    res.status(201).json(transaction);
    
  } catch (error) {
    console.error('Create transaction error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Graceful shutdown
process.on('SIGINT', async () => {
  await prisma.$disconnect();
  process.exit(0);
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`API server running on port ${PORT}`);
});
```

### Testing the Complete API

1. **Create a user**:
   ```bash
   curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"email": "test@example.com", "password": "password123"}'
   ```

2. **Login**:
   ```bash
   curl -X POST http://localhost:8080/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "test@example.com", "password": "password123"}'
   ```

3. **Create a transaction** (use token from login):
   ```bash
   curl -X POST http://localhost:8080/transactions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -d '{
       "beneficiaryId": "ben_123",
       "sourceAmount": 100,
       "sourceCurrency": "USD",
       "targetAmount": 85.50,
       "targetCurrency": "EUR",
       "fxRate": 0.855
     }'
   ```

4. **Get transactions**:
   ```bash
   curl -X GET http://localhost:8080/transactions \
     -H "Authorization: Bearer YOUR_TOKEN"
   ```

This demonstrates how Prisma integrates seamlessly with Express and PostgreSQL, providing type-safe database operations with minimal boilerplate.

## Key Benefits Demonstrated

- **Type Safety**: Prisma generates TypeScript types for your models
- **Migration Management**: Version control for your database schema
- **Query Builder**: Intuitive API for complex queries
- **Relationship Handling**: Easy joins and nested queries
- **Error Handling**: Descriptive error codes for common issues

---

**Previous**: [Backend: Node.js and Express](03-backend.md) | **Next**: [Caching: Redis](05-caching.md)