# How to Implement Magic Link Authentication with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Magic Link, Email, Token

Description: Learn how to implement passwordless magic link authentication with MongoDB, storing secure tokens with TTL indexes for automatic expiration.

---

## Overview

Magic link authentication allows users to sign in without a password by clicking a time-limited link sent to their email. MongoDB's TTL index feature makes it ideal for storing these tokens - expired tokens are automatically deleted by the database engine.

## How Magic Links Work

1. User enters their email address.
2. The server generates a unique, cryptographically secure token and stores it in MongoDB.
3. The server sends an email with a link containing the token.
4. User clicks the link; the server validates the token, creates a session, and deletes the token.

## Token Schema

```javascript
// models/MagicToken.js
const mongoose = require('mongoose');

const magicTokenSchema = new mongoose.Schema({
  email:     { type: String, required: true, index: true },
  token:     { type: String, required: true, unique: true },
  createdAt: { type: Date, default: Date.now, expires: '15m' }, // TTL index
});

module.exports = mongoose.model('MagicToken', magicTokenSchema);
```

The `expires: '15m'` option on `createdAt` tells MongoDB to automatically delete documents 15 minutes after creation using a TTL index.

## Generating and Storing the Token

```javascript
const crypto    = require('crypto');
const MagicToken = require('./models/MagicToken');
const User      = require('./models/User');

async function sendMagicLink(email) {
  // Create user if they don't exist yet
  let user = await User.findOne({ email });
  if (!user) {
    user = await User.create({ email });
  }

  // Generate a secure random token
  const token = crypto.randomBytes(32).toString('hex');

  // Delete any existing token for this email first
  await MagicToken.deleteOne({ email });

  // Store the new token
  await MagicToken.create({ email, token });

  // Build the magic link
  const magicLink = `${process.env.BASE_URL}/auth/verify?token=${token}`;

  // Send the email (integrate with your email provider)
  await sendEmail({
    to: email,
    subject: 'Sign in to MyApp',
    text: `Click to sign in: ${magicLink}\n\nThis link expires in 15 minutes.`,
  });

  return { message: 'Magic link sent' };
}
```

## Verifying the Token

```javascript
const jwt = require('jsonwebtoken');

async function verifyMagicLink(token) {
  if (!token) throw new Error('Token is required');

  // Find and delete the token atomically
  const record = await MagicToken.findOneAndDelete({ token });

  if (!record) {
    throw new Error('Invalid or expired magic link');
  }

  // Token found and not expired - fetch or create the user
  const user = await User.findOne({ email: record.email });
  if (!user) throw new Error('User not found');

  // Issue a session JWT
  const sessionToken = jwt.sign(
    { userId: user._id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );

  return { sessionToken, user };
}
```

Using `findOneAndDelete` ensures the token is consumed atomically - a token cannot be used twice even under concurrent requests.

## Express Routes

```javascript
const express = require('express');
const router  = express.Router();

// Request magic link
router.post('/auth/magic-link', async (req, res) => {
  try {
    const { email } = req.body;
    await sendMagicLink(email);
    res.json({ message: 'Check your email for a sign-in link' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Verify magic link
router.get('/auth/verify', async (req, res) => {
  try {
    const { token } = req.query;
    const { sessionToken } = await verifyMagicLink(token);
    res.redirect(`/dashboard?token=${sessionToken}`);
  } catch (err) {
    res.redirect('/login?error=invalid_link');
  }
});
```

## Security Considerations

- Always use `crypto.randomBytes(32)` or larger for token generation.
- Delete existing tokens before creating a new one to prevent token accumulation.
- Use `findOneAndDelete` to prevent replay attacks.
- Keep the TTL short (15 minutes or less).
- Rate-limit the magic link endpoint to prevent email spam.

## Summary

MongoDB's TTL index makes magic link authentication elegant - tokens expire and are cleaned up automatically without any cron jobs. The `findOneAndDelete` pattern ensures each token can only be used once. This passwordless approach reduces friction for users while maintaining strong security through cryptographically random, short-lived tokens.
