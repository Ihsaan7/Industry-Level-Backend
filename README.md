# Industry-Level Backend Architecture & Workflow Guide (MiniTube)

---

## 1. Project Structure & Initial Setup

- **Create folders (backend):** `src/`, `src/controllers/`, `src/models/`, `src/middlewares/`, `src/routes/`, `src/utils/`, plus `src/app.js` and `src/main.js`. Keep `constants.js` in `src/`.
- **Initialize Node project:** In `MiniTube/backend`, run:
  ```powershell
  cd MiniTube/backend
  npm init -y
  ```
- **Install packages:**
  ```powershell
  npm install express dotenv mongoose cookie-parser bcrypt jsonwebtoken multer cloudinary cors
  npm install -D nodemon
  ```
- **Add npm scripts (in `package.json`):**
  ```json
  {
    "type": "module",
    "scripts": {
      "dev": "nodemon src/main.js",
      "start": "node src/main.js"
    }
  }
  ```
- **Environment config (`.env`):** Create `.env` in the backend project root (same folder as `package.json`), not inside `src/`.
  ```ini
  PORT=8000
  MONGODB_URI=mongodb://127.0.0.1:27017/minitube
  JWT_SECRET=superlongrandomsecret
  CLOUDINARY_CLOUD_NAME=your_cloud
  CLOUDINARY_API_KEY=your_key
  CLOUDINARY_API_SECRET=your_secret
  ```
- **Load dotenv first (in `src/main.js`):** First line should be `import 'dotenv/config'` so env vars are ready for all imports.

Minimal scaffolds you write:

- `src/main.js` (entry): load dotenv, connect DB, start server.
- `src/app.js` (Express app): register middleware, routes, 404 + error handler, export `app`.

---

## 2. Server & Database Initialization

- **Express app (in `src/app.js`):**

  1. Create app and register core middleware.
  2. Add a simple health route.
  3. Mount API routes (later).
  4. Add 404 handler and centralized error handler.
  5. `export default app;`

  Minimal example:

  ```js
  // src/app.js
  import express from "express";
  import cookieParser from "cookie-parser";
  import cors from "cors";
  const app = express();

  app.use(cors({ origin: true, credentials: true }));
  app.use(express.json({ limit: "16kb" }));
  app.use(express.urlencoded({ extended: true, limit: "16kb" }));
  app.use(cookieParser());

  app.get("/health", (req, res) => res.status(200).json({ ok: true }));

  // app.use('/api/v1/users', userRouter);
  // app.use('/api/v1/videos', videoRouter);

  app.use((req, res, next) => res.status(404).json({ error: "Not Found" }));

  // Centralized error handler (shape it later)
  app.use((err, req, res, next) => {
    const status = err.statusCode || 500;
    res.status(status).json({ error: err.message || "Internal Server Error" });
  });

  export default app;
  ```

- **MongoDB connection (in `src/main.js`):**

  1. Load dotenv at the very top.
  2. Import `app`.
  3. `mongoose.connect(process.env.MONGODB_URI)`; add basic logging and exit on fatal errors.

  Minimal example:

  ```js
  // src/main.js
  import "dotenv/config"; // keep this first
  import mongoose from "mongoose";
  import app from "./app.js";

  const PORT = process.env.PORT || 8000;

  async function start() {
    try {
      await mongoose.connect(process.env.MONGODB_URI);
      console.log("MongoDB connected");
      const server = app.listen(PORT, () => {
        console.log(`Server listening on port ${PORT}`);
      });
      server.on("error", (err) => {
        console.error("HTTP server error:", err);
        process.exit(1);
      });
    } catch (err) {
      console.error("MongoDB connection failed:", err);
      process.exit(1);
    }
  }

  // Optional: catch unhandled rejections
  process.on("unhandledRejection", (err) => {
    console.error("Unhandled Rejection:", err);
  });

  start();
  ```

- **Start server:** Ensure DB is connected before `app.listen`. Read `PORT` from `.env` and log startup.

Run and verify locally:

```powershell
npm run dev
# then open http://localhost:8000/health
```

---

## 3. Middleware Setup

- **Global middleware (what/why):**

  - `express.json()`/`express.urlencoded()` — parse JSON/forms (limit size to avoid abuse).
  - `cookie-parser` — read signed/unsigned cookies (tokens, preferences).
  - `cors` — allow frontend origin and credentials during dev.

- **Custom middleware (create these files):**
  - `src/utils/asyncHandler.js` — wrap async controllers so errors go to the handler.
    ```js
    // src/utils/asyncHandler.js
    const asyncHandler = (fn) => (req, res, next) =>
      Promise.resolve(fn(req, res, next)).catch(next);
    export default asyncHandler;
    ```
  - `src/middlewares/multer.mware.js` — configure uploads.
    ```js
    // src/middlewares/multer.mware.js
    import multer from "multer";
    const storage = multer.diskStorage({}); // temp to disk; validate type in fileFilter
    const fileFilter = (req, file, cb) => {
      // accept images/videos only (adjust as needed)
      const ok = /image|video/.test(file.mimetype);
      cb(ok ? null : new Error("Invalid file type"), ok);
    };
    export const upload = multer({
      storage,
      fileFilter,
      limits: { fileSize: 20 * 1024 * 1024 },
    });
    ```
  - `src/middlewares/auth.mware.js` — verify JWT from cookie or header and attach `req.user`.
    ```js
    // src/middlewares/auth.mware.js
    import jwt from "jsonwebtoken";
    export function requireAuth(req, res, next) {
      try {
        const token =
          req.cookies?.accessToken ||
          req.headers?.authorization?.replace("Bearer ", "");
        if (!token) return res.status(401).json({ error: "Unauthorized" });
        const payload = jwt.verify(token, process.env.JWT_SECRET);
        req.user = { id: payload?.sub || payload?.id, ...payload };
        next();
      } catch (err) {
        return res.status(401).json({ error: "Invalid or expired token" });
      }
    }
    ```

---

## 4. Error Handling

- **Centralized error handler:**

  - Create `ApiError` and `ApiResponse` classes in `utils/` and use one error handler middleware at the end.
  - Keep response shape consistent; never leak stack traces in production.

  Minimal utils:

  ```js
  // src/utils/ApiError.js
  export default class ApiError extends Error {
    constructor(statusCode = 500, message = 'Something went wrong') {
      super(message);
      this.statusCode = statusCode;
    }
  }
  // src/utils/ApiResponse.js
  export default class ApiResponse {
    constructor(statusCode = 200, data = null, message = 'OK') {
      this.success = statusCode < 400;
      this.statusCode = statusCode;
      this.data = data;
      this.message = message;
    }
  }
  ```

  Error handler middleware:

  ```js
  // src/middlewares/errorHandler.js
  export default function errorHandler(err, req, res, next) {
    const status = err.statusCode || 500;
    const msg = err.message || "Internal Server Error";
    res.status(status).json({ success: false, error: msg });
  }
  ```

  Wire it in `src/app.js` after all routes:

  ```js
  import errorHandler from "./middlewares/errorHandler.js";
  // ...existing code...
  app.use(errorHandler);
  ```

- **Async errors:**
  - Wrap controllers with `asyncHandler(fn)` so thrown errors reach `errorHandler`.
  - Validate inputs early and throw `new ApiError(400, 'message')` when invalid.

---

## 5. Controller Logic & Route Design

- **RESTful routes:**
  - Organize by resource (users, videos, etc).
  - Use separate controller files for each resource.
- **Controller patterns:**
  - Validate request data.
  - Use try/catch or `asyncHandler` for async logic.
  - Interact with models (DB), handle business logic, and return responses.
- **Example:**
  - Registration: Validate, hash password, save user, return token.
  - Video upload: Validate, handle file, upload to Cloudinary, save DB record.

---

## 6. Authentication & Security

- **JWT-based auth:**
  - Issue tokens on login/registration.
  - Store refresh tokens securely (DB/cookie).
  - Use middleware to protect routes.
- **Password security:**
  - Hash passwords with `bcrypt` before saving.
  - Never store plain passwords.
- **Input validation:**
  - Always validate user input (length, type, format).
  - Sanitize data to prevent injection attacks.

---

## 7. File Uploads & Cloudinary Integration

- **Multer setup:**
  - Use for handling multipart/form-data.
  - Validate file type and size.
- **Cloudinary utility:**
  - Create a helper for uploading files.
  - Handle upload errors and clean up temp files.

---

## 8. Database Operations & Aggregation

- **Mongoose models:**
  - Define schemas for each resource (User, Video, etc).
  - Use pre-save hooks for password hashing.
- **Aggregation:**
  - Use MongoDB aggregation for analytics, search, etc.
  - Validate pipeline stages and handle errors.

---

## 9. Best Practices & Advanced Tips

- **Async/await everywhere:** Avoid callback hell.
- **Consistent error responses:** Use `ApiError` and `ApiResponse`.
- **Logging:** Log errors and important events.
- **Modular code:** Keep controllers, models, and middleware separate.
- **Environment variables:** Never hardcode secrets.
- **Testing:** Write unit/integration tests for controllers and middleware.
- **Documentation:** Keep README and guides up to date.

---

## 10. Typical Workflow (Step-by-Step)

1. **Plan feature:** Define requirements and endpoints.
2. **Design model:** Create/update Mongoose schema.
3. **Write controller:** Implement business logic, validation, error handling.
4. **Set up route:** Mount controller in route file.
5. **Add middleware:** Protect routes, handle uploads, validate input.
6. **Test locally:** Use Postman or similar tools.
7. **Handle errors:** Ensure all errors are caught and returned properly.
8. **Document:** Update README and guides.
9. **Deploy:** Use environment variables, check logs, monitor errors.

---

## 11. Common Pitfalls & How to Avoid Them

- **Import order:** Always load `.env` before using variables.
- **Async errors:** Use `asyncHandler` to avoid unhandled promise rejections.
- **Typos in aggregation:** Double-check pipeline stage names.
- **Cookie issues:** Ensure `cookie-parser` is used before accessing cookies.
- **File upload bugs:** Validate file type/size, handle errors gracefully.

---

## 12. Reference: Useful Code Patterns

- **Async handler:**
  ```js
  const asyncHandler = (fn) => (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);
  ```
- **Central error handler:**
  ```js
  app.use((err, req, res, next) => {
    // ...handle error, send response
  });
  ```
- **JWT middleware:**
  ```js
  // ...verify token, attach user to req
  ```

---

## 13. Final Advice

- **Read code, not just docs.**
- **Debug step-by-step.**
- **Ask why each line exists.**
- **Practice by building features.**
- **Keep learning!**

---

_This guide is tailored for the MiniTube backend. Adapt and expand as your project grows!_
