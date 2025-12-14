# ChromaDB Setup Guide

This guide will help you properly connect ChromaDB to your CloneWriter application.

## Quick Start

### Option 1: Using Docker Compose (Recommended)

The easiest way to set up ChromaDB is using Docker Compose, which includes both ChromaDB and your application:

```bash
# Start ChromaDB and CloneWriter together
docker-compose up -d

# Check if services are running
docker-compose ps

# View logs
docker-compose logs -f chromadb
```

ChromaDB will be available at `http://localhost:8000` and your app at `http://localhost:3000`.

### Option 2: Local Development Setup

If you prefer to run ChromaDB separately for local development:

#### 1. Install ChromaDB

**Using Docker:**
```bash
docker pull chromadb/chroma
docker run -d \
  --name chromadb \
  -p 8000:8000 \
  -v chromadb_data:/chroma/chroma \
  -e IS_PERSISTENT=TRUE \
  -e ANONYMIZED_TELEMETRY=FALSE \
  chromadb/chroma
```

**Using Python (Alternative):**
```bash
pip install chromadb
chroma run --host localhost --port 8000 --path ./chroma_data
```

#### 2. Configure Environment Variables

Create a `.env` file in your project root (or copy from `.env.example`):

```env
# Vector Store Configuration
VECTOR_STORE_TYPE=chroma

# ChromaDB Configuration
CHROMA_URL=http://localhost:8000

# Ollama Configuration
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=llama3.2:3b
```

#### 3. Start Your Application

```bash
npm run dev
```

## Verification

### Check ChromaDB Status

1. **Via API:**
   ```bash
   curl http://localhost:8000/api/v1/heartbeat
   ```
   Should return: `{"nanosecond heartbeat": <timestamp>}`

2. **Via Application:**
   - Open `http://localhost:3000/api/status` in your browser
   - Check the `vectorStore` object - it should show:
     ```json
     {
       "running": true,
       "type": "chroma",
       "collections": 0
     }
     ```

### Test the Connection

1. Upload some documents via the UI
2. Check the browser console or server logs for:
   ```
   ChromaDB vector store initialized
   Created new ChromaDB collection: clonewriter
   Added X documents to ChromaDB
   ```

## Configuration Options

### Collection Name

By default, ChromaDB uses the collection name `clonewriter`. You can customize this:

```env
# In your .env file, you can set it via code:
# The collection name is set in the VectorStoreConfig
```

Or modify the default in `lib/vector-stores/implementations/chromadb.ts`:
```typescript
this.collectionName = config?.collectionName || 'clonewriter';
```

### ChromaDB URL

For different environments:

- **Local Development:** `http://localhost:8000`
- **Docker Compose:** `http://chromadb:8000` (internal Docker network)
- **Remote Server:** `http://your-server-ip:8000`

### Persistence

ChromaDB data is persisted in Docker volumes:
- **Docker Compose:** `chromadb_data` volume
- **Local:** `./chroma_data` directory (if using Python)

## Troubleshooting

### ChromaDB Not Starting

**Check if port 8000 is available:**
```bash
# Windows
netstat -ano | findstr :8000

# Linux/Mac
lsof -i :8000
```

**Check Docker logs:**
```bash
docker logs chromadb
```

### Connection Refused

1. **Verify ChromaDB is running:**
   ```bash
   docker ps | grep chroma
   # or
   curl http://localhost:8000/api/v1/heartbeat
   ```

2. **Check environment variables:**
   - Ensure `VECTOR_STORE_TYPE=chroma` is set
   - Verify `CHROMA_URL` matches your ChromaDB instance

3. **Check network connectivity:**
   - If using Docker Compose, ensure services are on the same network
   - For local development, use `http://localhost:8000`
   - For Docker, use `http://chromadb:8000`

### Health Check Failing

The application automatically falls back to file-based storage if ChromaDB is unavailable. Check logs for:
```
Vector store chroma health check failed, falling back to file-based store
```

To fix:
1. Ensure ChromaDB is running
2. Check the `CHROMA_URL` environment variable
3. Verify network connectivity

### Documents Not Being Added

1. **Check ChromaDB logs:**
   ```bash
   docker logs chromadb
   ```

2. **Verify collection exists:**
   ```bash
   curl http://localhost:8000/api/v1/collections/clonewriter
   ```

3. **Check application logs** for error messages

### Performance Issues

- **Large datasets:** ChromaDB handles large datasets well, but consider batching
- **Memory:** Ensure Docker has enough memory allocated (4GB+ recommended)
- **Network:** If ChromaDB is remote, network latency can affect performance

## Advanced Configuration

### Custom Embedding Function

By default, ChromaDB uses its built-in embedding function. The current implementation relies on this. If you need custom embeddings, you would need to:

1. Generate embeddings client-side using a library like `@xenova/transformers`
2. Pass embeddings to ChromaDB in the `addDocuments` method
3. Update the `queryDocuments` method to use embeddings

### Multiple Collections

To use multiple collections for different purposes:

```typescript
import { VectorStoreFactory } from '@/lib/vector-stores/factory';

const store1 = VectorStoreFactory.create('chroma', { 
  collectionName: 'collection1' 
});
const store2 = VectorStoreFactory.create('chroma', { 
  collectionName: 'collection2' 
});
```

### Production Deployment

For production:

1. **Use persistent volumes:**
   ```yaml
   volumes:
     - chromadb_data:/chroma/chroma
   ```

2. **Set proper resource limits:**
   ```yaml
   deploy:
     resources:
       limits:
         memory: 4G
   ```

3. **Use environment variables for configuration:**
   ```env
   CHROMA_URL=http://chromadb:8000
   VECTOR_STORE_TYPE=chroma
   ```

4. **Enable authentication** (if ChromaDB supports it in your version)

## Migration from File-Based Storage

If you're currently using file-based storage and want to migrate to ChromaDB:

1. **Start ChromaDB:**
   ```bash
   docker-compose up -d chromadb
   ```

2. **Set environment variable:**
   ```env
   VECTOR_STORE_TYPE=chroma
   ```

3. **Re-upload your documents:**
   - The file-based data won't automatically migrate
   - Upload your original files again through the UI
   - ChromaDB will create embeddings and store them

## Additional Resources

- [ChromaDB Documentation](https://docs.trychroma.com/)
- [ChromaDB REST API](https://docs.trychroma.com/reference/rest-api)
- [Docker Hub - ChromaDB](https://hub.docker.com/r/chromadb/chroma)

## Support

If you encounter issues:
1. Check the troubleshooting section above
2. Review application logs: `docker-compose logs clonewriter`
3. Review ChromaDB logs: `docker-compose logs chromadb`
4. Check the `/api/status` endpoint for service health

