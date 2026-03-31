# How to Store Large Files in S3 and References in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, S3, Storage, File, AWS

Description: Store large files in Amazon S3 and keep metadata references in MongoDB to build scalable file management systems with pre-signed URLs.

---

## Introduction

MongoDB documents are limited to 16MB, making it unsuitable for storing large files directly. The recommended pattern is storing files in Amazon S3 (or S3-compatible storage) and keeping file metadata - including the S3 key - as a document in MongoDB. This gives you MongoDB's querying power for metadata and S3's scalability for binary data.

## Architecture

```text
Upload: Client -> API -> S3 (binary) + MongoDB (metadata)
Access: Client -> API -> MongoDB (get S3 key) -> S3 (pre-signed URL) -> Client
```

## Setting Up AWS SDK and MongoDB

```javascript
const { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } = require("@aws-sdk/client-s3")
const { getSignedUrl } = require("@aws-sdk/s3-request-presigner")
const { MongoClient, ObjectId } = require("mongodb")
const multer = require("multer")

const s3 = new S3Client({ region: process.env.AWS_REGION })
const BUCKET = process.env.S3_BUCKET

const mongo = new MongoClient(process.env.MONGODB_URI)
await mongo.connect()
const db = mongo.db("myapp")
```

## Upload Handler

```javascript
const upload = multer({ storage: multer.memoryStorage() })

app.post("/files/upload", upload.single("file"), async (req, res) => {
  const file = req.file
  const userId = req.user.id
  const s3Key = `uploads/${userId}/${Date.now()}-${file.originalname}`

  // Upload to S3
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: s3Key,
    Body: file.buffer,
    ContentType: file.mimetype,
    Metadata: {
      uploadedBy: userId
    }
  }))

  // Store metadata in MongoDB
  const result = await db.collection("files").insertOne({
    filename: file.originalname,
    s3Key,
    s3Bucket: BUCKET,
    contentType: file.mimetype,
    size: file.size,
    uploadedBy: new ObjectId(userId),
    createdAt: new Date(),
    tags: req.body.tags ? req.body.tags.split(",") : []
  })

  res.json({
    fileId: result.insertedId,
    filename: file.originalname,
    size: file.size
  })
})
```

## Generating Pre-Signed Download URLs

```javascript
app.get("/files/:fileId/download", async (req, res) => {
  const file = await db.collection("files").findOne({
    _id: new ObjectId(req.params.fileId),
    uploadedBy: new ObjectId(req.user.id)
  })

  if (!file) return res.status(404).json({ error: "File not found" })

  // Generate a pre-signed URL valid for 15 minutes
  const url = await getSignedUrl(
    s3,
    new GetObjectCommand({
      Bucket: file.s3Bucket,
      Key: file.s3Key,
      ResponseContentDisposition: `attachment; filename="${file.filename}"`
    }),
    { expiresIn: 900 }
  )

  res.json({ url, expiresIn: 900 })
})
```

## Querying File Metadata

```javascript
// Find all files uploaded by a user, sorted by date
const userFiles = await db.collection("files")
  .find({ uploadedBy: new ObjectId(userId) })
  .sort({ createdAt: -1 })
  .project({ filename: 1, size: 1, contentType: 1, createdAt: 1 })
  .toArray()

// Search files by tag
const taggedFiles = await db.collection("files")
  .find({ tags: "invoice", uploadedBy: new ObjectId(userId) })
  .toArray()
```

## Deleting Files

```javascript
app.delete("/files/:fileId", async (req, res) => {
  const file = await db.collection("files").findOne({
    _id: new ObjectId(req.params.fileId)
  })

  if (!file) return res.status(404).json({ error: "Not found" })

  // Delete from S3
  await s3.send(new DeleteObjectCommand({
    Bucket: file.s3Bucket,
    Key: file.s3Key
  }))

  // Delete metadata from MongoDB
  await db.collection("files").deleteOne({ _id: file._id })

  res.json({ success: true })
})
```

## Summary

Storing large files in S3 with MongoDB metadata references is the standard pattern for scalable file management. S3 handles binary storage with high durability, while MongoDB enables rich metadata queries, access control, and tag-based organization. Always generate pre-signed URLs for downloads rather than proxying files through your API, and ensure delete operations clean up both the S3 object and the MongoDB document to prevent orphaned storage costs.
