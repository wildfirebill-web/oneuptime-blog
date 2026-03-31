# How to Use MongoDB with Passport.js for Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Passport, Authentication, Express, Session

Description: Learn how to integrate MongoDB with Passport.js to build local username/password authentication with session persistence in a Node.js Express app.

---

Passport.js is the most widely used authentication middleware for Node.js. When paired with MongoDB, it gives you a flexible, extensible auth system without locking you into a hosted service. This guide walks through setting up local strategy authentication backed by a MongoDB user store.

## Installing Dependencies

```bash
npm install passport passport-local mongoose express-session connect-mongo bcryptjs
```

## Defining the User Model

Store users in MongoDB with a hashed password field. Never store plain text passwords.

```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true, lowercase: true },
  passwordHash: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
});

userSchema.methods.validatePassword = async function (password) {
  return bcrypt.compare(password, this.passwordHash);
};

userSchema.statics.hashPassword = async function (password) {
  return bcrypt.hash(password, 12);
};

module.exports = mongoose.model('User', userSchema);
```

## Configuring Passport Local Strategy

```javascript
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const User = require('./models/User');

passport.use(
  new LocalStrategy({ usernameField: 'email' }, async (email, password, done) => {
    try {
      const user = await User.findOne({ email });
      if (!user) return done(null, false, { message: 'User not found' });

      const valid = await user.validatePassword(password);
      if (!valid) return done(null, false, { message: 'Invalid password' });

      return done(null, user);
    } catch (err) {
      return done(err);
    }
  })
);

passport.serializeUser((user, done) => done(null, user._id.toString()));

passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id).select('-passwordHash');
    done(null, user);
  } catch (err) {
    done(err);
  }
});
```

## Setting Up Session Store with MongoDB

Use `connect-mongo` to persist sessions in MongoDB so they survive server restarts.

```javascript
const session = require('express-session');
const MongoStore = require('connect-mongo');

app.use(
  session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    store: MongoStore.create({
      mongoUrl: process.env.MONGODB_URI,
      collectionName: 'sessions',
      ttl: 14 * 24 * 60 * 60, // 14 days in seconds
    }),
    cookie: {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      maxAge: 14 * 24 * 60 * 60 * 1000,
    },
  })
);

app.use(passport.initialize());
app.use(passport.session());
```

## Registration and Login Routes

```javascript
const bcrypt = require('bcryptjs');

// Register
app.post('/auth/register', async (req, res) => {
  const { email, password } = req.body;
  const passwordHash = await User.hashPassword(password);
  const user = await User.create({ email, passwordHash });
  req.login(user, (err) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ message: 'Registered and logged in', userId: user._id });
  });
});

// Login
app.post('/auth/login', passport.authenticate('local'), (req, res) => {
  res.json({ message: 'Logged in', userId: req.user._id });
});

// Logout
app.post('/auth/logout', (req, res) => {
  req.logout((err) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ message: 'Logged out' });
  });
});

// Protected route middleware
function ensureAuthenticated(req, res, next) {
  if (req.isAuthenticated()) return next();
  res.status(401).json({ error: 'Not authenticated' });
}

app.get('/api/profile', ensureAuthenticated, (req, res) => {
  res.json({ user: req.user });
});
```

## Querying the Sessions Collection

MongoDB stores sessions as documents you can query directly:

```javascript
// Find all active sessions for a user
db.sessions.find({ "session.passport.user": "<userId>" })

// Remove all sessions (force logout all devices)
db.sessions.deleteMany({ "session.passport.user": "<userId>" })
```

## Summary

Passport.js combined with MongoDB gives you a straightforward local authentication system. The `connect-mongo` session store keeps sessions durable across restarts. Use `bcryptjs` for password hashing, set `httpOnly` and `secure` cookie flags in production, and store only the user ID in the session to keep deserialization fast.
