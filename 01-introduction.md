# Chapter 1: Introduction

Software development has changed dramatically over the past decade. In the early 2010s, developers often combined jQuery for frontend interactivity, PHP or Java servlets for backend logic, and MySQL for data. Each piece worked in isolation, often requiring manual setup, wiring, and endless debugging of environments.

By 2025, the picture looks very different. Today, developers have access to modern full stack frameworks that reduce boilerplate, improve scalability, and allow small teams to build production-grade systems quickly. These frameworks are not just convenient—they are strategic tools for hackathons, startups, and enterprise developers alike.

## 1.1 Why Full Stack Matters

A "full stack" application connects three critical layers:

1. **Frontend** – What users see and interact with (React, Next.js)
2. **Backend** – Logic, authentication, APIs (Node.js, Express)
3. **Database** – Persistent storage (PostgreSQL, Prisma)

Supporting technologies like Redis, Docker, and cloud platforms ensure the system is fast, consistent, and deployable anywhere.

## 1.2 The Evolution of Stacks

**Yesterday**: MERN (MongoDB, Express, React, Node) was popular in 2015. It worked but required a lot of manual configuration, and MongoDB's flexibility sometimes caused data consistency issues.

**Today**: A modern stack often looks like Next.js (frontend + light backend) + Express (API layer) + Prisma (DB access) + Postgres (data) + Redis (cache) + Docker (containerization). These pieces form a cohesive workflow that lets you move from idea to deploy rapidly.

## 1.3 A Story: Building at a Hackathon

Imagine you are at a 48-hour hackathon, trying to build a fintech app to send money across borders. Time is short, and every hour counts.

- **With React alone**, you'd spend hours configuring routing and SEO. Next.js solves this in minutes.
- **With raw SQL**, you'd be writing queries from scratch. Prisma generates type-safe queries automatically.
- **Without Redis**, your app would repeatedly call external APIs. With caching, you save costs and speed up responses.
- **Without Docker**, "it works on my machine" becomes a nightmare when deploying. Containers guarantee consistency.

By the end of the hackathon, teams using modern stacks spend more time innovating—and less time fighting boilerplate.

## 1.4 Who This Book is For

This book is designed for:

- **Students** learning how real-world applications are built
- **Hackathon builders** who need to go from idea to demo in a weekend
- **Startup developers** building MVPs that can scale

Each chapter offers:
- Clear explanations of the technology
- Visual diagrams
- Hands-on exercises
- Mini projects to practice

## 1.5 Looking Ahead

In the following chapters, we'll break down each layer of the modern stack:

- **Frontend**: React to Next.js
- **Backend**: Node.js with Express
- **Database**: PostgreSQL with Prisma
- **Caching**: Redis
- **Deployment**: Docker and managed cloud platforms

By the end, you'll not only understand how these tools work individually, but also how to stitch them together into one cohesive, production-ready system.

## Key Takeaway

Modern full stack frameworks are about speed, reliability, and focus. They help you spend less time on setup and more time on building what matters.

---

**Next**: [Frontend: React to Next.js](02-frontend.md)