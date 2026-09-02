# Prisma ORM with PostgreSQL — Express & JavaScript Masterclass

## Objective
Learn how to build modern, type-safe backend applications using **Prisma ORM** with **PostgreSQL**, **JavaScript (ES Modules)**, and **Express.js**. This course is designed so a **complete beginner** can follow along, step-by-step, without getting lost.

---

## Table of Contents
1. [What is Prisma & Why use an ORM?](#1-what-is-prisma--why-use-an-orm)
2. [Step 1: Setting up Node.js & Express Project](#step-1-setting-up-nodejs--express-project)
3. [Step 2: Initializing Prisma & Database Connection](#step-2-initializing-prisma--database-connection)
4. [Step 3: Designing your Database Schema (`schema.prisma`)](#step-3-designing-your-database-schema-schemaprisma)
5. [Step 4: Running Database Migrations](#step-4-running-database-migrations)
6. [Step 5: Setting up the Prisma Client](#step-5-setting-up-the-prisma-client)
7. [Step 6: User Authentication (Password Hashing & JWT)](#step-6-user-authentication-password-hashing--jwt)
8. [Step 7: Express Middleware & Protected Routes](#step-7-express-middleware--protected-routes)
9. [Step 8: Full Express API Endpoints (CRUD + Relations)](#step-8-full-express-api-endpoints-crud--relations)
10. [Step 9: Advanced Features (Transactions, Filtering, Pagination)](#step-9-advanced-features-transactions-filtering-pagination)
11. [Step 10: Testing Your API with Curl](#step-10-testing-your-api-with-curl)
12. [Common Pitfalls & Troubleshooting Guide](#common-pitfalls--troubleshooting-guide)

---

## 1. What is Prisma & Why use an ORM?

### What is an ORM?
An **ORM (Object-Relational Mapper)** is a tool that acts as a translator between your JavaScript code and your PostgreSQL database. 

Instead of writing raw SQL queries like:
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```
You write JavaScript methods:
```javascript
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' }
});
```

### Why use Prisma over Raw SQL for JavaScript apps?
- **No SQL Syntax Errors:** Prisma generates autocomplete options directly in VS Code.
- **Single Source of Truth:** Your tables and relationships are declared in one clear file (`schema.prisma`).
- **Automated Migrations:** Prisma writes and runs SQL migrations for you whenever your data model changes.
- **Built-in Security:** All queries automatically use parameterized inputs, eliminating **SQL Injection** security risks.

---

## Step 1: Setting up Node.js & Express Project

Let me show you how to build a clean, complete Express application with authentication from scratch.

### 1. Initialize project folder
Open your terminal and run:
```bash
mkdir prisma-postgres-express
cd prisma-postgres-express
npm init -y
```

### 2. Install dependencies
```bash
# Production dependencies: Express server, dotenv, password hashing (bcryptjs) & JWT tokens (jsonwebtoken)
npm install express dotenv bcryptjs jsonwebtoken

# Prisma runtime client
npm install @prisma/client

# Prisma CLI (Developer tool)
npm install prisma --save-dev
```

### 3. Enable ES Modules syntax (`import` statements)
Open `package.json` and add `"type": "module"`:
```json
{
  "name": "prisma-postgres-express",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node index.js"
  }
}
```

---

## Step 2: Initializing Prisma & Database Connection

Initialize Prisma in your project:
```bash
npx prisma init
```

This creates two files:
1. `prisma/schema.prisma`: The central configuration file.
2. `.env`: Environment variables file holding secret keys and database URLs.

### Configure `.env`
Open `.env` and set your PostgreSQL database URL along with a secret key for signing JSON Web Tokens:
```env
# Format: postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/school_db?schema=public"

# Secret key used to encrypt/verify JWT tokens (use a long secret string in production)
JWT_SECRET="super-secret-key-12345"
```
*(If using cloud hosting like Neon or Supabase, paste the connection string provided in your cloud dashboard).*

---

## Step 3: Designing your Database Schema (`schema.prisma`)

Open `prisma/schema.prisma` and replace its contents with:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Model 1: User (with hashed password for Authentication)
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String   // Hashed password string
  name      String
  role      String   @default("STUDENT")
  createdAt DateTime @default(now())

  // Relationship: One user can write many Posts
  posts     Post[]
}

// Model 2: Post (e.g., Blog post or Assignment)
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  createdAt DateTime @default(now())

  // Foreign key relationship pointing to User model
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
}
```

---

## Step 4: Running Database Migrations

Run this command to sync your PostgreSQL database:
```bash
npx prisma migrate dev --name add_auth_user_model
```

---

## Step 5: Setting up the Prisma Client

Create `db.js`:

```javascript
// db.js
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export default prisma;
```

---

## Step 6: User Authentication (Password Hashing & JWT)

Authentication requires two main steps:
1. **User Registration:** Hash password using `bcryptjs` before saving to Postgres via Prisma.
2. **User Login:** Compare incoming password against stored hash, then sign and return a **JWT token**.

Create `auth.js` for registration and login logic:

```javascript
// auth.js
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import prisma from './db.js';

// 1. REGISTER A NEW USER
export const register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    if (!email || !password || !name) {
      return res.status(400).json({ error: 'Name, email, and password are required.' });
    }

    // Hash password with salt rounds (10)
    const hashedPassword = await bcrypt.hash(password, 10);

    // Save to PostgreSQL using Prisma
    const user = await prisma.user.create({
      data: {
        name,
        email,
        password: hashedPassword,
        role: role || 'STUDENT'
      },
      // Exclude password from the returned object for security
      select: { id: true, name: true, email: true, role: true, createdAt: true }
    });

    res.status(201).json({ message: 'User registered successfully', user });
  } catch (error) {
    if (error.code === 'P2002') {
      return res.status(400).json({ error: 'An account with this email already exists.' });
    }
    res.status(500).json({ error: error.message });
  }
};

// 2. LOGIN USER & GENERATE JWT
export const login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Find user in database
    const user = await prisma.user.findUnique({ where: { email } });

    // Security practice: Use generic error message so attackers can't guess valid emails
    if (!user) {
      return res.status(401).json({ error: 'Invalid email or password.' });
    }

    // Verify password against hash
    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) {
      return res.status(401).json({ error: 'Invalid email or password.' });
    }

    // Generate JWT access token (valid for 24 hours)
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({
      message: 'Login successful',
      token,
      user: { id: user.id, name: user.name, email: user.email, role: user.role }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

---

## Step 7: Express Middleware & Protected Routes

To protect routes (e.g. only logged-in users can create posts), we create an Express **middleware** that verifies the `Authorization: Bearer <token>` HTTP header.

Create `middleware.js`:

```javascript
// middleware.js
import jwt from 'jsonwebtoken';

export const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  // Header format: "Bearer <TOKEN>"
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access denied. No token provided.' });
  }

  try {
    // Verify token validity signature
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    // Attach logged-in user details to request object
    req.user = decoded; 
    next(); // Pass control to the next route handler
  } catch (error) {
    return res.status(403).json({ error: 'Invalid or expired token.' });
  }
};
```

---

## Step 8: Full Express API Endpoints (CRUD + Relations)

Create `index.js` incorporating both public auth endpoints and protected CRUD routes:

```javascript
// index.js
import express from 'express';
import dotenv from 'dotenv';
import prisma from './db.js';
import { register, login } from './auth.js';
import { authenticateToken } from './middleware.js';

dotenv.config();

const app = express();
app.use(express.json());

// =================================================================
// PUBLIC AUTHENTICATION ROUTES
// =================================================================
app.post('/api/auth/register', register);
app.post('/api/auth/login', login);

// =================================================================
// PROTECTED USER ROUTES
// =================================================================

// Get profile of currently logged-in user
app.get('/api/users/me', authenticateToken, async (req, res) => {
  try {
    const user = await prisma.user.findUnique({
      where: { id: req.user.userId },
      select: { id: true, name: true, email: true, role: true, createdAt: true, posts: true }
    });
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all users (Protected)
app.get('/api/users', authenticateToken, async (req, res) => {
  try {
    const users = await prisma.user.findMany({
      select: { id: true, name: true, email: true, role: true, posts: true }
    });
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// =================================================================
// PROTECTED POST ROUTES (CRUD + Relations)
// =================================================================

// Create post for authenticated user
app.post('/api/posts', authenticateToken, async (req, res) => {
  try {
    const { title, content } = req.body;

    const newPost = await prisma.post.create({
      data: {
        title,
        content,
        authorId: req.user.userId // Extracted safely from JWT token!
      }
    });

    res.status(201).json(newPost);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update a post (Only if current user is author)
app.put('/api/posts/:id', authenticateToken, async (req, res) => {
  try {
    const { id } = req.params;
    const { title, content, published } = req.body;

    // Check ownership
    const existingPost = await prisma.post.findUnique({ where: { id: parseInt(id) } });
    if (!existingPost) return res.status(404).json({ error: 'Post not found' });
    if (existingPost.authorId !== req.user.userId) {
      return res.status(403).json({ error: 'Unauthorized to edit this post' });
    }

    const updatedPost = await prisma.post.update({
      where: { id: parseInt(id) },
      data: { title, content, published }
    });

    res.json(updatedPost);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete a post (Only if current user is author)
app.delete('/api/posts/:id', authenticateToken, async (req, res) => {
  try {
    const { id } = req.params;

    const existingPost = await prisma.post.findUnique({ where: { id: parseInt(id) } });
    if (!existingPost) return res.status(404).json({ error: 'Post not found' });
    if (existingPost.authorId !== req.user.userId) {
      return res.status(403).json({ error: 'Unauthorized to delete this post' });
    }

    await prisma.post.delete({ where: { id: parseInt(id) } });
    res.json({ message: 'Post deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
});
```

---

## Step 9: Advanced Features (Transactions, Filtering, Pagination)

### 1. Database Transactions (`$transaction`)
```javascript
app.post('/api/users-with-post', async (req, res) => {
  try {
    const { name, email, password, postTitle } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);

    const result = await prisma.$transaction(async (tx) => {
      const user = await tx.user.create({
        data: { name, email, password: hashedPassword }
      });

      const post = await tx.post.create({
        data: { title: postTitle, authorId: user.id }
      });

      return { user: { id: user.id, email: user.email }, post };
    });

    res.status(201).json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

---

## Step 10: Testing Your API with Curl

Start server: `node index.js`

### 1. Register User
```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com", "password": "password123"}'
```

### 2. Login User & Get JWT Token
```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@example.com", "password": "password123"}'
```
*Response will return `"token": "eyJhbGciOi..."`.*

### 3. Access Protected Route with Token
```bash
curl http://localhost:3000/api/users/me \
  -H "Authorization: Bearer YOUR_JWT_TOKEN_HERE"
```

### 4. Create Protected Post
```bash
curl -X POST http://localhost:3000/api/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN_HERE" \
  -d '{"title": "My Authenticated Prisma Post", "content": "Prisma + JWT + Express!"}'
```

---

## Common Pitfalls & Troubleshooting Guide

| Issue | Cause | Fix |
| :--- | :--- | :--- |
| **`401 Access denied`** | Missing `Authorization` header or missing `Bearer` prefix. | Ensure header is formatted as `Authorization: Bearer <token>`. |
| **`Storing plain-text passwords`** | Inserting user password directly without hashing. | Always hash with `bcrypt.hash(password, 10)` before running `prisma.user.create()`. |
| **`Leaking passwords in response`** | Selecting all fields on user queries. | Use Prisma's `select: { id: true, email: true }` to omit password hashes. |
| **`Argument id: Invalid value provided`** | Passing string ID from `req.params.id` directly into Prisma query. | Wrap ID in `parseInt(req.params.id)`. |
