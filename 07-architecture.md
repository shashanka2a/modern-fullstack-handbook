# Chapter 7: Putting It Together

Now that we've covered each component of the modern full-stack, let's see how they all work together in a complete system architecture.

## 7.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  Next.js Frontend (React + SSR + API Routes)                   │
│  ├── Pages (File-based routing)                                │
│  ├── Components (Reusable UI)                                  │
│  ├── API Routes (/api/*)                                       │
│  └── Static Assets                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │ HTTP/HTTPS
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API LAYER                                  │
├─────────────────────────────────────────────────────────────────┤
│  Express.js Backend                                             │
│  ├── Routes (/auth, /users, /transactions)                     │
│  ├── Middleware (Auth, CORS, Rate Limiting)                    │
│  ├── Controllers (Request/Response handling)                   │
│  └── Services (Business Logic)                                 │
└─────────────┬───────────────────┬───────────────────────────────┘
              │                   │
              ▼                   ▼
┌─────────────────────┐  ┌─────────────────────┐
│   CACHE LAYER       │  │   DATABASE LAYER    │
├─────────────────────┤  ├─────────────────────┤
│  Redis              │  │  PostgreSQL         │
│  ├── Session Store  │  │  ├── Users          │
│  ├── API Cache     │  │  ├── Transactions   │
│  ├── Rate Limits   │  │  └── Audit Logs     │
│  └── Pub/Sub       │  │                     │
└─────────────────────┘  │  Prisma ORM         │
                         │  ├── Type Safety    │
                         │  ├── Migrations     │
                         │  └── Query Builder  │
                         └─────────────────────┘
```

## 7.2 Complete System Integration

Let's build a complete application that demonstrates all the concepts working together.

### Project Structure

```
fullstack-app/
├── frontend/                 # Next.js application
│   ├── pages/
│   │   ├── index.js         # Landing page
│   │   ├── login.js         # Authentication
│   │   ├── dashboard.js     # Main app
│   │   └── api/             # API routes (optional)
│   ├── components/
│   └── package.json
├── backend/                  # Express API
│   ├── src/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── services/
│   │   └── middleware/
│   ├── prisma/
│   │   └── schema.prisma
│   └── package.json
├── docker-compose.yml        # Development environment
├── docker-compose.prod.yml   # Production environment
└── README.md
```

### Backend Integration (`backend/src/app.js`)

```javascript
import express from 'express';
import cors from 'cors';
import cookieParser from 'cookie-parser';
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';

// Route imports
import authRoutes from './routes/auth.js';
import userRoutes from './routes/users.js';
import transactionRoutes from './routes/transactions.js';
import weatherRoutes from './routes/weather.js';

// Middleware imports
import { authenticateToken } from './middleware/auth.js';
import { rateLimiter } from './middleware/rateLimit.js';
import { errorHandler } from './middleware/errorHandler.js';

const app = express();

// Initialize services
export const prisma = new PrismaClient();
export const redis = new Redis(process.env.REDIS_URL);

// Global middleware
app.use(express.json({ limit: '10mb' }));
app.use(cookieParser());
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));

// Rate limiting
app.use('/api/', rateLimiter);

// Health check
app.get('/health', async (req, res) => {
  try {
    // Check database
    await prisma.$queryRaw`SELECT 1`;
    
    // Check Redis
    await redis.ping();
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        database: 'connected',
        redis: 'connected',
        uptime: process.uptime()
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

// API routes
app.use('/api/auth', authRoutes);
app.use('/api/users', authenticateToken, userRoutes);
app.use('/api/transactions', authenticateToken, transactionRoutes);
app.use('/api/weather', weatherRoutes);

// Error handling
app.use(errorHandler);

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Route not found',
    path: req.originalUrl
  });
});

// Graceful shutdown
const gracefulShutdown = async () => {
  console.log('Shutting down gracefully...');
  
  try {
    await prisma.$disconnect();
    await redis.quit();
    console.log('Services disconnected successfully');
    process.exit(0);
  } catch (error) {
    console.error('Error during shutdown:', error);
    process.exit(1);
  }
};

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

export default app;
```

### Complete Prisma Schema (`backend/prisma/schema.prisma`)

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
  firstName String?
  lastName  String?
  avatar    String?
  verified  Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // Relations
  transactions Transaction[]
  sessions     Session[]
  auditLogs    AuditLog[]
  
  @@map("users")
}

model Session {
  id           String   @id @default(cuid())
  userId       String
  refreshToken String   @unique
  expiresAt    DateTime
  createdAt    DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("sessions")
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
  status           TransactionStatus @default(PENDING)
  highRisk         Boolean  @default(false)
  reference        String   @unique @default(cuid())
  notes            String?
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
  
  user User @relation(fields: [userId], references: [id])
  
  @@map("transactions")
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String?
  action    String
  resource  String
  details   Json?
  ipAddress String?
  userAgent String?
  createdAt DateTime @default(now())
  
  user User? @relation(fields: [userId], references: [id])
  
  @@map("audit_logs")
}

model WeatherRequest {
  id        String   @id @default(cuid())
  city      String
  cached    Boolean  @default(false)
  response  Json?
  timestamp DateTime @default(now())
  
  @@map("weather_requests")
}

enum TransactionStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
  CANCELLED
}
```

### Frontend Integration (`frontend/pages/dashboard.js`)

```jsx
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../components/Layout';
import TransactionList from '../components/TransactionList';
import WeatherWidget from '../components/WeatherWidget';

export default function Dashboard() {
  const [user, setUser] = useState(null);
  const [transactions, setTransactions] = useState([]);
  const [weather, setWeather] = useState(null);
  const [loading, setLoading] = useState(true);
  const router = useRouter();

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const token = localStorage.getItem('accessToken');
      if (!token) {
        router.push('/login');
        return;
      }

      // Load user profile
      const profileRes = await fetch('/api/users/profile', {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      if (!profileRes.ok) throw new Error('Failed to load profile');
      const profileData = await profileRes.json();
      setUser(profileData.user);

      // Load transactions
      const txnRes = await fetch('/api/transactions', {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      if (txnRes.ok) {
        const txnData = await txnRes.json();
        setTransactions(txnData.transactions);
      }

      // Load weather for user's location (demo: London)
      const weatherRes = await fetch('/api/weather/london');
      if (weatherRes.ok) {
        const weatherData = await weatherRes.json();
        setWeather(weatherData.data);
      }

    } catch (error) {
      console.error('Dashboard error:', error);
      // Handle token expiration
      if (error.message.includes('token')) {
        localStorage.removeItem('accessToken');
        router.push('/login');
      }
    } finally {
      setLoading(false);
    }
  };

  const createTransaction = async (transactionData) => {
    try {
      const token = localStorage.getItem('accessToken');
      const response = await fetch('/api/transactions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`
        },
        body: JSON.stringify(transactionData)
      });

      if (response.ok) {
        // Reload transactions
        loadDashboardData();
      }
    } catch (error) {
      console.error('Create transaction error:', error);
    }
  };

  if (loading) {
    return (
      <Layout>
        <div className="flex justify-center items-center h-64">
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500"></div>
        </div>
      </Layout>
    );
  }

  return (
    <Layout>
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
        {/* Header */}
        <div className="mb-8">
          <h1 className="text-3xl font-bold text-gray-900">
            Welcome back, {user?.firstName || user?.email}!
          </h1>
          <p className="text-gray-600">
            Manage your transactions and monitor market conditions
          </p>
        </div>

        {/* Dashboard Grid */}
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          {/* Main Content */}
          <div className="lg:col-span-2">
            <TransactionList 
              transactions={transactions}
              onCreateTransaction={createTransaction}
            />
          </div>

          {/* Sidebar */}
          <div className="space-y-6">
            <WeatherWidget weather={weather} />
            
            {/* Quick Stats */}
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="text-lg font-semibold mb-4">Quick Stats</h3>
              <div className="space-y-3">
                <div className="flex justify-between">
                  <span className="text-gray-600">Total Transactions</span>
                  <span className="font-semibold">{transactions.length}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-gray-600">This Month</span>
                  <span className="font-semibold">
                    {transactions.filter(t => 
                      new Date(t.createdAt).getMonth() === new Date().getMonth()
                    ).length}
                  </span>
                </div>
                <div className="flex justify-between">
                  <span className="text-gray-600">Completed</span>
                  <span className="font-semibold text-green-600">
                    {transactions.filter(t => t.status === 'COMPLETED').length}
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Layout>
  );
}
```

### Production Docker Compose (`docker-compose.prod.yml`)

```yaml
version: '3.9'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8080
    depends_on:
      - backend
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/fullstack_db?schema=public
      - REDIS_URL=redis://redis:6379
      - FRONTEND_URL=http://frontend:3000
      - ACCESS_TOKEN_SECRET=${ACCESS_TOKEN_SECRET}
      - REFRESH_TOKEN_SECRET=${REFRESH_TOKEN_SECRET}
      - NODE_ENV=production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: fullstack_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d fullstack_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

  # Reverse proxy (production)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - frontend
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

## 7.3 Data Flow Example

Let's trace a complete user action through the system:

### User Creates a Transaction

1. **Frontend (Next.js)**:
   ```jsx
   // User fills form and submits
   const handleSubmit = async (formData) => {
     const response = await fetch('/api/transactions', {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         'Authorization': `Bearer ${token}`
       },
       body: JSON.stringify(formData)
     });
   };
   ```

2. **API Gateway (Express)**:
   ```javascript
   // Route: POST /api/transactions
   app.post('/api/transactions', authenticateToken, async (req, res) => {
     // Middleware validates JWT token
     // Controller processes request
     const result = await transactionService.create(req.user.id, req.body);
     res.json(result);
   });
   ```

3. **Business Logic (Service Layer)**:
   ```javascript
   // services/transactionService.js
   export async function create(userId, data) {
     // Validate business rules
     // Calculate fees and exchange rates
     // Check rate limits via Redis
     const rateLimit = await checkRateLimit(userId);
     if (!rateLimit.allowed) throw new Error('Rate limit exceeded');
     
     // Create transaction in database
     const transaction = await prisma.transaction.create({
       data: { userId, ...data }
     });
     
     // Cache user's recent transactions
     await cacheUserTransactions(userId);
     
     return transaction;
   }
   ```

4. **Database (PostgreSQL + Prisma)**:
   ```javascript
   // Prisma handles the SQL generation and execution
   const transaction = await prisma.transaction.create({
     data: {
       userId: 'user_123',
       sourceAmount: 100.00,
       sourceCurrency: 'USD',
       // ... other fields
     }
   });
   ```

5. **Cache Layer (Redis)**:
   ```javascript
   // Update cache with new transaction
   await redis.setex(
     `user:${userId}:transactions`,
     300, // 5 minutes TTL
     JSON.stringify(updatedTransactions)
   );
   ```

## 7.4 System Benefits

This integrated architecture provides:

### **Performance**
- **Redis caching** reduces database load
- **Connection pooling** optimizes database connections
- **CDN integration** for static assets

### **Scalability**
- **Horizontal scaling** with Docker containers
- **Database read replicas** for heavy read workloads
- **Microservices ready** architecture

### **Security**
- **JWT authentication** with refresh tokens
- **Rate limiting** prevents abuse
- **Input validation** at multiple layers
- **Audit logging** for compliance

### **Developer Experience**
- **Type safety** with Prisma and TypeScript
- **Hot reloading** in development
- **Consistent environments** with Docker
- **Easy deployment** with Docker Compose

### **Monitoring & Observability**
- **Health checks** for all services
- **Structured logging** with correlation IDs
- **Metrics collection** ready
- **Error tracking** integration points

This architecture demonstrates how modern full-stack frameworks work together to create robust, scalable applications that can grow from hackathon prototypes to production systems.

---

**Previous**: [DevOps: Docker and Deployment](06-devops.md) | **Next**: [Conclusion](08-conclusion.md)