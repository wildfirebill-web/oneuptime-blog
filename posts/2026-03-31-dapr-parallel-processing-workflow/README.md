# How to Implement Parallel Processing Workflow with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Parallel Processing, Fan-Out, Orchestration

Description: Learn how to implement parallel processing workflows using Dapr Workflow to fan out work across multiple activities simultaneously and aggregate the results.

---

## Why Parallel Processing in Workflows?

Sequential workflows process steps one at a time, which is slow when steps are independent. Dapr Workflow supports parallel activity execution using `Task.WhenAll`, enabling fan-out patterns that run all independent steps concurrently and collect results when all complete.

## Image Processing Workflow (Fan-Out / Fan-In)

This workflow receives an uploaded image and simultaneously resizes it to three sizes, generates a thumbnail, and runs content moderation:

```csharp
public class ImageProcessingWorkflow : Workflow<ImageUploadRequest, ImageProcessingResult>
{
    public override async Task<ImageProcessingResult> RunAsync(
        WorkflowContext context, ImageUploadRequest request)
    {
        // Fan out - start all activities in parallel
        var resizeSmall = context.CallActivityAsync<ResizedImage>(
            nameof(ResizeImageActivity),
            new ResizeRequest { ImageId = request.ImageId, Width = 320, Height = 240 });

        var resizeMedium = context.CallActivityAsync<ResizedImage>(
            nameof(ResizeImageActivity),
            new ResizeRequest { ImageId = request.ImageId, Width = 800, Height = 600 });

        var resizeLarge = context.CallActivityAsync<ResizedImage>(
            nameof(ResizeImageActivity),
            new ResizeRequest { ImageId = request.ImageId, Width = 1920, Height = 1080 });

        var thumbnail = context.CallActivityAsync<ResizedImage>(
            nameof(GenerateThumbnailActivity),
            new ThumbnailRequest { ImageId = request.ImageId, Size = 64 });

        var moderation = context.CallActivityAsync<ModerationResult>(
            nameof(ContentModerationActivity),
            new ModerationRequest { ImageId = request.ImageId });

        // Fan in - wait for all to complete
        await Task.WhenAll(resizeSmall, resizeMedium, resizeLarge, thumbnail, moderation);

        var moderationResult = await moderation;
        if (!moderationResult.IsSafe)
        {
            await context.CallActivityAsync(
                nameof(DeleteImageActivity), request.ImageId);
            return new ImageProcessingResult { Status = "rejected", Reason = moderationResult.Reason };
        }

        return new ImageProcessingResult
        {
            Status = "processed",
            Variants = new[]
            {
                (await resizeSmall).Url,
                (await resizeMedium).Url,
                (await resizeLarge).Url,
                (await thumbnail).Url
            }
        };
    }
}
```

## Dynamic Fan-Out with Unknown Number of Tasks

```csharp
public class DataExportWorkflow : Workflow<ExportRequest, ExportResult>
{
    public override async Task<ExportResult> RunAsync(
        WorkflowContext context, ExportRequest request)
    {
        // Get list of partitions to export
        var partitions = await context.CallActivityAsync<List<string>>(
            nameof(GetPartitionsActivity), request);

        // Fan out across all partitions dynamically
        var exportTasks = partitions.Select(partition =>
            context.CallActivityAsync<ExportPartitionResult>(
                nameof(ExportPartitionActivity),
                new { Partition = partition, Format = request.Format }
            )
        ).ToList();

        var results = await Task.WhenAll(exportTasks);

        // Fan in - merge all partition results
        var merge = await context.CallActivityAsync<string>(
            nameof(MergeExportFilesActivity),
            results.Select(r => r.FilePath).ToArray());

        return new ExportResult { MergedFilePath = merge, PartitionCount = results.Length };
    }
}
```

## Submitting a Parallel Workflow

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

app.post('/images/:imageId/process', async (req, res) => {
  const instanceId = `image-processing-${req.params.imageId}`;

  await client.workflow.start('ImageProcessingWorkflow', {
    instanceId,
    input: {
      imageId: req.params.imageId,
      uploadedBy: req.user.id
    }
  });

  res.json({ instanceId, message: 'Processing started' });
});
```

## Polling for Completion

```javascript
app.get('/images/:imageId/status', async (req, res) => {
  const instanceId = `image-processing-${req.params.imageId}`;
  const status = await client.workflow.get(instanceId);

  res.json({
    status: status.runtimeStatus,
    result: status.runtimeStatus === 'COMPLETED'
      ? JSON.parse(status.serializedOutput)
      : null
  });
});
```

## Error Handling in Parallel Tasks

```csharp
// Collect errors from all tasks instead of short-circuiting on first failure
var results = await Task.WhenAll(
    exportTasks.Select(async t => {
        try { return await t; }
        catch (Exception ex) { return new ExportPartitionResult { Error = ex.Message }; }
    })
);
var failures = results.Where(r => r.Error != null).ToList();
```

## Summary

Dapr Workflow parallel processing uses `Task.WhenAll` to fan out to multiple activities simultaneously. Both fixed-count and dynamic fan-out are supported. The workflow engine persists intermediate state so parallel branches survive service restarts, and the fan-in step only executes once every branch completes successfully.
