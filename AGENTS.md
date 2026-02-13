# AGENTS.md - Developer Guidelines

This document provides guidelines for AI agents working in this codebase.

## Project Structure

```
ESTATE-BACKEND/
├── api/          # Express.js backend with Prisma ORM
├── client/       # React + Vite frontend
└── socket/       # Socket.io server for real-time communication
```

---

## Build/Lint/Test Commands

### API (`api/`)
```bash
cd api
npm start          # Start the Express server (port 8800)
npm run build      # Install deps + generate Prisma client
```

### Client (`client/`)
```bash
cd client
npm run dev        # Start Vite dev server
npm run build      # Build for production
npm run lint       # Run ESLint
npm run preview    # Preview production build
```

### Socket (`socket/`)
```bash
cd socket
node app.js        # Start socket server
# No tests configured
```

### Running a Single Test
No test framework is currently configured. To add tests, consider installing Jest or Vitest.

---

## Code Style Guidelines

### General Conventions

- **ES Modules**: Use `import`/`export` syntax (ESM), not CommonJS
- **File Extensions**: Use `.js` for Node.js, `.jsx` for React components
- **No TypeScript**: This is a plain JavaScript project

### Imports

**API (Express)**
```javascript
// Local imports - include .js extension
import authRoute from "./routes/auth.route.js";
import prisma from "../lib/prisma.js";

// Third-party
import express from "express";
import bcrypt from "bcrypt";
```

**Client (React)**
```javascript
// Relative paths for local imports
import HomePage from "./routes/homePage/homePage";
import { Layout } from "./routes/layout/layout";

// No file extension needed in React
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Files (routes) | `*.route.js` | `auth.route.js` |
| Files (controllers) | `*.controller.js` | `auth.controller.js` |
| Files (middleware) | `*.js` | `verifyToken.js` |
| React components | PascalCase | `HomePage.jsx`, `Navbar.jsx` |
| Variables/functions | camelCase | `newUser`, `getUserById` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRIES`, `PORT` |

### Error Handling

**Express Controllers**
```javascript
export const someAction = async (req, res) => {
  try {
    const result = await prisma.user.create({ /* ... */ });
    res.status(201).json(result);
  } catch (err) {
    console.log(err);  // Log error for debugging
    res.status(500).json({ message: "Failed to create user!" });
  }
};
```

- Always use try/catch for async operations
- Return appropriate HTTP status codes (200, 201, 400, 401, 404, 500)
- Provide meaningful error messages

### React Patterns

**Component Structure**
```javascript
import { useState, useEffect } from "react";

function ComponentName() {
  const [state, setState] = useState(null);

  useEffect(() => {
    // side effects
  }, []);

  return <div>JSX here</div>;
}

export default ComponentName;
```

**State Management**
- Use Zustand for global state (see `client/src/lib/notificationStore.js`)
- Use React Context for auth/socket (see `client/src/context/`)

**Routing**
- Use React Router v6 with `createBrowserRouter` and loaders for data fetching

### Database (Prisma)

- Schema is in `api/prisma/schema.prisma`
- Run `npx prisma generate` after schema changes
- Use Prisma client via `import prisma from "../lib/prisma.js"`

### API Routes Pattern

```javascript
// routes/xxx.route.js
import express from "express";
import { controllerFunc } from "../controllers/xxx.controller.js";

const router = express.Router();

router.post("/endpoint", controllerFunc);
// or router.get("/", controllerFunc);

export default router;
```

```javascript
// app.js
app.use("/api/resource", router);
```

### Security

- Never commit secrets to git (use `.env` files)
- Use `httpOnly`, `secure`, and `sameSite` for cookies
- Validate user input on both client and server

---

## Common Tasks

### Adding a New API Endpoint
1. Create controller in `api/controllers/`
2. Create route in `api/routes/`
3. Mount in `api/app.js`

### Adding a New Page
1. Create component in `client/src/routes/`
2. Add route to router in `client/src/App.jsx`

### Database Changes
1. Edit `api/prisma/schema.prisma`
2. Run `npx prisma db push` (dev) or create migration (`npx prisma migrate dev`)
3. Run `npx prisma generate`

---

## Notes

- Client CORS is configured for `https://primenest-client.vercel.app`
- API runs on port 8800 by default
- JWT secret stored in environment variable `JWT_SECRET_KEY`
