# How to Implement Password Reset Flow with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Password, Reset, Security

Description: Learn how to build a secure password reset flow with MongoDB using hashed tokens and TTL indexes to safely allow users to recover their accounts.

---

## Overview

A password reset flow lets users regain access to their account via email when they forget their password. The implementation must be secure: tokens must be random, short-lived, single-use, and stored as hashes - not plain text - to prevent token theft from the database.

## Token Schema with TTL

```javascript
// models/PasswordResetToken.js
const mongoose = require('mongoose');

const resetTokenSchema = new mongoose.Schema({
  userId:    { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  tokenHash: { type: String, required: true },
  createdAt: { type: Date, default: Date.now, expires: '1h' }, // Auto-delete after 1 hour
});

module.exports = mongoose.model('PasswordResetToken', resetTokenSchema);
```

The `expires` TTL index ensures orphaned tokens are automatically removed after one hour.

## Requesting a Password Reset

```javascript
const crypto = require('crypto');
const bcrypt = require('bcrypt');

async function requestPasswordReset(email) {
  const user = await User.findOne({ email });

  // Always respond with the same message to prevent email enumeration
  if (!user) return { message: 'If that email exists, a reset link has been sent.' };

  // Delete any existing reset tokens for this user
  await PasswordResetToken.deleteMany({ userId: user._id });

  // Generate a cryptographically secure token
  const plainToken = crypto.randomBytes(32).toString('hex');
  const tokenHash  = await bcrypt.hash(plainToken, 10);

  await PasswordResetToken.create({
    userId: user._id,
    tokenHash,
  });

  const resetLink = `${process.env.BASE_URL}/auth/reset-password?token=${plainToken}&userId=${user._id}`;

  await sendEmail({
    to: email,
    subject: 'Reset your password',
    text: `Reset your password here: ${resetLink}\n\nThis link expires in 1 hour.`,
  });

  return { message: 'If that email exists, a reset link has been sent.' };
}
```

## Verifying the Token and Setting a New Password

```javascript
async function resetPassword(userId, plainToken, newPassword) {
  const tokenRecord = await PasswordResetToken.findOne({ userId });

  if (!tokenRecord) {
    throw new Error('Invalid or expired reset token');
  }

  // Compare the plain token against the stored hash
  const isValid = await bcrypt.compare(plainToken, tokenRecord.tokenHash);
  if (!isValid) {
    throw new Error('Invalid or expired reset token');
  }

  // Hash the new password
  const passwordHash = await bcrypt.hash(newPassword, 12);

  // Update the user's password and delete the token
  await Promise.all([
    User.findByIdAndUpdate(userId, { password: passwordHash }),
    PasswordResetToken.deleteOne({ userId }),
  ]);

  return { message: 'Password updated successfully' };
}
```

## Express Routes

```javascript
const express = require('express');
const router  = express.Router();

router.post('/auth/forgot-password', async (req, res) => {
  const result = await requestPasswordReset(req.body.email);
  res.json(result);
});

router.post('/auth/reset-password', async (req, res) => {
  try {
    const { userId, token, newPassword } = req.body;

    if (newPassword.length < 8) {
      return res.status(400).json({ error: 'Password must be at least 8 characters' });
    }

    const result = await resetPassword(userId, token, newPassword);
    res.json(result);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});
```

## Security Checklist

- Store only the token hash in MongoDB, never the plain token.
- Delete existing tokens before creating a new one.
- Use constant-time comparison (`bcrypt.compare`) to prevent timing attacks.
- Return identical messages for valid and invalid emails to prevent user enumeration.
- Invalidate the token immediately after a successful reset.
- Consider invalidating all active sessions after a password reset.

## Invalidating Sessions After Reset

```javascript
// If using session tokens in MongoDB, invalidate all sessions on reset
await Session.deleteMany({ userId });
```

## Summary

A secure password reset flow in MongoDB stores token hashes (not plain tokens), uses TTL indexes for automatic expiration, and consumes tokens atomically on use. The key security principles are: prevent email enumeration with uniform responses, use bcrypt for token hashing, and immediately invalidate the token and optionally all sessions after a successful reset.
