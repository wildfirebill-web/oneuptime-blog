# How to Use GridFS with the MongoDB Node.js Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GridFS, Node.js, File Storage, Binary Data

Description: Learn how to store, retrieve, and manage large files in MongoDB using GridFS with the official Node.js driver, including streaming uploads and downloads.

---

## Overview

GridFS is a MongoDB specification for storing and retrieving files that exceed the 16 MB BSON document size limit. It divides files into chunks (default 255 KB each) and stores them in two collections: `fs.files` (file metadata) and `fs.chunks` (binary data). GridFS is built into the official Node.js driver.

## Setting Up GridFSBucket

```javascript
const { MongoClient, GridFSBucket } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017");
await client.connect();

const db = client.db("myapp");

// Create a GridFS bucket (default bucket name: "fs")
const bucket = new GridFSBucket(db);

// Custom bucket name and chunk size
const imageBucket = new GridFSBucket(db, {
  bucketName: "images",
  chunkSizeBytes: 1024 * 1024  // 1 MB chunks
});
```

## Uploading a File

### From a file path using streams

```javascript
const fs = require("fs");

async function uploadFile(bucket, localPath, filename, metadata = {}) {
  const uploadStream = bucket.openUploadStream(filename, {
    metadata,
    contentType: "image/jpeg"
  });

  const readStream = fs.createReadStream(localPath);
  readStream.pipe(uploadStream);

  return new Promise((resolve, reject) => {
    uploadStream.on("finish", () => resolve(uploadStream.id));
    uploadStream.on("error", reject);
  });
}

const fileId = await uploadFile(bucket, "./photo.jpg", "profile-photo.jpg", {
  userId: "u123",
  uploadedAt: new Date()
});

console.log("Uploaded file ID:", fileId);
```

### From a Buffer

```javascript
const { Readable } = require("stream");

async function uploadBuffer(bucket, buffer, filename) {
  const uploadStream = bucket.openUploadStream(filename);
  const readable = Readable.from(buffer);
  readable.pipe(uploadStream);

  return new Promise((resolve, reject) => {
    uploadStream.on("finish", () => resolve(uploadStream.id));
    uploadStream.on("error", reject);
  });
}
```

## Downloading a File

### To a local file

```javascript
const { ObjectId } = require("mongodb");
const fs = require("fs");

async function downloadFile(bucket, fileId, outputPath) {
  const downloadStream = bucket.openDownloadStream(new ObjectId(fileId));
  const writeStream = fs.createWriteStream(outputPath);

  downloadStream.pipe(writeStream);

  return new Promise((resolve, reject) => {
    writeStream.on("finish", resolve);
    downloadStream.on("error", reject);
  });
}

await downloadFile(bucket, "64a1234abc...", "./downloaded-photo.jpg");
```

### Stream to an HTTP response (Express.js)

```javascript
const express = require("express");
const { ObjectId } = require("mongodb");

const app = express();

app.get("/files/:id", async (req, res) => {
  try {
    const fileId = new ObjectId(req.params.id);

    // Check if the file exists
    const files = await bucket.find({ _id: fileId }).toArray();
    if (!files.length) {
      return res.status(404).json({ error: "File not found" });
    }

    res.set("Content-Type", files[0].contentType || "application/octet-stream");
    res.set("Content-Disposition", `attachment; filename="${files[0].filename}"`);

    const downloadStream = bucket.openDownloadStream(fileId);
    downloadStream.pipe(res);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

## Listing Files

```javascript
// List all files in the bucket
const files = await bucket.find({}).toArray();
files.forEach((file) => {
  console.log(file.filename, file.length, file.uploadDate);
});

// Filter by metadata
const userFiles = await bucket.find({ "metadata.userId": "u123" }).toArray();
```

## Deleting a File

```javascript
const { ObjectId } = require("mongodb");

async function deleteFile(bucket, fileId) {
  await bucket.delete(new ObjectId(fileId));
  console.log("File deleted");
}
```

## Renaming a File

```javascript
await bucket.rename(new ObjectId(fileId), "new-filename.jpg");
```

## Handling Multipart Uploads (with multer)

```javascript
const multer = require("multer");
const upload = multer({ storage: multer.memoryStorage() });

app.post("/upload", upload.single("file"), async (req, res) => {
  const { buffer, originalname, mimetype } = req.file;

  const uploadStream = bucket.openUploadStream(originalname, {
    contentType: mimetype,
    metadata: { uploadedBy: req.user.id }
  });

  uploadStream.end(buffer);

  uploadStream.on("finish", () => {
    res.status(201).json({ fileId: uploadStream.id });
  });

  uploadStream.on("error", (err) => {
    res.status(500).json({ error: err.message });
  });
});
```

## Summary

GridFS enables MongoDB to store files of any size by splitting them into chunks stored in two collections. The `GridFSBucket` API in the Node.js driver provides streaming upload and download methods. Stream files directly to HTTP responses to avoid buffering large files in memory, use metadata fields to associate files with users or entities, and always check for file existence before streaming to avoid unhandled stream errors.
