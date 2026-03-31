# How to Use GridFS with MongoDB PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, PHP, GridFS, File Storage, Library

Description: Learn how to store and retrieve large files in MongoDB using GridFS with the official MongoDB PHP Library.

---

## What Is GridFS?

GridFS stores files exceeding MongoDB's 16 MB BSON document size limit by splitting them into chunks stored in `fs.files` (metadata) and `fs.chunks` (binary data) collections. The MongoDB PHP Library provides a `GridFSBucket` class for seamless file management.

## Setup

```bash
composer require mongodb/mongodb
```

## Creating a GridFSBucket

```php
<?php
require 'vendor/autoload.php';

use MongoDB\Client;

$client = new Client('mongodb://localhost:27017');
$db = $client->myapp;

// Default bucket (prefix: "fs")
$bucket = $db->selectGridFSBucket();

// Custom bucket with options
$customBucket = $db->selectGridFSBucket([
    'bucketName'    => 'uploads',
    'chunkSizeBytes' => 1048576, // 1 MB
]);
```

## Uploading a File

Upload from a stream or file path:

```php
// Upload from a file
$stream = fopen('/tmp/report.pdf', 'rb');

$fileId = $bucket->uploadFromStream('report.pdf', $stream, [
    'metadata' => [
        'contentType' => 'application/pdf',
        'uploadedBy'  => 'user-123',
        'version'     => 1,
    ],
]);

fclose($stream);
echo "Uploaded file ID: $fileId\n";
```

Upload from a string in memory:

```php
$content = file_get_contents('/tmp/image.png');
$stream = fopen('php://temp', 'w+b');
fwrite($stream, $content);
rewind($stream);

$imageId = $bucket->uploadFromStream('image.png', $stream);
fclose($stream);
```

## Opening a GridFS Upload Stream

For large files, write chunks progressively:

```php
$uploadStream = $bucket->openUploadStream('video.mp4', [
    'metadata' => ['contentType' => 'video/mp4'],
]);

$source = fopen('/tmp/video.mp4', 'rb');
while (!feof($source)) {
    $chunk = fread($source, 65536); // 64 KB
    fwrite($uploadStream, $chunk);
}
fclose($source);
fclose($uploadStream);
```

## Downloading a File

```php
use MongoDB\BSON\ObjectId;

$fileId = new ObjectId('507f1f77bcf86cd799439011');

// Download to a file
$destination = fopen('/tmp/downloaded.pdf', 'wb');
$bucket->downloadToStream($fileId, $destination);
fclose($destination);

// Download to memory
$stream = fopen('php://temp', 'w+b');
$bucket->downloadToStream($fileId, $stream);
rewind($stream);
$contents = stream_get_contents($stream);
fclose($stream);
```

## Streaming to an HTTP Response

```php
// In a PHP web handler:
$fileId = new ObjectId($_GET['id']);

// Get file metadata first
$fileInfo = $bucket->findOne(['_id' => $fileId]);
if (!$fileInfo) {
    http_response_code(404);
    exit;
}

$contentType = $fileInfo['metadata']['contentType'] ?? 'application/octet-stream';
header("Content-Type: $contentType");
header("Content-Disposition: attachment; filename=\"{$fileInfo['filename']}\"");

$downloadStream = $bucket->openDownloadStream($fileId);
while (!feof($downloadStream)) {
    echo fread($downloadStream, 65536);
    flush();
}
fclose($downloadStream);
```

## Listing Files

```php
// List all files
$files = $bucket->find([]);
foreach ($files as $file) {
    echo "Name: {$file['filename']}, "
       . "Size: {$file['length']} bytes, "
       . "Uploaded: {$file['uploadDate']}\n";
}

// Filter by metadata
$userFiles = $bucket->find([
    'metadata.uploadedBy' => 'user-123',
]);
```

## Deleting a File

```php
$bucket->delete($fileId);
echo "File deleted\n";
```

## Renaming a File

```php
$bucket->rename($fileId, 'report-v2.pdf');
```

## Dropping the Bucket

```php
$bucket->drop(); // removes all files and chunks
```

## Summary

GridFS in the MongoDB PHP Library is accessed through `$db->selectGridFSBucket()`. Use `uploadFromStream()` for complete file uploads with metadata, `openUploadStream()` for large progressive uploads, and `downloadToStream()` or `openDownloadStream()` for retrieval. For HTTP download endpoints, stream directly to `php://output` without buffering the entire file in memory. Always close stream resources with `fclose()` after use.
