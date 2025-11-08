# Industry Backend Development: Build-Test-Repeat Workflow (MiniTube)

> **Philosophy:** Build one small piece → test it works → move to next piece. Never write everything at once.

---

## Step 1: Create Project & Install Basic Dependencies

### 1.1 Make folders
```powershell
cd D:\your\path
mkdir MiniTube
cd MiniTube
mkdir backend
cd backend
mkdir src
```

### 1.2 Initialize Node project
```powershell
npm init -y
```

### 1.3 Set module type in `package.json`
Open `package.json` and add:
```json
{
  "type": "module",
  ...
}
```
**Why?** So you can use `import/export` instead of `require`.

### 1.4 Install Express and dotenv first
```powershell
npm install express dotenv
npm install -D nodemon
```
**Why?** Start minimal—Express for server, dotenv for env vars, nodemon for auto-restart.

### 1.5 Create `.env` in backend root (not in `src`)
```ini
PORT=8000
```
**Why?** Keep secrets outside code. Git will ignore this file later.

### 1.6 Add npm script in `package.json`
```json
"scripts": {
  "dev": "nodemon src/main.js"
}
```

---

## Step 2: Write Minimal Server & Test

### 2.1 Create `src/main.js`
```js
// src/main.js
import 'dotenv/config';  // ALWAYS first line
import express from 'express';

const app = express();
const PORT = process.env.PORT || 8000;

app.get('/health', (req, res) => {
  res.json({ message: 'Server is alive!' });
});

app.listen(PORT, () => {
  console.log(`✅ Server running on http://localhost:${PORT}`);
});
```

### 2.2 Test it
```powershell
npm run dev
```
Open browser: `http://localhost:8000/health` → you should see `{"message":"Server is alive!"}`

**✅ Checkpoint:** Server runs, health route works.

---

## Step 3: Connect MongoDB & Test

### 3.1 Install mongoose
```powershell
npm install mongoose
```

### 3.2 Add MongoDB URI to `.env`
```ini
PORT=8000
MONGODB_URI=mongodb://127.0.0.1:27017/minitube
```
**Note:** Use MongoDB Atlas cloud URI if you prefer, or local MongoDB.

### 3.3 Update `src/main.js` to connect DB
```js
// src/main.js
import 'dotenv/config';
import express from 'express';
import mongoose from 'mongoose';

const app = express();
const PORT = process.env.PORT || 8000;

app.get('/health', (req, res) => {
  res.json({ message: 'Server is alive!', db: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected' });
});

async function start() {
  try {
    await mongoose.connect(process.env.MONGODB_URI);
    console.log('✅ MongoDB connected');
    
    app.listen(PORT, () => {
      console.log(`✅ Server running on http://localhost:${PORT}`);
    });
  } catch (err) {
    console.error('❌ MongoDB connection failed:', err.message);
    process.exit(1);
  }
}

start();
```

### 3.4 Test it
```powershell
npm run dev
```
Check terminal: you should see "MongoDB connected". Visit `/health` and see `"db":"connected"`.

**✅ Checkpoint:** Database connected, server starts only after DB is ready.

---

## Step 4: Separate App Logic from Server Start

**Why?** Keep `app.js` clean (routes, middleware) and `main.js` minimal (DB + start server).

### 4.1 Create `src/app.js`
```js
// src/app.js
import express from 'express';

const app = express();

// Basic middleware
app.use(express.json({ limit: '16kb' }));
app.use(express.urlencoded({ extended: true, limit: '16kb' }));

// Health route
app.get('/health', (req, res) => {
  res.json({ message: 'App is healthy' });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

export default app;
```

### 4.2 Update `src/main.js`
```js
// src/main.js
import 'dotenv/config';
import mongoose from 'mongoose';
import app from './app.js';

const PORT = process.env.PORT || 8000;

async function start() {
  try {
    await mongoose.connect(process.env.MONGODB_URI);
    console.log('✅ MongoDB connected');
    
    app.listen(PORT, () => {
      console.log(`✅ Server running on http://localhost:${PORT}`);
    });
  } catch (err) {
    console.error('❌ MongoDB connection failed:', err.message);
    process.exit(1);
  }
}

start();
```

### 4.3 Test it
```powershell
npm run dev
```
**✅ Checkpoint:** Server still works. Code is now modular.

---

## Step 5: Add Error Handling Utils

**Why?** Industry standard = consistent error responses. Build utilities before controllers.

### 5.1 Create `src/utils/ApiError.js`
```js
// src/utils/ApiError.js
class ApiError extends Error {
  constructor(statusCode, message = 'Something went wrong') {
    super(message);
    this.statusCode = statusCode;
    this.success = false;
  }
}

export default ApiError;
```

### 5.2 Create `src/utils/ApiResponse.js`
```js
// src/utils/ApiResponse.js
class ApiResponse {
  constructor(statusCode, data, message = 'Success') {
    this.statusCode = statusCode;
    this.data = data;
    this.message = message;
    this.success = statusCode < 400;
  }
}

export default ApiResponse;
```

### 5.3 Create `src/utils/asyncHandler.js`
```js
// src/utils/asyncHandler.js
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

export default asyncHandler;
```
**Why?** Wraps async controllers so errors go to error handler automatically.

### 5.4 Add error handler middleware in `src/app.js`
```js
// src/app.js (add at the end, before export)
import express from 'express';

const app = express();

app.use(express.json({ limit: '16kb' }));
app.use(express.urlencoded({ extended: true, limit: '16kb' }));

app.get('/health', (req, res) => {
  res.json({ message: 'App is healthy' });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// Centralized error handler
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal Server Error';
  res.status(statusCode).json({
    success: false,
    message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
});

export default app;
```

### 5.5 Test error handler
Add a test route in `src/app.js` (temporary):
```js
import ApiError from './utils/ApiError.js';
import asyncHandler from './utils/asyncHandler.js';

app.get('/test-error', asyncHandler(async (req, res) => {
  throw new ApiError(400, 'This is a test error');
}));
```
Visit `http://localhost:8000/test-error` → you should see `{"success":false,"message":"This is a test error"}`.

**✅ Checkpoint:** Error handling works. Remove test route after verifying.

---

## Step 6: Create Your First Model (User)

**Why?** Start with User model since auth depends on it.

### 6.1 Install bcrypt and jsonwebtoken
```powershell
npm install bcrypt jsonwebtoken
```

### 6.2 Add JWT secrets to `.env`
```ini
PORT=8000
MONGODB_URI=mongodb://127.0.0.1:27017/minitube
JWT_SECRET=your_super_secret_key_min_32_chars
JWT_REFRESH_SECRET=another_super_secret_for_refresh
```

### 6.3 Create `src/models/user.model.js`
```js
// src/models/user.model.js
import mongoose from 'mongoose';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true, lowercase: true, trim: true, index: true },
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  fullName: { type: String, required: true, trim: true },
  password: { type: String, required: true },
  avatar: { type: String }, // Cloudinary URL
  coverImage: { type: String },
  refreshToken: { type: String }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Method: compare password
userSchema.methods.isPasswordCorrect = async function(password) {
  return await bcrypt.compare(password, this.password);
};

// Method: generate access token
userSchema.methods.generateAccessToken = function() {
  return jwt.sign(
    { _id: this._id, email: this.email, username: this.username },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
};

// Method: generate refresh token
userSchema.methods.generateRefreshToken = function() {
  return jwt.sign(
    { _id: this._id },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );
};

export const User = mongoose.model('User', userSchema);
```

**✅ Checkpoint:** Model is ready. No test yet—we'll test when controller is written.

---

## Step 7: Create User Controller & Router (Register Endpoint)

### 7.1 Create `src/controllers/user.controller.js`
```js
// src/controllers/user.controller.js
import asyncHandler from '../utils/asyncHandler.js';
import ApiError from '../utils/ApiError.js';
import ApiResponse from '../utils/ApiResponse.js';
import { User } from '../models/user.model.js';

// Register user
export const registerUser = asyncHandler(async (req, res) => {
  const { username, email, fullName, password } = req.body;

  // Validate
  if ([username, email, fullName, password].some(field => !field?.trim())) {
    throw new ApiError(400, 'All fields are required');
  }

  // Check if user exists
  const existingUser = await User.findOne({ $or: [{ username }, { email }] });
  if (existingUser) {
    throw new ApiError(409, 'User already exists');
  }

  // Create user
  const user = await User.create({ username, email, fullName, password });

  // Fetch without password
  const createdUser = await User.findById(user._id).select('-password -refreshToken');

  if (!createdUser) {
    throw new ApiError(500, 'Something went wrong while registering user');
  }

  return res.status(201).json(new ApiResponse(201, createdUser, 'User registered successfully'));
});
```

### 7.2 Create `src/routes/user.routes.js`
```js
// src/routes/user.routes.js
import { Router } from 'express';
import { registerUser } from '../controllers/user.controller.js';

const router = Router();

router.post('/register', registerUser);

export default router;
```

### 7.3 Mount router in `src/app.js`
```js
// src/app.js (add after middleware, before 404)
import userRouter from './routes/user.routes.js';

app.use('/api/v1/users', userRouter);
```

### 7.4 Test with Postman or curl
```powershell
# Using curl
curl -X POST http://localhost:8000/api/v1/users/register `
  -H "Content-Type: application/json" `
  -d '{"username":"testuser","email":"test@test.com","fullName":"Test User","password":"password123"}'
```
You should get `201` status and user data (without password).

**✅ Checkpoint:** User registration works. Model, controller, router all connected.

---

## Step 8: Add Login & JWT Auth Middleware

### 8.1 Install cookie-parser
```powershell
npm install cookie-parser
```

### 8.2 Add cookie-parser to `src/app.js`
```js
// src/app.js (after json/urlencoded)
import cookieParser from 'cookie-parser';

app.use(cookieParser());
```

### 8.3 Add login controller in `src/controllers/user.controller.js`
```js
// src/controllers/user.controller.js (add this function)

export const loginUser = asyncHandler(async (req, res) => {
  const { username, email, password } = req.body;

  if (!username && !email) {
    throw new ApiError(400, 'Username or email is required');
  }

  const user = await User.findOne({ $or: [{ username }, { email }] });
  if (!user) {
    throw new ApiError(404, 'User not found');
  }

  const isPasswordValid = await user.isPasswordCorrect(password);
  if (!isPasswordValid) {
    throw new ApiError(401, 'Invalid credentials');
  }

  const accessToken = user.generateAccessToken();
  const refreshToken = user.generateRefreshToken();

  user.refreshToken = refreshToken;
  await user.save({ validateBeforeSave: false });

  const loggedInUser = await User.findById(user._id).select('-password -refreshToken');

  const options = { httpOnly: true, secure: true };

  return res
    .status(200)
    .cookie('accessToken', accessToken, options)
    .cookie('refreshToken', refreshToken, options)
    .json(new ApiResponse(200, { user: loggedInUser, accessToken, refreshToken }, 'Login successful'));
});
```

### 8.4 Add login route in `src/routes/user.routes.js`
```js
// src/routes/user.routes.js
import { registerUser, loginUser } from '../controllers/user.controller.js';

router.post('/register', registerUser);
router.post('/login', loginUser);
```

### 8.5 Test login
```powershell
curl -X POST http://localhost:8000/api/v1/users/login `
  -H "Content-Type: application/json" `
  -d '{"username":"testuser","password":"password123"}'
```
You should get tokens in response and cookies set.

**✅ Checkpoint:** Login works, tokens issued.

### 8.6 Create auth middleware `src/middlewares/auth.middleware.js`
```js
// src/middlewares/auth.middleware.js
import jwt from 'jsonwebtoken';
import asyncHandler from '../utils/asyncHandler.js';
import ApiError from '../utils/ApiError.js';
import { User } from '../models/user.model.js';

export const verifyJWT = asyncHandler(async (req, res, next) => {
  const token = req.cookies?.accessToken || req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    throw new ApiError(401, 'Unauthorized request');
  }

  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  const user = await User.findById(decoded._id).select('-password -refreshToken');

  if (!user) {
    throw new ApiError(401, 'Invalid access token');
  }

  req.user = user;
  next();
});
```

### 8.7 Test protected route
Add a protected route in `src/routes/user.routes.js`:
```js
// src/routes/user.routes.js
import { registerUser, loginUser } from '../controllers/user.controller.js';
import { verifyJWT } from '../middlewares/auth.middleware.js';

router.post('/register', registerUser);
router.post('/login', loginUser);

// Protected route example
router.get('/profile', verifyJWT, (req, res) => {
  res.json(new ApiResponse(200, req.user, 'Profile fetched'));
});
```

Test:
```powershell
curl http://localhost:8000/api/v1/users/profile `
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN_HERE"
```

**✅ Checkpoint:** Auth middleware works, protected routes accessible only with token.

---

## Step 9: Repeat for Video Model, Controller, Routes

### 9.1 Create `src/models/video.model.js`
```js
// src/models/video.model.js
import mongoose from 'mongoose';

const videoSchema = new mongoose.Schema({
  videoFile: { type: String, required: true }, // Cloudinary URL
  thumbnail: { type: String, required: true },
  title: { type: String, required: true },
  description: { type: String, required: true },
  duration: { type: Number, required: true },
  views: { type: Number, default: 0 },
  isPublished: { type: Boolean, default: true },
  owner: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
}, { timestamps: true });

export const Video = mongoose.model('Video', videoSchema);
```

### 9.2 Create `src/controllers/video.controller.js`
```js
// src/controllers/video.controller.js
import asyncHandler from '../utils/asyncHandler.js';
import ApiError from '../utils/ApiError.js';
import ApiResponse from '../utils/ApiResponse.js';
import { Video } from '../models/video.model.js';

export const getAllVideos = asyncHandler(async (req, res) => {
  const videos = await Video.find({ isPublished: true }).populate('owner', 'username avatar');
  res.json(new ApiResponse(200, videos, 'Videos fetched'));
});

export const getVideoById = asyncHandler(async (req, res) => {
  const video = await Video.findById(req.params.id).populate('owner', 'username avatar');
  if (!video) throw new ApiError(404, 'Video not found');
  res.json(new ApiResponse(200, video, 'Video fetched'));
});

// Add more: publishVideo, updateVideo, deleteVideo...
```

### 9.3 Create `src/routes/video.routes.js`
```js
// src/routes/video.routes.js
import { Router } from 'express';
import { getAllVideos, getVideoById } from '../controllers/video.controller.js';
import { verifyJWT } from '../middlewares/auth.middleware.js';

const router = Router();

router.get('/', getAllVideos);
router.get('/:id', getVideoById);
// router.post('/', verifyJWT, publishVideo);

export default router;
```

### 9.4 Mount in `src/app.js`
```js
// src/app.js
import videoRouter from './routes/video.routes.js';

app.use('/api/v1/videos', videoRouter);
```

### 9.5 Test
```powershell
curl http://localhost:8000/api/v1/videos
```

**✅ Checkpoint:** Video model, controller, routes work. Pattern established.

---

## Step 10: Add File Upload (Multer + Cloudinary)

### 10.1 Install multer and cloudinary
```powershell
npm install multer cloudinary
```

### 10.2 Add Cloudinary config to `.env`
```ini
CLOUDINARY_CLOUD_NAME=your_cloud
CLOUDINARY_API_KEY=your_key
CLOUDINARY_API_SECRET=your_secret
```

### 10.3 Create `src/utils/cloudinary.js`
```js
// src/utils/cloudinary.js
import { v2 as cloudinary } from 'cloudinary';
import fs from 'fs';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

export const uploadOnCloudinary = async (localFilePath) => {
  try {
    if (!localFilePath) return null;
    const response = await cloudinary.uploader.upload(localFilePath, { resource_type: 'auto' });
    fs.unlinkSync(localFilePath); // delete local file after upload
    return response;
  } catch (error) {
    fs.unlinkSync(localFilePath); // delete local file on error
    return null;
  }
};
```

### 10.4 Create `src/middlewares/multer.middleware.js`
```js
// src/middlewares/multer.middleware.js
import multer from 'multer';

const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, './public/temp');
  },
  filename: function (req, file, cb) {
    cb(null, Date.now() + '-' + file.originalname);
  }
});

export const upload = multer({ storage });
```

### 10.5 Create `public/temp` folder
```powershell
mkdir public
mkdir public\temp
```

### 10.6 Update user registration to handle avatar upload
Update `src/routes/user.routes.js`:
```js
import { upload } from '../middlewares/multer.middleware.js';

router.post('/register', upload.fields([
  { name: 'avatar', maxCount: 1 },
  { name: 'coverImage', maxCount: 1 }
]), registerUser);
```

Update `src/controllers/user.controller.js`:
```js
import { uploadOnCloudinary } from '../utils/cloudinary.js';

export const registerUser = asyncHandler(async (req, res) => {
  const { username, email, fullName, password } = req.body;

  if ([username, email, fullName, password].some(field => !field?.trim())) {
    throw new ApiError(400, 'All fields are required');
  }

  const existingUser = await User.findOne({ $or: [{ username }, { email }] });
  if (existingUser) {
    throw new ApiError(409, 'User already exists');
  }

  // Handle file uploads
  const avatarLocalPath = req.files?.avatar?.[0]?.path;
  const coverImageLocalPath = req.files?.coverImage?.[0]?.path;

  if (!avatarLocalPath) {
    throw new ApiError(400, 'Avatar is required');
  }

  const avatar = await uploadOnCloudinary(avatarLocalPath);
  const coverImage = coverImageLocalPath ? await uploadOnCloudinary(coverImageLocalPath) : null;

  if (!avatar) {
    throw new ApiError(500, 'Avatar upload failed');
  }

  const user = await User.create({
    username,
    email,
    fullName,
    password,
    avatar: avatar.url,
    coverImage: coverImage?.url || ''
  });

  const createdUser = await User.findById(user._id).select('-password -refreshToken');

  return res.status(201).json(new ApiResponse(201, createdUser, 'User registered successfully'));
});
```

### 10.7 Test file upload with Postman
Use form-data, add fields: username, email, fullName, password, avatar (file), coverImage (file).

**✅ Checkpoint:** File upload works. Avatar stored in Cloudinary.

---

## Summary: The Human Workflow

1. **Make folders** → npm init → install basics
2. **Write minimal server** → test `/health`
3. **Connect DB** → test connection
4. **Split app.js and main.js** → test still works
5. **Add error utils** → test error handler
6. **Create User model** → no test yet
7. **Create User controller & router (register)** → test registration
8. **Add login + auth middleware** → test login + protected route
9. **Repeat for Video** (model → controller → router) → test endpoints
10. **Add file upload** (multer + cloudinary) → test with files

**Key mindset:**
- Build ONE thing
- Test it works
- Move to next thing
- Never write 10 files at once

---

_Now go build your backend step-by-step. Test every piece. Ask questions when stuck!_
