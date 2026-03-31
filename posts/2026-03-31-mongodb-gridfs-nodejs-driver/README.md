# How to Use GridFS with Node.js and the MongoDB Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GridFS, Node.js, File Storage

Description: A complete guide to using MongoDB GridFS with the official Node.js driver, covering upload, download, delete, and integration with Express for file serving.

---

The official MongoDB Node.js driver includes built-in GridFS support via the `GridFSBucket` class. This guide walks through a complete file management workflow: uploading, downloading, listing, and deleting files, plus an Express integration example.

## Setup

```bash
npm install mongodb
```

```javascript
const { MongoClient, GridFSBucket, ObjectId } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017");
await client.connect();
const db = client.db("myapp");
const bucket = new GridFSBucket(db, {
  bucketName: "uploads",
  chunkSizeBytes: 255 * 1024  // 255 KB per chunk
});
```

## Uploading a File

```javascript
const fs = require("fs");

async function uploadFile(localPath, storedName, metadata = {}) {
  const uploadStream = bucket.openUploadStream(storedName, { metadata });

  await new Promise((resolve, reject) => {
    fs.createReadStream(localPath)
      .pipe(uploadStream)
      .on("finish", resolve)
      .on("error", reject);
  });

  console.log(`Uploaded ${storedName} with ID: ${uploadStream.id}`);
  return uploadStream.id;
}

const fileId = await uploadFile("./contract.pdf", "contract.pdf", {
  owner: "user_42",
  department: "legal"
});
```

## Downloading a File

```javascript
async function downloadFile(fileId, outputPath) {
  const downloadStream = bucket.openDownloadStream(new ObjectId(fileId));
  const writeStream = fs.createWriteStream(outputPath);

  await new Promise((resolve, reject) => {
    downloadStream.pipe(writeStream)
      .on("finish", resolve)
      .on("error", reject);
  });

  console.log(`Downloaded to ${outputPath}`);
}

await downloadFile("64abc123def456789abcdef0", "./output/contract.pdf");
```

## Deleting a File

```javascript
async function deleteFile(fileId) {
  await bucket.delete(new ObjectId(fileId));
  console.log(`Deleted: ${fileId}`);
}
```

## Listing Files

```javascript
async function listFiles(filter = {}) {
  const cursor = bucket.find(filter).sort({ uploadDate: -1 });
  const files = await cursor.toArray();

  return files.map((f) => ({
    id: f._id,
    name: f.filename,
    size: f.length,
    uploaded: f.uploadDate,
    metadata: f.metadata
  }));
}

const allFiles = await listFiles();
const userFiles = await listFiles({ "metadata.owner": "user_42" });
```

## Express File Server Integration

```javascript
const express = require("express");
const multer = require("multer");
const app = express();
const upload = multer({ storage: multer.memoryStorage() });

// Upload endpoint
app.post("/upload", upload.single("file"), async (req, res) => {
  try {
    const uploadStream = bucket.openUploadStream(req.file.originalname, {
      metadata: { contentType: req.file.mimetype }
    });
    uploadStream.end(req.file.buffer);

    await new Promise((resolve, reject) => {
      uploadStream.on("finish", resolve);
      uploadStream.on("error", reject);
    });

    res.json({ id: uploadStream.id, filename: req.file.originalname });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Download endpoint
app.get("/files/:id", async (req, res) => {
  try {
    const fileInfo = await db
      .collection("uploads.files")
      .findOne({ _id: new ObjectId(req.params.id) });

    if (!fileInfo) return res.status(404).send("File not found");

    res.set("Content-Type", fileInfo.metadata?.contentType || "application/octet-stream");
    res.set("Content-Disposition", `attachment; filename="${fileInfo.filename}"`);

    bucket
      .openDownloadStream(new ObjectId(req.params.id))
      .pipe(res);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => console.log("File server running on port 3000"));
```

## Creating Required Indexes

GridFS automatically creates indexes on the first write, but you can ensure they exist explicitly:

```javascript
await db.collection("uploads.files").createIndex({ filename: 1, uploadDate: 1 });
await db.collection("uploads.chunks").createIndex(
  { files_id: 1, n: 1 },
  { unique: true }
);
```

## Summary

The MongoDB Node.js driver's `GridFSBucket` class handles file chunking, storage, and retrieval transparently. Use `openUploadStream` to store files, `openDownloadStream` by ID or name to retrieve them, and `bucket.delete` for removal. For web applications, pipe streams directly to HTTP responses to avoid buffering large files in memory.
