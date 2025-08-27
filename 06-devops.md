# Chapter 6: DevOps - Docker and Deployment

## 6.1 Docker Compose

Now let's wire multiple services together using Docker Compose. This allows us to run our entire stack (API, PostgreSQL, and Redis) with a single command.

### Complete Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.9'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb?schema=public
      - REDIS_URL=redis://redis:6379
      - ACCESS_TOKEN_SECRET=your-access-token-secret
      - REFRESH_TOKEN_SECRET=your-refresh-token-secret
      - NODE_ENV=development
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Optional: Redis GUI
  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8081:8081"
    depends_on:
      - redis

volumes:
  postgres_data:
  redis_data:
```

### Dockerfile for the API

Create `Dockerfile`:

```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Start the application
CMD ["npm", "start"]
```

### Package.json Scripts

Update your `package.json`:

```json
{
  "name": "fullstack-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js",
    "build": "echo 'No build step needed'",
    "db:migrate": "npx prisma migrate dev",
    "db:generate": "npx prisma generate",
    "db:studio": "npx prisma studio",
    "db:seed": "node prisma/seed.js"
  },
  "dependencies": {
    "@prisma/client": "^5.0.0",
    "express": "^4.18.0",
    "jsonwebtoken": "^9.0.0",
    "bcrypt": "^5.1.0",
    "ioredis": "^5.3.0",
    "cookie-parser": "^1.4.0",
    "cors": "^2.8.0"
  },
  "devDependencies": {
    "prisma": "^5.0.0",
    "nodemon": "^3.0.0"
  }
}
```

### Database Initialization

Create `init.sql` for initial database setup:

```sql
-- This file runs when PostgreSQL container starts
-- Add any initial database setup here

-- Create extensions if needed
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- You can add initial data here, but Prisma migrations are preferred
```

### Environment Configuration

Create `.env.example`:

```env
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# Redis
REDIS_URL="redis://localhost:6379"

# JWT Secrets (generate strong secrets for production)
ACCESS_TOKEN_SECRET="your-super-secret-access-token-key"
REFRESH_TOKEN_SECRET="your-super-secret-refresh-token-key"

# Environment
NODE_ENV="development"
PORT=8080
```

## 6.2 Deployment Options

### Popular Beginner-Friendly Platforms

1. **Railway** – Simple, free tier for students, one-click deploys
2. **Render** – Great free tier, supports web services and background workers
3. **Neon** – Managed PostgreSQL with generous free tier
4. **Upstash** – Managed Redis with serverless pricing

### Deployment Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Railway/      │    │      Neon       │    │    Upstash     │
│   Render        │    │   (PostgreSQL)  │    │    (Redis)     │
│   (API Server)  │───▶│                 │    │                │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Typical Deployment Steps

1. **Push your code to GitHub**
2. **Connect the repo to Railway/Render**
3. **Add environment variables** (DB connection string, Redis URL, JWT secret)
4. **Deploy** – the platform builds the Docker image automatically

## 6.3 Hands-On Exercise

### Step 1: Install Docker Desktop

Download and install Docker Desktop for your platform:
- **macOS**: https://docs.docker.com/desktop/mac/install/
- **Windows**: https://docs.docker.com/desktop/windows/install/
- **Linux**: https://docs.docker.com/desktop/linux/install/

### Step 2: Create Production-Ready Dockerfile

Create an optimized `Dockerfile`:

```dockerfile
# Multi-stage build for smaller production image
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install all dependencies (including dev)
RUN npm ci

# Generate Prisma client
RUN npx prisma generate

# Copy source code
COPY . .

# Production stage
FROM node:18-alpine AS production

# Install curl for health checks
RUN apk add --no-cache curl

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy built application from builder stage
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/prisma ./prisma
COPY --chown=nextjs:nodejs . .

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Start the application
CMD ["npm", "start"]
```

### Step 3: Create Docker Compose for Development

Create `docker-compose.dev.yml`:

```yaml
version: '3.9'

services:
  api:
    build:
      context: .
      target: builder  # Use builder stage for development
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb?schema=public
      - REDIS_URL=redis://redis:6379
      - ACCESS_TOKEN_SECRET=dev-access-secret
      - REFRESH_TOKEN_SECRET=dev-refresh-secret
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  # Database management UI
  adminer:
    image: adminer
    ports:
      - "8082:8080"
    depends_on:
      - db

volumes:
  postgres_data:
  redis_data:
```

### Step 4: Run Locally with Docker Compose

```bash
# Start all services
docker-compose -f docker-compose.dev.yml up --build

# Or run in background
docker-compose -f docker-compose.dev.yml up -d --build

# View logs
docker-compose -f docker-compose.dev.yml logs -f api

# Stop all services
docker-compose -f docker-compose.dev.yml down

# Stop and remove volumes (clean slate)
docker-compose -f docker-compose.dev.yml down -v
```

### Step 5: Deploy to Railway

1. **Create Railway account**: https://railway.app/
2. **Install Railway CLI**:
   ```bash
   npm install -g @railway/cli
   railway login
   ```

3. **Initialize Railway project**:
   ```bash
   railway init
   ```

4. **Add services**:
   ```bash
   # Add PostgreSQL
   railway add postgresql
   
   # Add Redis
   railway add redis
   ```

5. **Deploy your app**:
   ```bash
   railway up
   ```

6. **Set environment variables** in Railway dashboard:
   - `DATABASE_URL` (automatically set by Railway PostgreSQL)
   - `REDIS_URL` (automatically set by Railway Redis)
   - `ACCESS_TOKEN_SECRET`
   - `REFRESH_TOKEN_SECRET`
   - `NODE_ENV=production`

### Step 6: Deploy to Render

1. **Create Render account**: https://render.com/
2. **Connect GitHub repository**
3. **Create Web Service**:
   - **Build Command**: `npm install && npx prisma generate`
   - **Start Command**: `npm start`
   - **Environment**: Node

4. **Add PostgreSQL database**:
   - Create new PostgreSQL service
   - Copy connection string to `DATABASE_URL`

5. **Add Redis** (using Upstash):
   - Create Upstash account: https://upstash.com/
   - Create Redis database
   - Copy connection string to `REDIS_URL`

## 6.4 Mini Project

Take the weather app example from Chapter 5 and make it production-ready:

### Complete Weather App with Docker

Create `weather-app/` directory with the following structure:

```
weather-app/
├── Dockerfile
├── docker-compose.yml
├── package.json
├── server.js
├── weather-service.js
├── prisma/
│   └── schema.prisma
└── .env.example
```

### Enhanced Server (`server.js`)

```javascript
import express from 'express';
import cors from 'cors';
import { PrismaClient } from '@prisma/client';
import { getWeather, clearWeatherCache } from './weather-service.js';

const app = express();
const prisma = new PrismaClient();

// Middleware
app.use(express.json());
app.use(cors());

// Health check
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await prisma.$queryRaw`SELECT 1`;
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        database: 'connected',
        redis: 'connected'
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

// Weather endpoints
app.get('/weather/:city', async (req, res) => {
  try {
    const { city } = req.params;
    const weather = await getWeather(city);
    
    // Log request to database
    await prisma.weatherRequest.create({
      data: {
        city: city.toLowerCase(),
        cached: weather.cached,
        timestamp: new Date()
      }
    });
    
    res.json({
      success: true,
      data: weather
    });
  } catch (error) {
    console.error('Weather API error:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to fetch weather data'
    });
  }
});

// Analytics endpoint
app.get('/analytics', async (req, res) => {
  try {
    const stats = await prisma.weatherRequest.groupBy({
      by: ['city'],
      _count: {
        city: true
      },
      orderBy: {
        _count: {
          city: 'desc'
        }
      },
      take: 10
    });
    
    res.json({
      success: true,
      data: stats
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');
  await prisma.$disconnect();
  process.exit(0);
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`🌤️  Weather API running on port ${PORT}`);
  console.log(`📊 Health check: http://localhost:${PORT}/health`);
  console.log(`🌍 Try: http://localhost:${PORT}/weather/london`);
});
```

### Prisma Schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model WeatherRequest {
  id        String   @id @default(cuid())
  city      String
  cached    Boolean  @default(false)
  timestamp DateTime @default(now())
  
  @@map("weather_requests")
}
```

### Docker Compose (`docker-compose.yml`)

```yaml
version: '3.9'

services:
  weather-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/weatherdb?schema=public
      - REDIS_URL=redis://redis:6379
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
      POSTGRES_DB: weatherdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d weatherdb"]
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

volumes:
  postgres_data:
  redis_data:
```

### Deployment Commands

```bash
# Local development
docker-compose up --build

# Production deployment
docker-compose -f docker-compose.yml up -d --build

# View logs
docker-compose logs -f weather-api

# Scale the API (multiple instances)
docker-compose up --scale weather-api=3

# Update and redeploy
docker-compose pull
docker-compose up -d --build
```

This exercise ties together Docker concepts with a practical, student-friendly use case that demonstrates:

- **Containerization** of a full-stack application
- **Service orchestration** with Docker Compose
- **Production readiness** with health checks and graceful shutdown
- **Scalability** with multiple service instances
- **Monitoring** with request analytics

## Key Benefits Demonstrated

- **Consistency**: "Works on my machine" → "Works everywhere"
- **Isolation**: Services run in separate containers
- **Scalability**: Easy to scale individual services
- **Portability**: Deploy anywhere Docker runs
- **Development Speed**: One command to start entire stack

---

**Previous**: [Caching: Redis](05-caching.md) | **Next**: [Putting It Together](07-architecture.md)