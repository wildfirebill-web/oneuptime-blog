# How to Use Passport.js Local Strategy with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Passport.js, Authentication, Node.js, Security

Description: Learn how to integrate Passport.js Local Strategy with MongoDB to implement username and password authentication in a Node.js Express application.

---

## Overview

Passport.js is the most popular authentication middleware for Node.js. Its Local Strategy handles username/password authentication and pairs naturally with MongoDB for storing user credentials. This guide shows how to wire them together securely.

## Project Setup

Install the required packages:

```bash
npm install express passport passport-local mongoose bcrypt express-session
```

## Defining the User Model

Store passwords as bcrypt hashes, never in plaintext:

```javascript
// models/User.js
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true, lowercase: true },
  email:    { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true }
});

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

userSchema.methods.comparePassword = async function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = mongoose.model("User", userSchema);
```

## Configuring the Local Strategy

```javascript
// config/passport.js
const passport = require("passport");
const { Strategy: LocalStrategy } = require("passport-local");
const User = require("../models/User");

passport.use(
  new LocalStrategy(
    { usernameField: "email", passwordField: "password" },
    async (email, password, done) => {
      try {
        const user = await User.findOne({ email });
        if (!user) {
          return done(null, false, { message: "No account with that email." });
        }
        const match = await user.comparePassword(password);
        if (!match) {
          return done(null, false, { message: "Incorrect password." });
        }
        return done(null, user);
      } catch (err) {
        return done(err);
      }
    }
  )
);

passport.serializeUser((user, done) => done(null, user._id));

passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id).select("-password");
    done(null, user);
  } catch (err) {
    done(err);
  }
});
```

## Setting Up Express with Sessions

```javascript
// app.js
const express = require("express");
const session = require("express-session");
const passport = require("passport");
const mongoose = require("mongoose");
require("./config/passport");

const app = express();
mongoose.connect("mongodb://localhost:27017/auth_demo");

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(session({
  secret: process.env.SESSION_SECRET || "change-this-secret",
  resave: false,
  saveUninitialized: false,
  cookie: { secure: process.env.NODE_ENV === "production", httpOnly: true }
}));
app.use(passport.initialize());
app.use(passport.session());
```

## Registration and Login Routes

```javascript
// routes/auth.js
const express = require("express");
const passport = require("passport");
const User = require("../models/User");
const router = express.Router();

router.post("/register", async (req, res) => {
  const { email, password, username } = req.body;
  try {
    const existing = await User.findOne({ email });
    if (existing) {
      return res.status(409).json({ error: "Email already registered." });
    }
    const user = new User({ email, password, username });
    await user.save();
    res.status(201).json({ message: "Account created." });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

router.post("/login", passport.authenticate("local", {
  successRedirect: "/dashboard",
  failureRedirect: "/login",
  failureMessage: true
}));

router.post("/logout", (req, res) => {
  req.logout(err => {
    if (err) return res.status(500).json({ error: err.message });
    res.redirect("/login");
  });
});

module.exports = router;
```

## Protecting Routes

```javascript
function ensureAuthenticated(req, res, next) {
  if (req.isAuthenticated()) return next();
  res.status(401).json({ error: "Please log in." });
}

app.get("/dashboard", ensureAuthenticated, (req, res) => {
  res.json({ message: `Welcome, ${req.user.username}` });
});
```

## Creating the Unique Index for Email

Ensure the unique index exists on the email field:

```javascript
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ username: 1 }, { unique: true });
```

Or through Mongoose, which handles this from the schema definition on startup.

## Summary

Integrating Passport.js Local Strategy with MongoDB involves defining a Mongoose user model with bcrypt password hashing, configuring the strategy to look up users by email, and using `serializeUser`/`deserializeUser` for session persistence. The unique index on the email field in MongoDB ensures no duplicate accounts are created, and `express-session` maintains the login state between requests.
