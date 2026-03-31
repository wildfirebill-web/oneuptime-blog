# How to Use mongofiles to Manage GridFS Files in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, mongofiles, GridFS, File Storage, Binary Data

Description: Use mongofiles to upload, download, list, and delete files in MongoDB GridFS for storing large binary objects like images and documents.

---

## What Is GridFS?

GridFS is MongoDB's specification for storing and retrieving files larger than the 16 MB BSON document size limit. It splits files into chunks (default 255 KB each) stored in `fs.chunks` and metadata stored in `fs.files`. `mongofiles` is the command-line interface for managing GridFS.

## Prerequisites

`mongofiles` is part of the MongoDB Database Tools package:

```bash
# Ubuntu/Debian
sudo apt-get install mongodb-database-tools

# macOS
brew install mongodb-database-tools
```

## Uploading a File

```bash
mongofiles \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myfiles \
  put /path/to/document.pdf
```

Upload with a custom remote filename:

```bash
mongofiles --db myfiles put /local/path/image.jpg --local image.jpg --filename uploads/2026/image.jpg
```

## Listing Files

List all files stored in GridFS:

```bash
mongofiles --db myfiles list
```

Filter by filename prefix:

```bash
mongofiles --db myfiles list uploads/2026/
```

Output:

```text
2026-01-15T10:30:00.000+0000	uploads/2026/image.jpg	245760
2026-01-15T11:00:00.000+0000	uploads/2026/report.pdf	1048576
```

## Downloading a File

```bash
mongofiles \
  --db myfiles \
  get uploads/2026/image.jpg
```

Download to a specific local path:

```bash
mongofiles \
  --db myfiles \
  get uploads/2026/report.pdf \
  --local /tmp/report.pdf
```

## Searching for Files

Search by filename pattern:

```bash
mongofiles --db myfiles search "report"
```

## Deleting a File

```bash
mongofiles --db myfiles delete uploads/2026/old-file.jpg
```

## Working with Multiple GridFS Buckets

By default, `mongofiles` uses the `fs` bucket. To use a custom bucket:

```bash
mongofiles \
  --db myfiles \
  --prefix avatars \
  put /path/to/user-avatar.png
```

This stores files in `avatars.files` and `avatars.chunks` instead of `fs.files` and `fs.chunks`.

## Querying GridFS from mongosh

```javascript
use myfiles

// List all files
db.fs.files.find({}, { filename: 1, length: 1, uploadDate: 1 })

// Find files larger than 1MB
db.fs.files.find({ length: { $gt: 1048576 } })

// Check chunks for a file
const file = db.fs.files.findOne({ filename: "uploads/2026/report.pdf" });
db.fs.chunks.countDocuments({ files_id: file._id });
```

## Creating GridFS Indexes

GridFS collections should have proper indexes:

```javascript
// These are created automatically but can be re-created if needed
db.fs.files.createIndex({ filename: 1, uploadDate: 1 })
db.fs.chunks.createIndex({ files_id: 1, n: 1 }, { unique: true })
```

## Streaming Files with the Driver

For application use, use the GridFS streaming API instead of `mongofiles`:

```javascript
const { MongoClient, GridFSBucket } = require("mongodb");
const fs = require("fs");

const client = new MongoClient("mongodb://localhost:27017");
await client.connect();
const db = client.db("myfiles");
const bucket = new GridFSBucket(db);

// Upload
fs.createReadStream("/local/file.pdf")
  .pipe(bucket.openUploadStream("document.pdf"))
  .on("finish", () => console.log("Uploaded"));

// Download
bucket.openDownloadStreamByName("document.pdf")
  .pipe(fs.createWriteStream("/tmp/document.pdf"));
```

## Summary

`mongofiles` provides a simple CLI interface to MongoDB GridFS for uploading, downloading, listing, and deleting files. Use custom prefixes for multiple storage buckets, and query `fs.files` directly from mongosh for metadata lookups. For production applications, use the GridFSBucket API in your driver for streaming support.
