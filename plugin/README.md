# MongoDB Plugin for Genkit

A comprehensive MongoDB plugin for the Genkit AI framework that provides vector search, text search, hybrid search, CRUD operations, and search index management capabilities.

## Features

- **Vector Search**: Semantic search using embeddings with MongoDB's vector search capabilities
- **Text Search**: Full-text search with fuzzy matching and synonyms support
- **Hybrid Search**: Combine vector and text search using MongoDB's `$rankFusion` for enhanced results
- **CRUD Operations**: Create, read, update, and delete documents by ID
- **Search Index Management**: Create, list, and drop search indexes
- **Batch Indexing**: Efficient document indexing with configurable batch sizes
- **Retry Logic**: Built-in retry mechanisms with configurable attempts, delays, and jitter
- **Flexible Field Configuration**: Customizable field names for data, metadata, and embeddings
- **Multiple Connection Support**: Configure multiple MongoDB connections with different settings
- **Multimodal Support**: Process images and documents with multimodal embeddings
- **Pipeline Support**: Custom aggregation pipelines for advanced querying

## Prerequisites

- MongoDB 6.0+ with Atlas Search or local search indexes
- Node.js 18+
- Genkit framework

## Installation

```bash
pnpm install
```

## Building the Plugin

This is a plugin library that can be built but not run directly. To build the plugin:

```bash
pnpm build
```

This will compile the TypeScript code and generate the distribution files.

**Note**: This plugin is designed to be used within applications that integrate with the Genkit framework. You cannot run this plugin directly - you need to use it in an application that imports and configures the plugin. eg: testapp

## Quick Start

### Basic Setup

```typescript
import { genkit } from "genkit";
import { mongodb } from "genkitx-mongodb";
import { googleAI } from "@genkit-ai/googleai";

const ai = genkit({
  plugins: [
    mongodb([
      {
        url: "mongodb://localhost:27017",
        indexer: {
          id: "indexer",
          retry: {
            retryAttempts: 3,
            baseDelay: 1000,
            jitterFactor: 0.1,
          },
        },
        retriever: {
          id: "retriever",
          retry: {
            retryAttempts: 2,
            baseDelay: 500,
          },
        },
        crudTools: {
          id: "crud",
        },
        searchIndexTools: {
          id: "search-index",
        },
      },
    ]),
  ],
});
```

### Indexing Documents

```typescript
import { Document } from "genkit";
import { mongoIndexerRef } from "genkitx-mongodb";

const documents = [
  Document.fromText("Sample document content", { id: "1", category: "sample" }),
  Document.fromText("Another document", { id: "2", category: "example" }),
];

await ai.index({
  indexer: mongoIndexerRef("indexer"),
  documents,
  options: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    embedder: googleAI.embedder("text-embedding-004"),
    embeddingField: "embedding",
    batchSize: 100,
    skipData: false, // Optional: Set to true to exclude original data from storage
    dataField: "data", // Optional: Custom field name for document data
    metadataField: "metadata", // Optional: Custom field name for metadata
    dataTypeField: "dataType", // Optional: Custom field name for data type
  },
});
```

### Vector Search

```typescript
import { mongoRetrieverRef } from "genkitx-mongodb";

const results = await ai.retrieve({
  retriever: mongoRetrieverRef("retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    embedder: googleAI.embedder("text-embedding-004"),
    vectorSearch: {
      index: "embedding_index",
      path: "embedding",
      exact: false,
      numCandidates: 100,
      limit: 10,
      filter: { category: "sample" },
    },
  },
});
```

### Text Search

```typescript
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    search: {
      index: "text_index",
      text: {
        path: "content",
        matchCriteria: "any",
        fuzzy: {
          maxEdits: 2,
          prefixLength: 0,
          maxExpansions: 50,
        },
      },
    },
    pipelines: [{ $limit: 10 }, { $sort: { score: -1 } }],
  },
});
```

### Hybrid Search

The plugin supports hybrid search using MongoDB's `$rankFusion` aggregation, which combines vector and text search results for enhanced retrieval:

```typescript
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    embedder: googleAI.embedder("text-embedding-004"),
    hybridSearch: {
      search: {
        index: "text_index",
        text: {
          path: "content",
          fuzzy: {
            maxEdits: 2,
            prefixLength: 0,
            maxExpansions: 50,
          },
        },
      },
      vectorSearch: {
        index: "embedding_index",
        path: "embedding",
        exact: false,
        numCandidates: 100,
        limit: 10,
      },
      combination: {
        weights: {
          vectorPipeline: 0.7, // Weight for vector search results
          fullTextPipeline: 0.3, // Weight for text search results
        },
      },
      scoreDetails: true, // Include detailed scoring information
    },
  },
});
```

### CRUD Operations

The plugin provides tools for basic CRUD operations by document ID:

```typescript
// Create a document
await ai.runTool({
  name: "mongodb/crud/create",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    document: { name: "John", age: 30 },
  },
});

// Read a document by ID
const result = await ai.runTool({
  name: "mongodb/crud/read",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    id: "507f1f77bcf86cd799439011",
  },
});

// Update a document by ID
await ai.runTool({
  name: "mongodb/crud/update",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    id: "507f1f77bcf86cd799439011",
    document: { age: 31 },
  },
});

// Delete a document by ID
await ai.runTool({
  name: "mongodb/crud/delete",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    id: "507f1f77bcf86cd799439011",
  },
});
```

### Search Index Management

```typescript
// Create a search index
await ai.runTool({
  name: "mongodb/search-index/create",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    indexName: "text_index",
    definition: {
      mappings: {
        dynamic: true,
        fields: {
          content: {
            type: "string",
            analyzer: "lucene.english",
          },
        },
      },
    },
  },
});

// List search indexes
const indexes = await ai.runTool({
  name: "mongodb/search-index/list",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
  },
});

// Drop a search index
await ai.runTool({
  name: "mongodb/search-index/drop",
  input: {
    dbName: "myDatabase",
    collectionName: "myCollection",
    indexName: "text_index",
  },
});
```

## Configuration

### Connection Configuration

```typescript
{
  url: string;                   // MongoDB connection string
  mongoClientOptions?: object;   // MongoDB client options
  indexer?: BaseDefinition;      // Indexer configuration
  retriever?: BaseDefinition;    // Retriever configuration
  crudTools?: BaseDefinition;    // CRUD tools configuration
  searchIndexTools?: BaseDefinition; // Search index tools configuration
}
```

### Multiple Connections

You can configure multiple MongoDB connections with different settings:

```typescript
mongodb([
  {
    url: "mongodb://primary:27017",
    indexer: {
      id: "primary-indexer",
      retry: {
        retryAttempts: 3,
        baseDelay: 1000,
      },
    },
    retriever: {
      id: "primary-retriever",
      retry: {
        retryAttempts: 2,
        baseDelay: 500,
      },
    },
    crudTools: { id: "primary-crud" },
    searchIndexTools: { id: "primary-search" },
  },
  {
    url: "mongodb://secondary:27017",
    indexer: {
      id: "secondary-indexer",
      retry: {
        retryAttempts: 5,
        baseDelay: 2000,
        jitterFactor: 0.2,
      },
    },
    retriever: { id: "secondary-retriever" },
    crudTools: { id: "secondary-crud" },
    searchIndexTools: { id: "secondary-search" },
  },
]);
```

### Base Definition Configuration

Each component (indexer, retriever, crudTools, searchIndexTools) uses a base definition:

```typescript
{
  id: string;                    // Unique identifier for the component
  retry?: RetryOptions;          // Optional retry options for this component
}
```

### Retry Options

Retry options can be configured for individual components:

```typescript
{
  retryAttempts?: number;        // Number of retry attempts (default: 0)
  baseDelay?: number;            // Base delay in milliseconds (default: 1000)
  jitterFactor?: number;         // Jitter factor for exponential backoff (default: 0.1)
}
```

## Advanced Usage

### Multimodal Document Processing

The plugin supports multimodal embeddings for processing images and documents:

```typescript
import { multimodalEmbedding001 } from "@genkit-ai/vertexai";

// Index images with multimodal embeddings
await ai.index({
  indexer: mongoIndexerRef("indexer"),
  documents: imageDocuments,
  options: {
    dbName: "myDatabase",
    collectionName: "imageCollection",
    embedder: multimodalEmbedding001,
    embeddingField: "imageEmbedding",
    dataField: "imageData",
    metadataField: "imageMetadata",
    dataTypeField: "imageType",
  },
});

// Retrieve similar images
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("retriever"),
  query: "find images similar to a cat",
  options: {
    dbName: "myDatabase",
    collectionName: "imageCollection",
    embedder: multimodalEmbedding001,
    vectorSearch: {
      index: "image_embedding_index",
      path: "imageEmbedding",
      numCandidates: 50,
      limit: 5,
    },
  },
});
```

### Custom Field Configuration

```typescript
// Use custom field names for different data types
await ai.index({
  indexer: mongoIndexerRef("indexer"),
  documents: imageDocuments,
  options: {
    dbName: "myDatabase",
    collectionName: "images",
    embedder: multimodalEmbedding001,
    embeddingField: "imageEmbedding",
    dataField: "imageData",
    metadataField: "imageMetadata",
    dataTypeField: "imageType",
    skipData: false, // Store original image data
  },
});

// Retrieve with custom field mapping
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("retriever"),
  query: "find similar images",
  options: {
    dbName: "myDatabase",
    collectionName: "images",
    embedder: multimodalEmbedding001,
    dataField: "imageData",
    metadataField: "imageMetadata",
    dataTypeField: "imageType",
    vectorSearch: {
      index: "image_vector_index",
      path: "imageEmbedding",
      numCandidates: 20,
      limit: 5,
    },
  },
});
```

### Tool References

The plugin provides helper functions to generate tool references:

```typescript
import {
  mongoCrudToolsRefArray,
  mongoSearchIndexToolsRefArray,
} from "genkitx-mongodb";

// Get all CRUD tool references for a connection
const crudTools = mongoCrudToolsRefArray("my-connection-id");
// Returns: ['mongodb/my-connection-id/create', 'mongodb/my-connection-id/read', ...]

// Get all search index tool references for a connection
const searchIndexTools = mongoSearchIndexToolsRefArray("my-connection-id");
// Returns: ['mongodb/my-connection-id/create', 'mongodb/my-connection-id/list', ...]
```

## Configuration Options

### Indexer Options

```typescript
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  embedder: EmbedderArgument;    // Embedder for generating vectors
  embedderOptions?: object;      // Optional embedder-specific options
  embeddingField?: string;       // Field name for embeddings (default: 'embedding')
  batchSize?: number;            // Batch size for indexing (default: 100)
  skipData?: boolean;            // Optional: Skip storing original data (default: false)
  dataField?: string;            // Field name for data (default: 'data')
  metadataField?: string;        // Field name for metadata (default: 'metadata')
  dataTypeField?: string;        // Field name for data type (default: 'dataType')
}
```

### Retriever Options

```typescript
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  // For vector search:
  embedder?: EmbedderArgument;   // Embedder for query vectorization
  embedderOptions?: object;      // Optional embedder-specific options
  vectorSearch?: {
    index: string;               // Vector search index name
    path: string;                // Field path for vectors
    exact?: boolean;             // Use exact search
    numCandidates?: number;      // Number of candidates (max: 10000)
    limit?: number;              // Result limit
    filter?: object;             // MongoDB filter
  };
  // For text search:
  search?: {
    index: string;               // Text search index name
    text: {
      path: string;              // Field path for text
      matchCriteria?: 'any' | 'all';
      fuzzy?: {
        maxEdits?: number;       // Maximum edit distance (1-2)
        prefixLength?: number;   // Prefix length
        maxExpansions?: number;  // Maximum expansions
      };
      score?: object;            // Score configuration
      synonyms?: string;         // Synonyms collection
    };
  };
  // For hybrid search:
  hybridSearch?: {
    search: TextSearchOptions;   // Text search configuration
    vectorSearch: VectorSearchOptions; // Vector search configuration
    combination?: {
      weights?: {
        vectorPipeline?: number; // Weight for vector results (0-1, default: 0.5)
        fullTextPipeline?: number; // Weight for text results (0-1, default: 0.5)
      };
    };
    scoreDetails?: boolean;      // Include score details (default: false)
  };
  pipelines?: array;             // Aggregation pipeline stages
}
```

### CRUD Tool Options

```typescript
// Create
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  document: object;              // Document to create
}

// Read
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  id: string;                    // Document ID (24-character hex string)
}

// Update
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  id: string;                    // Document ID (24-character hex string)
  document: object;              // Update document (use MongoDB operators like $set)
}

// Delete
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  id: string;                    // Document ID (24-character hex string)
}
```

### Search Index Tool Options

```typescript
// Create
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  indexName: string;             // Index name
  definition: object;            // Index definition
}

// List
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
}

// Drop
{
  dbName: string;                // Database name
  dbOptions?: object;            // Database options
  collectionName: string;        // Collection name
  collectionOptions?: object;    // Collection options
  indexName: string;             // Index name to drop
}
```

## License

Apache 2.0

This is an independent MongoDB plugin for Genkit. Please file issues and pull requests against this repository.

Usage information and reference details can be found in [Genkit documentation](https://genkit.dev/docs/get-started).
