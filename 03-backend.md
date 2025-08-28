# Chapter 3: Backend - Node.js and Express

## 3.1 Concepts

Node.js allows us to run JavaScript outside the browser, on the server. By default, it doesn't provide routing or middleware. That's where Express comes in.

Express is a minimal and flexible Node.js web application framework. It provides:

- **Routing** (defining endpoints such as `GET /login`, `POST /users`)
- **Middleware** (authentication, validation, logging, rate limiting)
- **Integration** with databases and ORMs like Prisma

## 3.2 API Layering

A clean architecture for APIs often looks like this:

- **Routes** – define URLs and map them to controllers
- **Controllers** – parse requests, validate data, and call services
- **Services** – contain the business logic
- **Database Layer** – Prisma functions that handle persistence

## 3.3 Visual Diagram

```
┌─────────────────────────┐
│   Routes (/login, /users) │
└─────────────┬───────────┘
              │
┌─────────────▼───────────┐
│ Controllers (Parse, Validate) │
└─────────────┬───────────┘
              │
┌─────────────▼───────────┐
│ Services (Business Logic) │
└─────────────┬───────────┘
              │
┌─────────────▼───────────┐
│   Database via Prisma   │
└─────────────────────────┘
```

## 3.4 Hands-On Exercise

1. **Initialize an Express app**:
   ```bash
   npm install express
   ```

2. **Add a health check route**:
   ```javascript
   import express from 'express';
   
   const app = express();
   
   app.get('/health', (_req, res) => res.json({ ok: true }));
   
   app.listen(8080, () => console.log('API running on 8080'));
   ```

3. **Add middleware for JSON parsing and CORS**:
   ```javascript
   import express from 'express';
   import cors from 'cors';
   
   const app = express();
   
   // Middleware
   app.use(express.json());
   app.use(cors());
   
   // Routes
   app.get('/health', (_req, res) => res.json({ ok: true }));
   
   app.listen(8080, () => console.log('API running on 8080'));
   ```

4. **Implement JWT authentication with access and refresh tokens** (see section 3.5 below)

## 3.5 JWT Authentication Flow

JWT (JSON Web Token) is used to securely transmit user identity between frontend and backend.

### Flow:
1. User logs in with email/password
2. Backend verifies credentials, generates an **access token** (short-lived) and a **refresh token** (long-lived)
3. Access token is sent in the Authorization header (Bearer scheme)
4. Refresh token is stored in a secure httpOnly cookie
5. Middleware checks access token; if expired, frontend calls refresh endpoint to get a new one

### Implementation:

```javascript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET;
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET;

// Mock user database
const users = [
  { id: 1, email: 'user@example.com', password: '$2b$10$...' } // hashed password
];

// Login endpoint
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;
  
  // Find user
  const user = users.find(u => u.email === email);
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
  
  // Set refresh token as httpOnly cookie
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
  });
  
  res.json({ accessToken, user: { id: user.id, email: user.email } });
});

// Middleware to verify access token
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN
  
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

// Protected route
app.get('/profile', authenticateToken, (req, res) => {
  res.json({ 
    message: 'Protected data', 
    user: req.user 
  });
});

// Refresh token endpoint
app.post('/auth/refresh', (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  
  if (!refreshToken) {
    return res.status(401).json({ error: 'Refresh token required' });
  }
  
  jwt.verify(refreshToken, REFRESH_TOKEN_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid refresh token' });
    }
    
    const accessToken = jwt.sign(
      { userId: user.userId },
      ACCESS_TOKEN_SECRET,
      { expiresIn: '15m' }
    );
    
    res.json({ accessToken });
  });
});
```

## 3.6 Mini Project

Build a weather API with authentication:

- **POST /auth/login** – returns JWT tokens after verifying a dummy user
- **GET /weather/:city** – returns weather data only if a valid access token is provided

### Complete Implementation:

```javascript
import express from 'express';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import cookieParser from 'cookie-parser';
import cors from 'cors';

const app = express();

// Middleware
app.use(express.json());
app.use(cookieParser());
app.use(cors({ credentials: true, origin: 'http://localhost:3000' }));

// Environment variables (use .env in real app)
const ACCESS_TOKEN_SECRET = 'your-access-token-secret';
const REFRESH_TOKEN_SECRET = 'your-refresh-token-secret';

// Mock user (password: "password123")
const users = [
  { 
    id: 1, 
    email: 'demo@example.com', 
    password: '$2b$10$rOOjXkgBkU7zOcKKnKzKOeQQQQQQQQQQQQQQQQQQQQQQQQ' // hashed "password123"
  }
];

// Routes
app.get('/health', (req, res) => {
  res.json({ ok: true, timestamp: new Date().toISOString() });
});

app.post('/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Validate input
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }
    
    // Find user (in real app, query database)
    const user = users.find(u => u.email === email);
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // For demo purposes, accept "password123" as valid
    if (password !== 'password123') {
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
    
    // Set refresh token cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: false, // set to true in production with HTTPS
      sameSite: 'lax',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });
    
    res.json({ 
      accessToken, 
      user: { id: user.id, email: user.email } 
    });
    
  } catch (error) {
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

// Mock weather data
const weatherData = {
  london: { temp: 15, condition: 'Cloudy', humidity: 78 },
  newyork: { temp: 22, condition: 'Sunny', humidity: 65 },
  tokyo: { temp: 18, condition: 'Rainy', humidity: 82 },
  sydney: { temp: 25, condition: 'Sunny', humidity: 60 }
};

app.get('/weather/:city', authenticateToken, (req, res) => {
  const { city } = req.params;
  const cityKey = city.toLowerCase();
  
  const weather = weatherData[cityKey] || {
    temp: Math.floor(Math.random() * 30) + 5,
    condition: 'Unknown',
    humidity: Math.floor(Math.random() * 40) + 40
  };
  
  res.json({
    city: city,
    ...weather,
    timestamp: new Date().toISOString(),
    user: req.user.email
  });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`API server running on port ${PORT}`);
});
```

### Testing the API:

1. **Start the server**:
   ```bash
   node server.js
   ```

2. **Test login**:
   ```bash
   curl -X POST http://localhost:8080/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "demo@example.com", "password": "password123"}'
   ```

3. **Test weather route** (use the token from login response):
   ```bash
   curl -X GET http://localhost:8080/weather/london \
     -H "Authorization: Bearer YOUR_ACCESS_TOKEN_HERE"
   ```

This project demonstrates authentication and clean API layering with a simple, relatable weather service that students can easily understand and extend.

---

**Previous**: [Frontend: React to Next.js](02-frontend.md) | **Next**: [Database: PostgreSQL and Prisma](04-database.md)