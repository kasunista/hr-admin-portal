# Azure Blob Storage Integration Guide

This guide explains how to integrate your Azure Blob Storage with the HR Admin Portal for document management and AI processing.

## Architecture Overview

The HR Admin Portal works with your existing Azure RAG application in the following flow:

1. **Admin uploads documents** via the portal's upload interface
2. **Documents are stored** in Azure Blob Storage
3. **Azure AI Search** indexes documents daily (as configured in your RAG app)
4. **OpenAI backend** uses indexed documents to answer HR questions

## Configuration Steps

### Step 1: Gather Azure Credentials

You'll need the following information from your Azure account:

- **Storage Account Name**: The name of your Azure Storage Account (e.g., `hrstorageaccount`)
- **Storage Account Key**: The primary or secondary access key from your storage account
- **Container Name**: The name of your blob container (e.g., `hr-documents-prod`)

### Step 2: Set Environment Variables

Add the following environment variables to your `.env` file or deployment configuration:

```bash
AZURE_STORAGE_ACCOUNT_NAME=your_account_name
AZURE_STORAGE_ACCOUNT_KEY=your_account_key
AZURE_STORAGE_CONTAINER_NAME=your_container_name
```

### Step 3: Backend Implementation

The backend code needs to be updated to handle Azure Blob Storage operations. Here's where to add the Azure integration:

#### File: `server/routers.ts`

Add a new router for document operations:

```typescript
import { publicProcedure, router } from "./_core/trpc";
import { z } from "zod";
import { BlobServiceClient } from "@azure/storage-blob";

const blobServiceClient = BlobServiceClient.fromConnectionString(
  `DefaultEndpointsProtocol=https;AccountName=${process.env.AZURE_STORAGE_ACCOUNT_NAME};AccountKey=${process.env.AZURE_STORAGE_ACCOUNT_KEY};EndpointSuffix=core.windows.net`
);

export const documentsRouter = router({
  // List all documents in the container
  list: publicProcedure.query(async () => {
    try {
      const containerClient = blobServiceClient.getContainerClient(
        process.env.AZURE_STORAGE_CONTAINER_NAME || ""
      );

      const documents = [];
      for await (const blob of containerClient.listBlobsFlat()) {
        documents.push({
          id: blob.name,
          name: blob.name,
          size: blob.properties.contentLength || 0,
          uploadedAt: blob.properties.createdOn?.toISOString() || "",
          url: `${containerClient.getBlockBlobClient(blob.name).url}`,
        });
      }

      return documents;
    } catch (error) {
      console.error("Failed to list documents:", error);
      throw new Error("Failed to list documents from Azure Blob Storage");
    }
  }),

  // Upload a document
  upload: publicProcedure
    .input(
      z.object({
        fileName: z.string(),
        fileData: z.instanceof(Buffer),
      })
    )
    .mutation(async ({ input }) => {
      try {
        const containerClient = blobServiceClient.getContainerClient(
          process.env.AZURE_STORAGE_CONTAINER_NAME || ""
        );

        const blockBlobClient = containerClient.getBlockBlobClient(input.fileName);

        await blockBlobClient.upload(input.fileData, input.fileData.length);

        return {
          success: true,
          fileName: input.fileName,
          url: blockBlobClient.url,
        };
      } catch (error) {
        console.error("Failed to upload document:", error);
        throw new Error("Failed to upload document to Azure Blob Storage");
      }
    }),

  // Delete a document
  delete: publicProcedure
    .input(z.object({ fileName: z.string() }))
    .mutation(async ({ input }) => {
      try {
        const containerClient = blobServiceClient.getContainerClient(
          process.env.AZURE_STORAGE_CONTAINER_NAME || ""
        );

        const blockBlobClient = containerClient.getBlockBlobClient(input.fileName);
        await blockBlobClient.delete();

        return { success: true };
      } catch (error) {
        console.error("Failed to delete document:", error);
        throw new Error("Failed to delete document from Azure Blob Storage");
      }
    }),
});
```

Then add this router to your main `appRouter`:

```typescript
export const appRouter = router({
  system: systemRouter,
  auth: router({
    me: publicProcedure.query(opts => opts.ctx.user),
    logout: publicProcedure.mutation(({ ctx }) => {
      // ... existing logout logic
    }),
  }),
  documents: documentsRouter,
});
```

### Step 4: Install Azure SDK

Install the required Azure Storage Blob SDK:

```bash
npm install @azure/storage-blob
```

Or if using pnpm:

```bash
pnpm add @azure/storage-blob
```

### Step 5: Update Frontend to Use Backend

Update `client/src/pages/AdminPortal.tsx` to call the backend procedures instead of using local state:

```typescript
import { trpc } from "@/lib/trpc";

export default function AdminPortal({ adminName, onLogout }: AdminPortalProps) {
  const { data: documents = [], isLoading } = trpc.documents.list.useQuery();
  const uploadMutation = trpc.documents.upload.useMutation();
  const deleteMutation = trpc.documents.delete.useMutation();

  const handleUpload = async () => {
    if (!selectedFile) return;

    try {
      const arrayBuffer = await selectedFile.arrayBuffer();
      const buffer = Buffer.from(arrayBuffer);

      await uploadMutation.mutateAsync({
        fileName: selectedFile.name,
        fileData: buffer,
      });

      // Refresh documents list
      trpc.useUtils().documents.list.invalidate();
    } catch (error) {
      setUploadMessage({ type: "error", text: "Upload failed" });
    }
  };

  const handleDelete = async (fileName: string) => {
    try {
      await deleteMutation.mutateAsync({ fileName });
      trpc.useUtils().documents.list.invalidate();
    } catch (error) {
      console.error("Delete failed:", error);
    }
  };

  // ... rest of the component
}
```

## Integration with Azure AI Search

Once documents are uploaded to your blob container, your Azure AI Search indexer (configured in your RAG application) will:

1. **Automatically index** new documents based on your scheduled indexing (daily, as mentioned)
2. **Extract text** from PDFs, Word documents, and other formats
3. **Create searchable embeddings** for semantic search
4. **Feed data** to your OpenAI backend for question-answering

No additional configuration is needed in this portal for AI Search integration—it works independently with your blob storage.

## Security Considerations

### For Production Deployment:

1. **Use Managed Identity** instead of storage account keys:
   ```typescript
   const blobServiceClient = new BlobServiceClient(
     `https://${process.env.AZURE_STORAGE_ACCOUNT_NAME}.blob.core.windows.net`,
     new DefaultAzureCredential()
   );
   ```

2. **Implement Access Control**:
   - Add role-based access control (RBAC) to restrict who can upload/delete
   - Validate admin credentials against a proper user database
   - Add audit logging for all document operations

3. **Enable Encryption**:
   - Use Azure Storage encryption at rest (enabled by default)
   - Enable HTTPS only (already configured in connection string)

4. **Set Container Permissions**:
   - Keep the container private (not public)
   - Use Shared Access Signatures (SAS) for temporary access if needed

## Testing the Integration

1. **Test Upload**:
   - Log in with any admin name and 6+ character password
   - Select a document and click "Upload Document"
   - Verify it appears in the "Uploaded Documents" list

2. **Test Listing**:
   - Navigate to the admin portal
   - Verify all documents from your blob container are displayed

3. **Test Delete**:
   - Click the trash icon next to a document
   - Verify it's removed from the list and blob storage

4. **Verify AI Search Integration**:
   - Check your Azure AI Search index in the Azure Portal
   - Confirm documents are being indexed
   - Test your RAG application's Q&A functionality

## Troubleshooting

### Connection Issues

**Error**: "Failed to list documents from Azure Blob Storage"

**Solutions**:
- Verify environment variables are set correctly
- Check storage account name spelling
- Confirm the access key is valid (not expired)
- Ensure the container name exists

### Upload Failures

**Error**: "Failed to upload document to Azure Blob Storage"

**Solutions**:
- Check file size (max 50MB in portal, but Azure supports larger)
- Verify storage account has available quota
- Ensure the container exists and is accessible
- Check network connectivity to Azure

### Missing Documents

**Error**: Documents uploaded but not appearing in list

**Solutions**:
- Refresh the page
- Check the correct container name is configured
- Verify documents are in the right container (not a different one)
- Check Azure Portal to confirm documents exist

## Next Steps

1. Implement the backend code changes above
2. Set your Azure credentials in environment variables
3. Test the upload/list/delete functionality
4. Verify documents appear in Azure AI Search index
5. Test your RAG application's Q&A with the uploaded documents
6. Deploy to production with proper security measures

## Support

For Azure-specific issues, refer to:
- [Azure Storage Blob documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Azure SDK for JavaScript](https://github.com/Azure/azure-sdk-for-js)
- [Azure AI Search documentation](https://learn.microsoft.com/en-us/azure/search/)
