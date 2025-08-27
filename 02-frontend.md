# Chapter 2: Frontend - React to Next.js

## 2.1 Concepts

React is a UI library that focuses only on the "view" layer of applications. It provides reusable components, props for passing data, and state for dynamic changes. However, React by itself is limited:

- No built-in routing (you need libraries like React Router)
- SEO challenges due to client-side rendering
- Configuration heavy (build tools like Vite or CRA are needed)

Next.js builds on React and addresses these gaps:

- **File-based routing**: pages are created by placing files in the `pages/` directory
- **Server-Side Rendering (SSR)**, Static Site Generation (SSG), and Incremental Static Regeneration (ISR)
- **API routes**: build backend endpoints inside the same project
- **Server Components** (default) vs Client Components (with "use client")
- **Optimizations** like Image component, Middleware, and production defaults

## 2.2 Visual Diagram

```
React (UI Library)          →          Next.js (Full Framework)
├── Components                         ├── Components
├── Props & State                      ├── Props & State
├── Manual Routing Setup               ├── File-based Routing
├── Client-side Only                   ├── SSR/SSG/ISR
├── External Build Tools               ├── Built-in Optimization
└── Separate Backend Needed            └── API Routes Included
```

## 2.3 Key Differences

- **React** renders only in the browser; **Next.js** can render on both server and client
- **React** requires external tools for routing; **Next.js** has built-in routing
- **React** projects often need separate backends; **Next.js** can serve APIs from the same codebase

## 2.4 Hands-On Exercise

1. **Create a new app**: 
   ```bash
   npx create-next-app myapp
   ```

2. **Add a login page**: create `pages/login.js` with a simple form:
   ```jsx
   export default function Login() {
     return (
       <div>
         <h1>Login</h1>
         <form>
           <input type="text" placeholder="Username" />
           <input type="password" placeholder="Password" />
           <button type="submit">Login</button>
         </form>
       </div>
     );
   }
   ```

3. **Add a dashboard page**: create `pages/dashboard.js` that fetches mock data server-side:
   ```jsx
   export default function Dashboard({ data }) {
     return (
       <div>
         <h1>Dashboard</h1>
         <pre>{JSON.stringify(data, null, 2)}</pre>
       </div>
     );
   }

   export async function getServerSideProps() {
     // This runs on the server
     const data = { message: "Hello from server!", timestamp: new Date().toISOString() };
     return { props: { data } };
   }
   ```

4. **Run the app**: 
   ```bash
   npm run dev
   ```
   Open http://localhost:3000

## 2.5 Mini Project

Build a two-page app:

- **A login page** (client component) with a simple username form
- **A dashboard page** (server component) that fetches current cryptocurrency prices from a public API

### Implementation Steps:

1. **Login Page** (`pages/login.js`):
   ```jsx
   'use client';
   import { useState } from 'react';
   import { useRouter } from 'next/router';

   export default function Login() {
     const [username, setUsername] = useState('');
     const router = useRouter();

     const handleSubmit = (e) => {
       e.preventDefault();
       if (username) {
         localStorage.setItem('username', username);
         router.push('/dashboard');
       }
     };

     return (
       <div style={{ padding: '2rem', maxWidth: '400px', margin: '0 auto' }}>
         <h1>Login</h1>
         <form onSubmit={handleSubmit}>
           <input
             type="text"
             placeholder="Enter username"
             value={username}
             onChange={(e) => setUsername(e.target.value)}
             style={{ width: '100%', padding: '0.5rem', marginBottom: '1rem' }}
           />
           <button type="submit" style={{ width: '100%', padding: '0.5rem' }}>
             Login
           </button>
         </form>
       </div>
     );
   }
   ```

2. **Dashboard Page** (`pages/dashboard.js`):
   ```jsx
   export default function Dashboard({ cryptoData, username }) {
     return (
       <div style={{ padding: '2rem' }}>
         <h1>Welcome, {username}!</h1>
         <h2>Cryptocurrency Prices</h2>
         <div>
           {cryptoData.map((coin) => (
             <div key={coin.id} style={{ margin: '1rem 0', padding: '1rem', border: '1px solid #ccc' }}>
               <h3>{coin.name} ({coin.symbol.toUpperCase()})</h3>
               <p>Price: ${coin.current_price}</p>
               <p>24h Change: {coin.price_change_percentage_24h?.toFixed(2)}%</p>
             </div>
           ))}
         </div>
       </div>
     );
   }

   export async function getServerSideProps(context) {
     try {
       // Fetch crypto data from CoinGecko API
       const response = await fetch(
         'https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=5&page=1'
       );
       const cryptoData = await response.json();

       return {
         props: {
           cryptoData,
           username: 'User' // In real app, get from session/auth
         }
       };
     } catch (error) {
       return {
         props: {
           cryptoData: [],
           username: 'User'
         }
       };
     }
   }
   ```

This project shows how Next.js combines frontend and backend features into one cohesive workflow.

## Key Benefits Demonstrated

- **File-based routing**: No need to configure routes manually
- **Server-side rendering**: SEO-friendly pages with pre-fetched data
- **API integration**: Easy to fetch external data during build/render
- **Client-side interactivity**: Forms and state management work seamlessly

---

**Previous**: [Introduction](01-introduction.md) | **Next**: [Backend: Node.js and Express](03-backend.md)