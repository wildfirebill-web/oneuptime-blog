# How to Use GridFS with the MongoDB .NET Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CSharp, GridFS, DotNet, File Storage

Description: Learn how to store, retrieve, and manage large files in MongoDB using GridFS with the official MongoDB .NET Driver in C#.

---

## What Is GridFS?

GridFS stores files that exceed MongoDB's 16 MB document size limit by splitting them into chunks stored in `fs.files` (metadata) and `fs.chunks` (binary data) collections. The .NET Driver provides the `GridFSBucket` class to interact with GridFS.

## Setup

```bash
dotnet add package MongoDB.Driver
```

## Creating a GridFSBucket

```csharp
using MongoDB.Driver;
using MongoDB.Driver.GridFS;

var client = new MongoClient("mongodb://localhost:27017");
var db = client.GetDatabase("myapp");

// Default bucket (prefix: "fs")
var bucket = new GridFSBucket(db);

// Custom bucket with options
var customBucket = new GridFSBucket(db, new GridFSBucketOptions
{
    BucketName = "uploads",
    ChunkSizeBytes = 1024 * 1024, // 1 MB chunks
    WriteConcern = WriteConcern.WMajority,
    ReadPreference = ReadPreference.Secondary
});
```

## Uploading a File

Upload from a `Stream`:

```csharp
using MongoDB.Bson;
using System.IO;

// Upload from file path
using var fileStream = File.OpenRead("/tmp/report.pdf");

var options = new GridFSUploadOptions
{
    Metadata = new BsonDocument
    {
        { "contentType", "application/pdf" },
        { "uploadedBy", "user-123" },
        { "version", 1 }
    }
};

ObjectId fileId = await bucket.UploadFromStreamAsync("report.pdf", fileStream, options);
Console.WriteLine($"Uploaded file ID: {fileId}");
```

Upload from a byte array:

```csharp
byte[] fileBytes = File.ReadAllBytes("/tmp/image.png");
ObjectId imageId = await bucket.UploadFromBytesAsync("image.png", fileBytes);
```

## Downloading a File

Download to a stream by `ObjectId` or filename:

```csharp
// Download by ObjectId to a file
using var outputStream = File.Create("/tmp/downloaded-report.pdf");
await bucket.DownloadToStreamAsync(fileId, outputStream);

// Download by filename (most recent revision)
using var output2 = File.Create("/tmp/report-copy.pdf");
await bucket.DownloadToStreamByNameAsync("report.pdf", output2);

// Download to byte array
byte[] fileBytes = await bucket.DownloadAsBytesAsync(fileId);
```

## Streaming Download (for HTTP responses)

```csharp
// In an ASP.NET Core controller
[HttpGet("files/{id}")]
public async Task<IActionResult> DownloadFile(string id)
{
    var objectId = new ObjectId(id);

    var fileInfo = await bucket.Find(
        Builders<GridFSFileInfo>.Filter.Eq("_id", objectId))
        .FirstOrDefaultAsync();

    if (fileInfo == null) return NotFound();

    var downloadStream = await bucket.OpenDownloadStreamAsync(objectId);
    var contentType = fileInfo.Metadata?
        .GetValue("contentType", "application/octet-stream").AsString;

    return File(downloadStream, contentType, fileInfo.Filename);
}
```

## Listing Files

```csharp
// List all files
var files = await bucket.Find(
    Builders<GridFSFileInfo>.Filter.Empty).ToListAsync();

foreach (var file in files)
{
    Console.WriteLine($"Name: {file.Filename}, " +
                      $"Size: {file.Length} bytes, " +
                      $"Uploaded: {file.UploadDateTime}");
}

// Filter by metadata
var userFiles = await bucket.Find(
    Builders<GridFSFileInfo>.Filter.Eq("metadata.uploadedBy", "user-123"))
    .ToListAsync();
```

## Deleting a File

```csharp
await bucket.DeleteAsync(fileId);
```

This removes both the metadata document and all associated chunks.

## Renaming a File

```csharp
await bucket.RenameAsync(fileId, "report-v2.pdf");
```

## Dropping the Entire Bucket

```csharp
await bucket.DropAsync(); // removes all files and chunks
```

## Summary

GridFS in the MongoDB .NET Driver is managed through `GridFSBucket`. Use `UploadFromStreamAsync` to store files with optional metadata, `DownloadToStreamAsync` or `OpenDownloadStreamAsync` for retrieval, and `Find` with filter builders to locate files by name or metadata. For ASP.NET Core file download endpoints, `OpenDownloadStreamAsync` allows streaming the file directly to the HTTP response without buffering the entire file in memory.
