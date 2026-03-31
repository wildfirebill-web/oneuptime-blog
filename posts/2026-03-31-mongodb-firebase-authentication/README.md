# How to Use MongoDB with Firebase Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Firebase, Authentication, JWT, Node

Description: Learn how to verify Firebase ID tokens in a Node.js backend and store application-specific user data in MongoDB alongside Firebase identity records.

---

Firebase Authentication handles sign-in flows (Google, email/password, phone) while MongoDB stores the rest of your application data. The integration relies on verifying Firebase ID tokens server-side and syncing user profiles to MongoDB on first access.

## Installing Dependencies

```bash
npm install firebase-admin mongoose express
```

## Initializing Firebase Admin SDK

```javascript
const admin = require('firebase-admin');

admin.initializeApp({
  credential: admin.credential.cert({
    projectId: process.env.FIREBASE_PROJECT_ID,
    privateKey: process.env.FIREBASE_PRIVATE_KEY.replace(/\\n/g, '\n'),
    clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
  }),
});
```

## Defining the MongoDB User Model

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  firebaseUid: { type: String, required: true, unique: true, index: true },
  email: String,
  displayName: String,
  photoURL: String,
  emailVerified: { type: Boolean, default: false },
  // Application-specific data
  role: { type: String, enum: ['user', 'moderator', 'admin'], default: 'user' },
  bio: String,
  onboarded: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  lastSeenAt: Date,
});

module.exports = mongoose.model('User', userSchema);
```

## Verifying Firebase Tokens in Express Middleware

```javascript
const User = require('./models/User');

async function firebaseAuth(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing authorization header' });
  }

  const idToken = authHeader.split('Bearer ')[1];

  try {
    const decoded = await admin.auth().verifyIdToken(idToken);

    // Upsert user in MongoDB
    const user = await User.findOneAndUpdate(
      { firebaseUid: decoded.uid },
      {
        $set: {
          email: decoded.email,
          displayName: decoded.name,
          photoURL: decoded.picture,
          emailVerified: decoded.email_verified,
          lastSeenAt: new Date(),
        },
        $setOnInsert: { firebaseUid: decoded.uid, role: 'user' },
      },
      { upsert: true, new: true }
    );

    req.firebaseUser = decoded;
    req.dbUser = user;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```

## Protected Route Example

```javascript
app.get('/api/profile', firebaseAuth, (req, res) => {
  res.json({
    uid: req.firebaseUser.uid,
    mongoId: req.dbUser._id,
    role: req.dbUser.role,
    onboarded: req.dbUser.onboarded,
  });
});

app.patch('/api/profile', firebaseAuth, async (req, res) => {
  const { bio, displayName } = req.body;
  const user = await User.findByIdAndUpdate(
    req.dbUser._id,
    { $set: { bio, displayName, onboarded: true } },
    { new: true }
  );
  res.json(user);
});
```

## Revoking Tokens and Syncing Deletions

When a user is deleted in Firebase, clean up MongoDB too:

```javascript
async function deleteUser(firebaseUid) {
  // Revoke tokens in Firebase
  await admin.auth().revokeRefreshTokens(firebaseUid);
  await admin.auth().deleteUser(firebaseUid);

  // Remove from MongoDB
  await User.deleteOne({ firebaseUid });
}
```

## Querying MongoDB for App Analytics

```javascript
// Find users who completed onboarding
db.users.find({ onboarded: true, emailVerified: true })

// Count users by role
db.users.aggregate([
  { $group: { _id: "$role", total: { $sum: 1 } } },
  { $sort: { total: -1 } }
])

// Find inactive users (not seen in 60 days)
db.users.find({
  lastSeenAt: { $lt: new Date(Date.now() - 60 * 86400000) }
})
```

## Summary

Firebase Authentication and MongoDB complement each other well - Firebase manages identity and token issuance while MongoDB stores rich application data. Use `firebaseUid` as the foreign key between systems, verify tokens with the Firebase Admin SDK on every request, and upsert user documents to keep both stores consistent without webhooks.
