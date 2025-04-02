# [Arc (Rig)](https://github.com/0xPlaygrounds/rig) Framework Analysis
## Overview
Rig is a Rust library designed for building LLM-powered applications with a focus on ergonomics and modularity. It provides a comprehensive set of tools and abstractions for working with language models, vector stores, and AI agents.

Key Features:
- Modular architecture for flexible AI application development
- Strong type safety and error handling
- Efficient resource utilization and performance optimization
- Support for both synchronous and asynchronous operations
- Extensive tooling for RAG (Retrieval Augmented Generation) systems

## Core Components

### 1. LLM Integration
- **Model Support**:
  - Full support for LLM completion workflows
  - Comprehensive embedding capabilities
  - Streaming responses and batch processing

- **Provider Abstraction**:
  - Multiple provider support (OpenAI, Cohere, etc.)
  - Consistent API across providers
  - Easy provider switching and fallback

- **Integration Features**:
  ```rust
  // Example of provider setup
  let openai_client = Client::new(&openai_api_key);
  let agent = openai_client.agent("gpt-4")
      .preamble("System prompt here")
      .build();
  ```

### 2. Vector Store Support
- **Implementations**:
  - MongoDB for distributed storage
  - In-memory for fast local processing
  - Postgres for relational database integration
  - Custom implementations possible

- **Common Interface**:
  ```rust
  pub trait VectorStore {
      type Error;
      type Document;
      
      async fn insert(&self, documents: Vec<Document>) -> Result<(), Error>;
      async fn search(&self, query: Vector, limit: usize) -> Result<Vec<Document>, Error>;
  }
  ```

- **Features**:
  - Efficient vector similarity search
  - Batch operations support
  - Flexible document schema
  - Index management

### 3. Agent System
- **Agent Types**:
  - Simple completion agents
  - RAG-enabled agents
  - Multi-tool agents
  - Pipeline agents

- **Configuration**:
  ```rust
  let agent = openai_client.agent("gpt-4")
      .preamble("System prompt")
      .context("Background info")
      .tool(Calculator)
      .dynamic_context(5, vector_store)
      .build();
  ```

- **Features**:
  - Tool integration
  - Context management
  - Error handling
  - Response streaming

## Pipeline System

### 1. What is a Pipeline?
A pipeline in Rig is a flexible API for defining a sequence of operations that may or may not use AI components. It's inspired by orchestration pipelines like Airflow and Dagster, but implemented with idiomatic Rust patterns and AI-specific operations.

### 2. Core Pipeline Concepts
- **Operations (Ops)**
- **Sequential Processing**
- **Parallel Execution**
- **Error Handling**
- **Type Safety**

### 3. Pipeline Features
```rust
// Sequential Operations
let chain = pipeline::new()
    .map(|input| format!("Query: {}", input))
    .prompt(agent1)
    .map(|response| process_response(response));

// Parallel Operations
let chain = pipeline::new()
    .chain(parallel!(
        passthrough(),
        lookup(vector_store, 1),
    ));

// Error Handling
let chain = pipeline::new()
    .map(|x| if valid(x) { Ok(x) } else { Err("Invalid") })
    .map_err(|err| format!("Error: {}", err));
```

## Dynamic Context and Tools

### 1. Dynamic Context
Dynamic context enables automatic retrieval of relevant information based on the current query. This is particularly useful for RAG systems and knowledge-based applications.

#### Basic Setup
```rust
// Dynamic Context Setup
let agent = openai_client.agent("gpt-4")
    .dynamic_context(5, vector_store_index)  // Top 5 relevant docs
    .build();
```

#### Advanced Context Management
```rust
// Custom context selection with filtering
let agent = openai_client.agent("gpt-4")
    .dynamic_context_with_options(
        DynamicContextOptions {
            max_documents: 5,
            min_similarity: 0.7,
            filter: |doc| doc.metadata.date > one_year_ago(),
        },
        vector_store_index
    )
    .build();

// Multiple context sources
let agent = openai_client.agent("gpt-4")
    .dynamic_context(3, primary_index)
    .dynamic_context(2, secondary_index)
    .build();

// Context with metadata handling
let rag_pipeline = pipeline::new()
    .chain(parallel!(
        passthrough(),
        lookup_with_metadata(vector_store, 1),
    ))
    .map(|(query, docs)| {
        let context = docs.iter()
            .map(|doc| format!("Source: {}\nDate: {}\nContent: {}", 
                doc.metadata.source,
                doc.metadata.date,
                doc.content))
            .collect::<Vec<_>>()
            .join("\n\n");
        format!("Query: {}\n\nAvailable Context:\n{}", query, context)
    })
    .prompt(agent);
```

#### RAG Implementation Examples
```rust
// Basic RAG with dynamic context
let rag_agent = pipeline::new()
    .map(|query: String| {
        // Preprocess query
        format!("Answer the following question: {}", query)
    })
    .chain(parallel!(
        passthrough(),
        lookup(vector_store, 1),
    ))
    .map(|(query, context)| combine_query_and_context(query, context))
    .prompt(agent);

// Advanced RAG with multiple steps
let advanced_rag = pipeline::new()
    // Step 1: Query Analysis
    .map(|query: String| analyze_query(query))
    .prompt(query_analyzer_agent)
    // Step 2: Context Retrieval
    .chain(parallel!(
        passthrough(),
        lookup_with_filters(vector_store, 
            SearchOptions {
                limit: 5,
                min_score: 0.7,
                filters: vec!["category = 'technical'"]
            }
        ),
    ))
    // Step 3: Context Processing
    .map(|(query, docs)| process_and_rank_documents(query, docs))
    // Step 4: Response Generation
    .prompt(response_agent);
```

### 2. Dynamic Tools
Tools that are selected based on semantic relevance to the current task. This allows for efficient tool management and context window usage.

#### Basic Setup
```rust
// Dynamic Tools Setup
let toolset = ToolSet::builder()
    .dynamic_tool(Calculator)
    .dynamic_tool(WeatherAPI)
    .dynamic_tool(Calendar)
    .build();

let agent = openai_client.agent("gpt-4")
    .dynamic_tools(2, tool_index, toolset)
    .build();
```

#### Advanced Tool Management
```rust
// Tool with custom embedding
#[derive(Tool, ToolEmbedding)]
struct Calculator {
    #[tool_description]
    description: &'static str = "Performs basic arithmetic operations";
    
    #[tool_examples]
    examples: Vec<String> = vec![
        "Calculate 2 + 2",
        "Multiply 5 by 3",
    ];
}

// Tool with state
#[derive(Tool, ToolEmbedding)]
struct WeatherAPI {
    client: ApiClient,
    cache: Cache<String, WeatherData>,
    
    #[tool_description]
    description: &'static str = "Gets weather information for a location";
}

// Complex toolset with categories
let toolset = ToolSet::builder()
    // Math tools
    .category("math")
        .dynamic_tool(Calculator)
        .dynamic_tool(UnitConverter)
        .dynamic_tool(GraphPlotter)
    .end()
    // API tools
    .category("api")
        .dynamic_tool(WeatherAPI)
        .dynamic_tool(StockAPI)
        .dynamic_tool(NewsAPI)
    .end()
    // Utility tools
    .category("utility")
        .dynamic_tool(Calendar)
        .dynamic_tool(EmailClient)
        .dynamic_tool(FileManager)
    .end()
    .build();

// Agent with advanced tool configuration
let agent = openai_client.agent("gpt-4")
    .dynamic_tools_with_options(
        DynamicToolOptions {
            max_tools: 3,
            min_relevance: 0.6,
            category_limits: HashMap::from([
                ("math", 1),
                ("api", 1),
                ("utility", 1),
            ]),
        },
        tool_index,
        toolset
    )
    .build();
```

#### Implementation Examples
```rust
// Multi-tool pipeline
let multi_tool_pipeline = pipeline::new()
    // Analyze request
    .map(|request: String| analyze_request_requirements(request))
    // Select and prepare tools
    .then(|requirements| async {
        let selected_tools = toolset.select_tools(requirements, 3);
        prepare_tools_for_execution(selected_tools)
    })
    // Execute tools in parallel if possible
    .then(|tools| async {
        parallel!(
            tools.iter()
                .map(|tool| tool.execute())
                .collect()
        )
    })
    // Process results
    .map(|results| combine_tool_results(results));

// Tool with error handling and retries
let robust_tool_pipeline = pipeline::new()
    .map(|input| validate_input(input))
    .and_then(|validated| async {
        retry_with_backoff(3, || async {
            execute_tool_with_timeout(validated, Duration::from_secs(5))
        }).await
    })
    .map_err(|err| handle_tool_error(err))
    .or_else(|err| async {
        fallback_tool_execution(err)
    });
```

### 3. Comparison
| Feature | Static | Dynamic |
|---------|--------|---------|
| Context | Fixed, always included | Query-dependent |
| Tools | All available | Relevance-based |
| Latency | Lower | Higher (search overhead) |
| Memory | Higher (all loaded) | Lower (selective) |
| Use Case | Small, focused tasks | Large, diverse tasks |
| Flexibility | Limited | High adaptability |
| Setup Complexity | Simple | More complex |
| Maintenance | Easier | Requires tuning |

### 4. Best Practices for Dynamic Systems

#### Context Management
```rust
// Implement context refresh strategy
let agent = openai_client.agent("gpt-4")
    .dynamic_context_with_refresh(
        RefreshStrategy {
            max_age: Duration::from_hours(24),
            refresh_fn: |doc| async {
                if doc.is_stale() {
                    update_document_content(doc).await
                } else {
                    doc
                }
            }
        },
        vector_store_index
    )
    .build();
```

#### Tool Management
```rust
// Implement tool monitoring and metrics
let monitored_toolset = ToolSet::builder()
    .with_metrics(
        MetricsConfig {
            latency_threshold: Duration::from_millis(100),
            error_threshold: 0.01,
            usage_tracking: true,
        }
    )
    .dynamic_tool(Calculator)
    .dynamic_tool(WeatherAPI)
    .build();

// Tool result caching
let cached_toolset = ToolSet::builder()
    .with_cache(
        CacheConfig {
            max_size: 1000,
            ttl: Duration::from_minutes(30),
            strategy: CacheStrategy::LRU,
        }
    )
    .dynamic_tool(Calculator)
    .dynamic_tool(WeatherAPI)
    .build();
```

## Best Practices

### 1. Tool Management
- **Selection Strategy**:
  - Use static tools for small, focused agents
  - Use dynamic tools for diverse capabilities
  - Consider tool dependencies

- **Documentation**:
  - Clear tool descriptions
  - Example inputs/outputs
  - Error scenarios

- **Performance**:
  - Tool result caching
  - Parallel tool execution
  - Timeout handling

### 2. Context Handling
- **Dynamic Context**:
  - Appropriate sample sizes
  - Relevance thresholds
  - Context window monitoring

- **Vector Store**:
  - Index optimization
  - Regular maintenance
  - Backup strategies

- **Performance**:
  - Caching strategies
  - Batch processing
  - Resource monitoring

### 3. Error Handling
- Use `TryOp` for fallible operations
- Implement recovery strategies
- Provide meaningful error messages

### 4. Performance Optimization
- Parallel operations where possible
- Efficient resource utilization
- Batch processing for multiple inputs

### 5. Code Organization
- Modular pipeline design
- Clear documentation
- Consistent naming conventions 

## Integration Examples

### 1. Basic Agent Setup
```rust
use rig::{prelude::*, providers::openai::Client};

// Step 1: Initialize the OpenAI client
let openai_client = Client::new(&std::env::var("OPENAI_API_KEY")?);

// Step 2: Create a basic agent
let basic_agent = openai_client.agent("gpt-4")
    .preamble("You are a helpful assistant.")
    .build();

// Step 3: Use the agent
let response = basic_agent
    .prompt("What is the capital of France?")
    .await?;
```

### 2. RAG System Integration
```rust
// Step 1: Set up vector store
let vector_store = MongoVectorStore::new(
    "mongodb://localhost:27017",
    "knowledge_base",
    "documents"
).await?;

// Step 2: Create embeddings for documents
let embedding_model = openai_client.embeddings("text-embedding-ada-002");
let documents = vec![
    Document {
        content: "Paris is the capital of France".to_string(),
        metadata: json!({ "category": "geography" }),
    },
    // ... more documents
];

let embeddings = embedding_model
    .create_embeddings(documents)
    .await?;

// Step 3: Insert documents into vector store
vector_store.insert_batch(embeddings).await?;

// Step 4: Create vector store index
let index = vector_store.index(embedding_model);

// Step 5: Create RAG agent
let rag_agent = openai_client.agent("gpt-4")
    .preamble("You are a knowledgeable assistant. Use the provided context to answer questions accurately.")
    .dynamic_context_with_options(
        DynamicContextOptions {
            max_documents: 5,
            min_similarity: 0.7,
            filter: |doc| doc.metadata["category"] == "geography",
        },
        index
    )
    .build();

// Step 6: Create RAG pipeline
let rag_pipeline = pipeline::new()
    // Preprocess query
    .map(|query: String| {
        format!("Answer based on the following question: {}", query)
    })
    // Retrieve context and process
    .chain(parallel!(
        passthrough(),
        lookup_with_metadata(vector_store, 
            SearchOptions {
                limit: 3,
                min_score: 0.7,
                filters: vec!["category = 'geography'"]
            }
        ),
    ))
    // Format context and query
    .map(|(query, docs)| format_context_and_query(query, docs))
    // Generate response
    .prompt(rag_agent);
```

### 3. Advanced Multi-Tool Agent
```rust
// Step 1: Define custom tools
#[derive(Tool, ToolEmbedding)]
struct Calculator {
    #[tool_description]
    description: &'static str = "Performs mathematical calculations";
    
    #[tool_examples]
    examples: Vec<String> = vec![
        "Calculate 2 + 2",
        "What is 15% of 200?",
    ];

    #[tool_schema]
    schema: Schema = json!({
        "type": "object",
        "properties": {
            "operation": {"type": "string"},
            "numbers": {"type": "array", "items": {"type": "number"}}
        }
    });
}

#[derive(Tool, ToolEmbedding)]
struct WeatherAPI {
    client: ApiClient,
    cache: Cache<String, WeatherData>,
    
    #[tool_description]
    description: &'static str = "Gets weather information";
}

// Step 2: Create tool categories
let toolset = ToolSet::builder()
    .with_metrics(MetricsConfig {
        latency_threshold: Duration::from_millis(100),
        error_threshold: 0.01,
        usage_tracking: true,
    })
    .with_cache(CacheConfig {
        max_size: 1000,
        ttl: Duration::from_minutes(30),
        strategy: CacheStrategy::LRU,
    })
    // Math tools
    .category("math")
        .dynamic_tool(Calculator)
        .dynamic_tool(UnitConverter)
    .end()
    // API tools
    .category("api")
        .dynamic_tool(WeatherAPI)
        .dynamic_tool(StockAPI)
    .end()
    .build();

// Step 3: Create tool embeddings and index
let tool_embeddings = embedding_model
    .create_embeddings(toolset.get_descriptions())
    .await?;

let tool_store = InMemoryVectorStore::new();
tool_store.insert_batch(tool_embeddings).await?;
let tool_index = tool_store.index(embedding_model);

// Step 4: Create advanced agent with tools and RAG
let advanced_agent = openai_client.agent("gpt-4")
    // System setup
    .preamble("You are an advanced assistant with multiple capabilities.")
    .temperature(0.7)
    .max_tokens(2000)
    
    // Dynamic context configuration
    .dynamic_context_with_options(
        DynamicContextOptions {
            max_documents: 5,
            min_similarity: 0.7,
            filter: |doc| doc.metadata.date > one_year_ago(),
        },
        knowledge_index
    )
    
    // Dynamic tools configuration
    .dynamic_tools_with_options(
        DynamicToolOptions {
            max_tools: 3,
            min_relevance: 0.6,
            category_limits: HashMap::from([
                ("math", 1),
                ("api", 1),
            ]),
            timeout: Duration::from_secs(5),
        },
        tool_index,
        toolset
    )
    .build();

// Step 5: Create advanced pipeline
let advanced_pipeline = pipeline::new()
    // Input validation and preprocessing
    .map(|input: String| validate_and_preprocess(input))
    .map_err(|err| handle_preprocessing_error(err))
    
    // Parallel context and tool selection
    .chain(parallel!(
        // Query analysis
        passthrough()
            .then(|query| analyze_query_intent(query))
            .prompt(intent_analyzer_agent),
            
        // Context retrieval
        lookup_with_filters(knowledge_store, 
            SearchOptions {
                limit: 5,
                min_score: 0.7,
                filters: vec!["date > '2023-01-01'"]
            }
        ),
        
        // Tool selection
        select_relevant_tools(tool_index, 3)
    ))
    
    // Process and combine results
    .map(|(intent, context, tools)| {
        format_prompt_with_context_and_tools(intent, context, tools)
    })
    
    // Generate response with retry logic
    .then(|prompt| async {
        retry_with_backoff(3, || async {
            advanced_agent
                .prompt_with_timeout(prompt, Duration::from_secs(10))
                .await
        }).await
    })
    
    // Post-process response
    .map(|response| format_and_validate_response(response))
    
    // Error handling
    .map_err(|err| handle_pipeline_error(err))
    .or_else(|err| async {
        fallback_response_generation(err)
    });

// Step 6: Execute pipeline with monitoring
let response = advanced_pipeline
    .with_monitoring(
        MonitoringConfig {
            latency_tracking: true,
            error_tracking: true,
            usage_metrics: true,
        }
    )
    .call("What's the weather like in Paris and calculate 15% tax on the average temperature?")
    .await?;
```

### 4. Production Configuration Example
```rust
// Step 1: Configure environment
let config = Config {
    // API configurations
    openai: OpenAIConfig {
        api_key: std::env::var("OPENAI_API_KEY")?,
        organization: std::env::var("OPENAI_ORG")?,
        timeout: Duration::from_secs(30),
        retry_config: RetryConfig {
            max_retries: 3,
            initial_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(5),
        },
    },
    
    // Vector store configuration
    vector_store: VectorStoreConfig {
        mongodb_uri: std::env::var("MONGODB_URI")?,
        cache_size: 10000,
        max_connections: 100,
    },
    
    // Monitoring and logging
    monitoring: MonitoringConfig {
        prometheus_endpoint: "localhost:9090".to_string(),
        log_level: "info",
        trace_sampling_rate: 0.1,
    },
};

// Step 2: Initialize services
let services = Services::new(config)
    .with_openai()
    .with_mongodb()
    .with_prometheus()
    .with_tracing()
    .build()
    .await?;

// Step 3: Create production pipeline
let production_pipeline = pipeline::new()
    .with_retry(RetryConfig::default())
    .with_timeout(Duration::from_secs(30))
    .with_monitoring(services.monitoring)
    .with_tracing(services.tracer)
    // ... rest of the pipeline configuration
    .build();

// Step 4: Start server
let server = Server::new(production_pipeline)
    .with_rate_limiting(
        RateLimitConfig {
            requests_per_second: 100,
            burst_size: 50,
        }
    )
    .with_health_checks()
    .with_metrics()
    .build();

server.run().await?;
``` 