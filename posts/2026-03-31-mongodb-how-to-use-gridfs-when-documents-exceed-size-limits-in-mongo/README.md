# How to Use GridFS When Documents Exceed Size Limits in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GridFS, File Storage, Large File, Binary Data

Description: Learn how to use MongoDB GridFS to store and retrieve files larger than 16MB by splitting them into chunks stored across multiple documents.

---

## What Is GridFS

GridFS is a specification for storing and retrieving files that exceed MongoDB's 16MB BSON document size limit. It splits files into chunks (default 255KB each) stored in two collections: `fs.files` (metadata) and `fs.chunks` (binary data).

## When to Use GridFS

Use GridFS when:
- Files exceed 16MB (mandatory)
- You need metadata stored alongside file content
- You want partial reads (streaming sections of a large file)
- You already use MongoDB and want to avoid adding separate file storage

For files under 16MB, consider storing binary data directly as BSON Binary type.

## Setup

```javascript
const { MongoClient, GridFSBucket } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017");
await client.connect();
const db = client.db("myapp");

const bucket = new GridFSBucket(db, {
  bucketName: "uploads",
  chunkSizeBytes: 261120  // 255KB chunks (default)
});
```

## Uploading a File

```javascript
const fs = require("fs");
const path = require("path");

async function uploadFile(filePath) {
  const filename = path.basename(filePath);

  const uploadStream = bucket.openUploadStream(filename, {
    metadata: {
      uploadedBy: "user123",
      contentType: "application/pdf",
      tags: ["report", "2026"]
    }
  });

  await new Promise((resolve, reject) => {
    fs.createReadStream(filePath)
      .pipe(uploadStream)
      .on("finish", resolve)
      .on("error", reject);
  });

  console.log("Uploaded file ID:", uploadStream.id);
  return uploadStream.id;
}
```

## Downloading a File

```javascript
async function downloadFile(fileId, outputPath) {
  const downloadStream = bucket.openDownloadStream(fileId);

  await new Promise((resolve, reject) => {
    downloadStream
      .pipe(fs.createWriteStream(outputPath))
      .on("finish", resolve)
      .on("error", reject);
  });
}

// Download by filename
async function downloadByName(filename, outputPath) {
  const downloadStream = bucket.openDownloadStreamByName(filename);
  // pipe to output...
}
```

## Listing Files

```javascript
const files = await bucket.find({
  "metadata.uploadedBy": "user123"
}).toArray();

files.forEach(file => {
  console.log(file.filename, file.length, file.uploadDate);
});
```

## Streaming to HTTP Response (Express.js)

```javascript
app.get("/files/:fileId", async (req, res) => {
  const fileId = new ObjectId(req.params.fileId);

  const file = await bucket.find({ _id: fileId }).next();
  if (!file) return res.status(404).send("Not found");

  res.set({
    "Content-Type": file.metadata.contentType,
    "Content-Length": file.length,
    "Content-Disposition": `inline; filename="${file.filename}"`
  });

  bucket.openDownloadStream(fileId).pipe(res);
});
```

## Deleting a File

```javascript
async function deleteFile(fileId) {
  await bucket.delete(fileId);
  console.log("Deleted file and all chunks");
}
```

## PyMongo Example

```python
from gridfs import GridFS
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client.myapp
fs = GridFS(db, collection="uploads")

# Upload
with open("/path/to/large_file.zip", "rb") as f:
    file_id = fs.put(f, filename="large_file.zip", contentType="application/zip")

# Download
with fs.get(file_id) as f:
    with open("/tmp/downloaded.zip", "wb") as out:
        out.write(f.read())
```

## Summary

GridFS stores files exceeding MongoDB's 16MB limit by splitting them into configurable-size chunks across two collections. Use the GridFSBucket API in Node.js (or GridFS in PyMongo) to upload, download, and list files. Store metadata alongside each file and stream content directly to HTTP responses to avoid loading entire files into memory.
