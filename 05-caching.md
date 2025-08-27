# Chapter 5: Caching - Redis

## 5.1 Concepts

Redis is an open-source, in-memory data store often described as a "key–value database." Unlike relational databases that write primarily to disk, Redis keeps data in memory, making it extremely fast. It is frequently used for:

- **Caching** – storing API responses, database query results, or computations
- **Sessions** – managing logged-in user sessions
- **Rate Limiting** – counting API requests per user/IP
- **Message Queues** – building producer–consumer pipelines

Redis supports **TTL (time-to-live)** values, meaning data can expire after a set duration. This is particularly useful for temporary data like authentication tokens or cached API results.

## 5.2 Visual Diagram

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ API Request │───▶│ Redis Cache │───▶│ PostgreSQL  │
└─────────────┘    └─────────────┘    │  Database   │
                          │           └─────────────┘
                          ▼
                   ┌─────────────┐
                   │ Cache Hit   │
                   │ Return Fast │
                   └─────────────┘
```

**Flow**:
1. API receives request
2. Check Redis cache first
3. If **cache hit**: return data immediately
4. If **cache miss**: query database, store in cache, return data

## 5.3 Example: Caching Weather by City

For a student dashboard that shows current weather, you can cache responses per city to avoid hitting the weather API for every request:

1. Build a cache key like `weather:city:London`
2. Use a shorter TTL (e.g., 300 seconds) because weather changes frequently
3. On a miss, fetch from a public weather API, store it, and serve

## 5.4 Code Snippet

```javascript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

export async function getWeather(city) {
  const key = `weather:city:${city.toLowerCase()}`;
  
  // Try to get from cache first
  const cached = await redis.get(key);
  if (cached) {
    console.log(`Cache hit for ${city}`);
    return JSON.parse(cached);
  }
  
  console.log(`Cache miss for ${city}, fetching from API`);
  
  // Fetch from external API (replace with real weather API)
  const response = await fetch(
    `https://api.open-meteo.com/v1/forecast?latitude=51.5072&longitude=0.1276&current_weather=true`
  );
  const data = await response.json();
  
  // Add city info to response
  const weatherData = {
    city,
    temperature: data.current_weather.temperature,
    windspeed: data.current_weather.windspeed,
    timestamp: new Date().toISOString()
  };
  
  // Store in cache with 5-minute TTL
  await redis.set(key, JSON.stringify(weatherData), 'EX', 300);
  
  return weatherData;
}
```

## 5.5 Hands-On Exercise

### Step 1: Install Redis Locally

Using Docker (recommended):
```bash
docker run -d -p 6379:6379 --name redis redis:7
```

Or install directly:
- **macOS**: `brew install redis && brew services start redis`
- **Ubuntu**: `sudo apt install redis-server`
- **Windows**: Use Docker or WSL

### Step 2: Install Redis Client

```bash
npm install ioredis
```

### Step 3: Create Weather Caching Service

Create `weather-service.js`:

```javascript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

// Mock weather data for different cities
const mockWeatherData = {
  london: { temp: 15, condition: 'Cloudy', humidity: 78 },
  newyork: { temp: 22, condition: 'Sunny', humidity: 65 },
  tokyo: { temp: 18, condition: 'Rainy', humidity: 82 },
  sydney: { temp: 25, condition: 'Sunny', humidity: 60 }
};

export async function getWeather(city) {
  const key = `weather:city:${city.toLowerCase()}`;
  
  try {
    // Check cache first
    const cached = await redis.get(key);
    if (cached) {
      console.log(`✅ Cache HIT for ${city}`);
      return { ...JSON.parse(cached), cached: true };
    }
    
    console.log(`❌ Cache MISS for ${city}, fetching data...`);
    
    // Simulate API delay
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Get mock data or default
    const cityKey = city.toLowerCase();
    const weatherData = mockWeatherData[cityKey] || {
      temp: Math.floor(Math.random() * 30) + 5,
      condition: 'Unknown',
      humidity: Math.floor(Math.random() * 40) + 40
    };
    
    const response = {
      city,
      ...weatherData,
      timestamp: new Date().toISOString(),
      cached: false
    };
    
    // Store in cache with TTL
    await redis.set(key, JSON.stringify(response), 'EX', 300); // 5 minutes
    
    return response;
    
  } catch (error) {
    console.error('Redis error:', error);
    // Fallback to direct data without caching
    return {
      city,
      temp: 20,
      condition: 'Unknown',
      humidity: 50,
      timestamp: new Date().toISOString(),
      cached: false,
      error: 'Cache unavailable'
    };
  }
}

export async function clearWeatherCache(city) {
  const key = `weather:city:${city.toLowerCase()}`;
  const result = await redis.del(key);
  return result > 0;
}

export async function getWeatherCacheInfo() {
  const keys = await redis.keys('weather:city:*');
  const info = [];
  
  for (const key of keys) {
    const ttl = await redis.ttl(key);
    const data = await redis.get(key);
    info.push({
      key,
      ttl,
      data: JSON.parse(data)
    });
  }
  
  return info;
}
```

### Step 4: Test the Caching

Create `test-weather.js`:

```javascript
import { getWeather, clearWeatherCache, getWeatherCacheInfo } from './weather-service.js';

async function testWeatherCaching() {
  console.log('🌤️  Testing Weather Caching\n');
  
  const cities = ['London', 'New York', 'Tokyo'];
  
  // First requests (should be cache misses)
  console.log('--- First requests (cache misses) ---');
  for (const city of cities) {
    const start = Date.now();
    const weather = await getWeather(city);
    const duration = Date.now() - start;
    console.log(`${city}: ${weather.temp}°C, ${weather.condition} (${duration}ms) ${weather.cached ? '🟢' : '🔴'}`);
  }
  
  console.log('\n--- Second requests (should be cache hits) ---');
  // Second requests (should be cache hits)
  for (const city of cities) {
    const start = Date.now();
    const weather = await getWeather(city);
    const duration = Date.now() - start;
    console.log(`${city}: ${weather.temp}°C, ${weather.condition} (${duration}ms) ${weather.cached ? '🟢' : '🔴'}`);
  }
  
  // Show cache info
  console.log('\n--- Cache Information ---');
  const cacheInfo = await getWeatherCacheInfo();
  cacheInfo.forEach(info => {
    console.log(`${info.key}: TTL ${info.ttl}s, Data: ${info.data.city} ${info.data.temp}°C`);
  });
  
  // Clear cache for one city
  console.log('\n--- Clearing London cache ---');
  const cleared = await clearWeatherCache('London');
  console.log(`London cache cleared: ${cleared}`);
  
  // Test London again (should be cache miss)
  console.log('\n--- London after cache clear ---');
  const start = Date.now();
  const weather = await getWeather('London');
  const duration = Date.now() - start;
  console.log(`London: ${weather.temp}°C, ${weather.condition} (${duration}ms) ${weather.cached ? '🟢' : '🔴'}`);
  
  process.exit(0);
}

testWeatherCaching().catch(console.error);
```

Run the test:
```bash
node test-weather.js
```

Expected output:
```
🌤️  Testing Weather Caching

--- First requests (cache misses) ---
London: 15°C, Cloudy (1002ms) 🔴
New York: 22°C, Sunny (1001ms) 🔴
Tokyo: 18°C, Rainy (1003ms) 🔴

--- Second requests (should be cache hits) ---
London: 15°C, Cloudy (2ms) 🟢
New York: 22°C, Sunny (1ms) 🟢
Tokyo: 18°C, Rainy (2ms) 🟢
```

### Step 5: Integrate with Express API

Add caching to your Express server:

```javascript
import express from 'express';
import { getWeather, clearWeatherCache } from './weather-service.js';

const app = express();
app.use(express.json());

// Weather endpoint with caching
app.get('/weather/:city', async (req, res) => {
  try {
    const { city } = req.params;
    const weather = await getWeather(city);
    
    res.json({
      success: true,
      data: weather,
      cached: weather.cached
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Clear cache endpoint
app.delete('/weather/:city/cache', async (req, res) => {
  try {
    const { city } = req.params;
    const cleared = await clearWeatherCache(city);
    
    res.json({
      success: true,
      cleared,
      message: cleared ? 'Cache cleared' : 'No cache found'
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

app.listen(8080, () => {
  console.log('Weather API running on http://localhost:8080');
  console.log('Try: GET /weather/london');
  console.log('Try: DELETE /weather/london/cache');
});
```

### Step 6: Advanced Caching Patterns

Create `advanced-caching.js` to explore more patterns:

```javascript
import Redis from 'ioredis';

const redis = new Redis();

// Pattern 1: Cache with automatic refresh
export async function getCachedWithRefresh(key, fetchFn, ttl = 300) {
  const cached = await redis.get(key);
  
  if (cached) {
    const data = JSON.parse(cached);
    
    // If data is more than half expired, refresh in background
    const remainingTtl = await redis.ttl(key);
    if (remainingTtl < ttl / 2) {
      // Refresh in background (don't await)
      fetchFn().then(newData => {
        redis.set(key, JSON.stringify(newData), 'EX', ttl);
      }).catch(console.error);
    }
    
    return data;
  }
  
  // Cache miss - fetch and store
  const data = await fetchFn();
  await redis.set(key, JSON.stringify(data), 'EX', ttl);
  return data;
}

// Pattern 2: Rate limiting
export async function checkRateLimit(userId, maxRequests = 100, windowSeconds = 3600) {
  const key = `rate_limit:${userId}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    // First request in window - set expiration
    await redis.expire(key, windowSeconds);
  }
  
  return {
    allowed: current <= maxRequests,
    current,
    remaining: Math.max(0, maxRequests - current),
    resetTime: Date.now() + (windowSeconds * 1000)
  };
}

// Pattern 3: Distributed locking
export async function withLock(lockKey, fn, ttl = 10) {
  const lockValue = Math.random().toString(36);
  const acquired = await redis.set(lockKey, lockValue, 'PX', ttl * 1000, 'NX');
  
  if (!acquired) {
    throw new Error('Could not acquire lock');
  }
  
  try {
    return await fn();
  } finally {
    // Release lock only if we still own it
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    await redis.eval(script, 1, lockKey, lockValue);
  }
}
```

## Key Benefits Demonstrated

- **Performance**: Sub-millisecond response times for cached data
- **Cost Reduction**: Fewer external API calls
- **Scalability**: Reduced database load
- **Flexibility**: TTL management for different data types
- **Reliability**: Graceful fallback when cache is unavailable

## Common Use Cases

1. **API Response Caching**: Store expensive computation results
2. **Session Management**: User login state and preferences
3. **Rate Limiting**: Prevent API abuse
4. **Real-time Features**: Pub/sub for chat, notifications
5. **Leaderboards**: Sorted sets for rankings

---

**Previous**: [Database: PostgreSQL and Prisma](04-database.md) | **Next**: [DevOps: Docker and Deployment](06-devops.md)