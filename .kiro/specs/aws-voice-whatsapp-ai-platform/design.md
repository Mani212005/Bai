# Design Document: AWS-Deployed Voice + WhatsApp AI Platform

## Overview

This design document specifies the architecture and implementation approach for a production-grade, cloud-native AI communication platform deployed on AWS. The system enables multi-channel interactions through voice calls (Twilio) and WhatsApp messaging, featuring real-time WebSocket support, multi-agent orchestration powered by Amazon Bedrock and Amazon Q, and comprehensive observability.

### Architecture Philosophy

The platform follows cloud-native principles:
- **Containerized microservices** on ECS Fargate for scalability and isolation
- **Managed AWS services** for reduced operational overhead
- **Event-driven architecture** for real-time communication
- **Infrastructure as Code** for reproducible deployments
- **Defense in depth** security model
- **Observability-first** design for production operations
- **AI-native** design leveraging Amazon Bedrock and Amazon Q

### High-Level Architecture

```
Internet
   ↓
Route 53 (DNS)
   ↓
ACM (SSL/TLS)
   ↓
Application Load Balancer (ALB)
   ├─→ HTTPS Traffic (Port 443)
   └─→ WebSocket Connections (Upgrade from HTTP)
   ↓
ECS Fargate Cluster (2-10 tasks)
   ├─→ FastAPI Backend Containers
   ↓
   ├─→ Amazon Bedrock (Foundation Models: Claude, Titan, Llama)
   ├─→ Amazon Q (Intelligent Agent Builder)
   ├─→ RDS PostgreSQL (Call logs, transcripts, metadata)
   ├─→ ElastiCache Redis (Sessions, cache, rate limits)
   ├─→ S3 (Audio recordings, debug exports)
   ├─→ Secrets Manager (Credentials)
   └─→ CloudWatch (Logs, metrics, alarms)
```

### Technology Stack

**Backend Framework:**
- FastAPI (Python 3.11+) for high-performance async web framework
- Uvicorn ASGI server with WebSocket support
- SQLAlchemy ORM for PostgreSQL interactions
- Redis-py for cache operations with connection pooling
- Boto3 AWS SDK for Bedrock, Q, S3, and Secrets Manager
- Pydantic for data validation and settings management

**AI Services:**
- Amazon Bedrock for foundation model access (Claude 3, Titan, Llama 2/3)
- Amazon Q for intelligent agent builder and conversational AI
- Bedrock Runtime API for model invocation and streaming responses
- Amazon Q Agent Builder for custom agent creation and management

**Infrastructure:**
- ECS Fargate for serverless container compute
- Application Load Balancer for Layer 7 load balancing with WebSocket support
- RDS PostgreSQL 15 for relational data storage
- ElastiCache Redis 7 for in-memory caching
- S3 for object storage
- CloudWatch for monitoring and logging
- Secrets Manager for credential management
- IAM for access control

**External Services:**
- Twilio Voice API for voice call handling
- Twilio Media Streams for real-time audio streaming
- WhatsApp Business API for messaging integration


## Architecture

### System Components

#### 1. API Gateway Layer (Application Load Balancer)

The ALB serves as the entry point for all traffic, providing:

**Responsibilities:**
- SSL/TLS termination using ACM certificates
- HTTP to HTTPS redirection for security
- WebSocket connection upgrade handling for real-time voice streams
- Health check routing to backend instances
- Path-based request routing
- Connection draining during rolling deployments

**Configuration Approach:**
- HTTPS listener on port 443 with ACM certificate
- HTTP listener on port 80 redirecting to 443
- Target group pointing to ECS Fargate tasks on port 8000
- Health check endpoint `/health` with 30-second intervals
- 30-second deregistration delay for graceful shutdown
- Stateless design without sticky sessions

#### 2. Compute Layer (ECS Fargate)

Containerized FastAPI application running on serverless compute:

**Container Design:**
- Python 3.11 slim base image for minimal footprint
- 1 vCPU and 2 GB memory per task
- Port 8000 exposed for HTTP/WebSocket traffic
- Health check endpoint for ALB integration
- Environment variables loaded from Secrets Manager at startup

**Auto-scaling Strategy:**
- Minimum 2 tasks across 2 availability zones for high availability
- Maximum 10 tasks for cost control
- Target CPU utilization of 70% triggers scaling
- 60-second scale-up cooldown for rapid response
- 300-second scale-down cooldown to prevent flapping

**Deployment Strategy:**
- Rolling deployment updating one task at a time
- New tasks must pass health checks before receiving traffic
- Old tasks drain connections for 30 seconds before termination
- Automatic rollback if new tasks fail health checks 3 times

#### 3. AI Intelligence Layer

**Amazon Bedrock Integration:**

The platform leverages Amazon Bedrock for foundation model access with a multi-model strategy:

**Model Selection by Use Case:**
- **Claude 3 Sonnet**: Complex reasoning, multi-turn conversations, nuanced understanding
- **Claude 3 Haiku**: Fast responses, simple queries, high-throughput scenarios
- **Titan Text Express**: Cost-effective general purpose, summarization, classification
- **Llama 3 70B**: Open-source option, customizable for specific domains

**Bedrock Service Design:**
- Async invocation using Boto3 Bedrock Runtime client
- Retry logic with exponential backoff (3 attempts, adaptive mode)
- Support for both synchronous and streaming responses
- Model-specific request/response formatting
- Conversation history management (last 10 messages)
- System prompt customization per agent type

**Amazon Q Integration:**

Amazon Q provides intelligent agent building capabilities:

**Agent Types:**
- **Greeting Agent**: Handles initial user interactions and welcomes
- **Intent Classifier Agent**: Determines user intent and routes appropriately
- **Information Retrieval Agent**: Answers questions using knowledge bases
- **Task Executor Agent**: Performs actions based on user requests
- **Fallback Agent**: Handles unclear or out-of-scope requests

**Amazon Q Service Design:**
- Agent creation with custom instructions and knowledge bases
- Session-based conversation management
- Citation tracking for knowledge base responses
- Trace logging for debugging and optimization
- Async invocation for non-blocking operations

**Bedrock vs Amazon Q Decision Matrix:**
- Use Amazon Q when knowledge base integration is required
- Use Bedrock for custom prompt engineering and fine-tuned responses
- Use Bedrock streaming for real-time voice interactions
- Use Amazon Q for complex multi-step reasoning tasks
- Use Claude 3 Haiku for intent classification (fast and cheap)

#### 4. Data Layer

**RDS PostgreSQL Design:**

**Instance Configuration:**
- PostgreSQL 15.x engine
- db.t4g.medium instance (2 vCPU, 4 GB RAM)
- 100 GB gp3 storage with autoscaling to 500 GB
- Multi-AZ deployment for automatic failover
- 7-day automated backup retention
- Encryption at rest (KMS) and in transit (SSL)

**Schema Design:**
- Users table: Phone numbers, timestamps
- Conversations table: User linkage, channel type, status, metadata (JSONB)
- Messages table: Conversation linkage, direction, content, agent ID, timestamps
- Call logs table: Twilio call SID, duration, recording references, transcripts
- Agent metadata table: Agent configuration, types, active status

**Indexing Strategy:**
- Primary keys on all ID fields (UUID)
- Unique index on phone numbers
- Composite index on (conversation_id, timestamp) for message queries
- Index on agent_id for agent performance analysis
- Index on channel_type for channel-specific queries

**ElastiCache Redis Design:**

**Cluster Configuration:**
- Redis 7.x engine
- cache.t4g.medium nodes (2 vCPU, 3.09 GB RAM)
- Cluster mode enabled with 2 shards
- 1 replica per shard for automatic failover
- Encryption at rest and in transit

**Key Patterns and TTLs:**
- `session:{user_id}:{channel}` → Conversation context (30 min TTL)
- `rate_limit:{user_id}:requests` → Request counter (1 min TTL)
- `rate_limit:{user_id}:connections` → WebSocket counter (1 hour TTL)
- `agent_state:{conversation_id}` → Active agent state (30 min TTL)
- `webhook_nonce:{nonce}` → Replay prevention (5 min TTL)
- `bedrock_response:{hash}` → Cached AI responses (30 min TTL)

#### 5. Storage Layer (S3)

**Bucket Organization:**
- `audio-recordings/{year}/{month}/{day}/{call_sid}.wav` - Voice call recordings
- `debug-exports/{year}/{month}/{day}/{conversation_id}.json` - Debug traces
- `conversation-traces/{year}/{month}/{day}/{conversation_id}_trace.json` - Full traces

**Lifecycle Management:**
- Audio recordings transition to Glacier after 90 days
- Debug exports deleted after 30 days
- Conversation traces transition to Glacier after 180 days

**Security Configuration:**
- Versioning enabled for audit trail
- SSE-S3 encryption (AES-256)
- Public access blocked
- CORS configured for frontend access if needed

#### 6. Security Layer

**Secrets Manager Organization:**
- `/voice-whatsapp-platform/prod/database` - RDS credentials
- `/voice-whatsapp-platform/prod/redis` - ElastiCache auth token
- `/voice-whatsapp-platform/prod/twilio` - Twilio API credentials
- `/voice-whatsapp-platform/prod/whatsapp` - WhatsApp API credentials
- `/voice-whatsapp-platform/prod/bedrock` - Bedrock configuration (if needed)
- `/voice-whatsapp-platform/prod/amazon-q` - Amazon Q agent IDs

**IAM Role Design:**

ECS Task Execution Role (for container management):
- Pull images from ECR
- Write logs to CloudWatch
- Read secrets from Secrets Manager

ECS Task Role (for application operations):
- Read/write to S3 bucket
- Invoke Amazon Bedrock models
- Invoke Amazon Q agents
- Publish metrics to CloudWatch
- Read secrets from Secrets Manager
- Network access to RDS and ElastiCache (via security groups)

**Security Group Architecture:**
- ALB SG: Inbound 443/80 from internet, outbound 8000 to ECS
- ECS SG: Inbound 8000 from ALB, outbound 443 for AWS APIs, 5432 to RDS, 6379 to Redis
- RDS SG: Inbound 5432 from ECS only
- ElastiCache SG: Inbound 6379 from ECS only


## Components and Interfaces

### 1. Channel Abstraction Layer

**Design Purpose:**
Provides a unified interface for handling different communication channels (Voice and WhatsApp), enabling easy addition of new channels without modifying core business logic.

**Core Abstractions:**
- **ChannelAdapter Interface**: Defines methods for webhook validation, message receiving, message sending, and status tracking
- **Message Normalization**: Converts channel-specific formats into a common Message structure
- **Channel-Specific Validation**: Handles signature verification for Twilio and WhatsApp webhooks

**TwilioVoiceAdapter Responsibilities:**
- Validate X-Twilio-Signature header using HMAC-SHA1
- Generate TwiML responses for call control
- Manage WebSocket connections for Media Streams
- Convert audio streams to text (speech-to-text integration)
- Convert AI responses to audio (text-to-speech integration)
- Handle call status callbacks

**WhatsAppAdapter Responsibilities:**
- Validate WhatsApp webhook signatures
- Parse WhatsApp message formats (text, media, interactive buttons)
- Handle message status callbacks (sent, delivered, read, failed)
- Download media from WhatsApp CDN
- Format outbound messages according to WhatsApp API requirements
- Handle message templates for business messaging

**Message Structure:**
All messages normalized to include: ID, conversation ID, user ID, channel type, direction (inbound/outbound), content, timestamp, and metadata dictionary for channel-specific data.

### 2. Multi-Agent Orchestrator

**Design Purpose:**
Routes incoming messages to appropriate AI agents powered by Amazon Bedrock or Amazon Q, maintaining conversation coherence across agent transitions.

**Agent Architecture:**

**Agent Types:**
1. **Greeting Agent** (Amazon Q): Welcomes users, provides initial guidance
2. **Intent Classifier Agent** (Bedrock Claude 3 Haiku): Fast intent detection and routing
3. **Information Retrieval Agent** (Amazon Q with knowledge base): Answers factual questions
4. **Task Executor Agent** (Bedrock Claude 3 Sonnet): Performs complex tasks
5. **Fallback Agent** (Bedrock Claude 3 Haiku): Handles unclear requests gracefully

**Agent Selection Strategy:**
1. Check if current agent can continue (confidence > 0.7)
2. If not, query all agents for confidence scores using Claude 3 Haiku
3. Select agent with highest confidence score
4. If all scores < 0.5, route to fallback agent
5. Cache agent selection in Redis for 30 minutes

**Conversation Context Management:**
- Store last 10 messages in context
- Include current agent ID
- Track conversation metadata (start time, channel, user preferences)
- Cache in Redis with 30-minute TTL
- Fall back to PostgreSQL if cache miss

**Bedrock Integration Approach:**
- Use async Boto3 client for non-blocking operations
- Build model-specific request bodies (Claude, Titan, Llama formats differ)
- Include system prompts tailored to agent type
- Pass conversation history for context-aware responses
- Configure temperature and max_tokens per agent type
- Parse model-specific response formats

**Amazon Q Integration Approach:**
- Create agents with custom instructions and knowledge bases
- Use session IDs tied to conversation IDs
- Track citations for knowledge base responses
- Log trace data for debugging
- Handle multi-step reasoning tasks

**Response Caching Strategy:**
- Hash (prompt + conversation_history + model_id) as cache key
- Store responses in Redis with 30-minute TTL
- Only cache for deterministic queries (temperature = 0)
- Invalidate cache on agent configuration changes

### 3. WebSocket Manager

**Design Purpose:**
Handles real-time bidirectional communication for Twilio Media Streams, enabling low-latency voice interactions.

**Connection Management:**
- Accept WebSocket connections with call SID identification
- Maintain connection registry with thread-safe locking
- Track connection metadata (user ID, call SID, timestamps)
- Implement connection limits per user (max 3 concurrent)
- Handle graceful disconnection and cleanup

**Audio Processing Pipeline:**
1. Receive base64-encoded audio chunks from Twilio
2. Accumulate chunks in buffer (~1 second of audio)
3. Convert audio to text using speech-to-text service
4. Create normalized Message and route to orchestrator
5. Receive AI response from orchestrator
6. Convert response text to audio using text-to-speech
7. Send base64-encoded audio back through WebSocket

**Streaming Response Handling:**
- Use Bedrock streaming API for real-time responses
- Stream audio chunks as they're generated
- Reduce perceived latency for voice interactions
- Handle stream interruptions gracefully

**Error Handling:**
- Detect WebSocket disconnections
- Clean up connection state
- Persist partial conversation data
- Log disconnection reasons for debugging

### 4. Database Service

**Design Purpose:**
Manages all PostgreSQL interactions with connection pooling, error handling, and query optimization.

**Connection Pooling:**
- Pool size: 20 connections per backend instance
- No overflow connections (fail fast on exhaustion)
- Pre-ping enabled to detect stale connections
- Async SQLAlchemy engine for non-blocking queries

**Core Operations:**
- Create/retrieve conversations
- Save messages with timestamps
- Store call logs with audio references
- Query conversation history (paginated)
- Update agent metadata
- Retrieve agent configurations

**Query Optimization:**
- Use indexes for common query patterns
- Limit conversation history queries to last 50 messages
- Implement pagination for large result sets
- Use JSONB queries for metadata filtering
- Prepared statements for repeated queries

**Transaction Management:**
- Use async context managers for automatic commit/rollback
- Implement retry logic for transient failures
- Handle connection timeouts gracefully
- Log slow queries for optimization

### 5. Cache Service

**Design Purpose:**
Manages all Redis operations for sessions, rate limiting, and caching with high performance.

**Connection Management:**
- Connection pool with 50 max connections
- Async Redis client for non-blocking operations
- Automatic reconnection on failures
- Cluster-aware client for sharded Redis

**Core Operations:**
- Get/set conversation context with TTL
- Increment rate limit counters with expiry
- Track WebSocket connection counts
- Cache Bedrock responses
- Store webhook nonces for replay prevention

**Rate Limiting Implementation:**
- Sliding window using sorted sets
- Per-user request counters (100/minute)
- Per-user connection counters (3 concurrent)
- IP-based blocking for repeated failures (15-minute blocks)
- Atomic increment operations for accuracy

**Cache Invalidation:**
- TTL-based expiration for most keys
- Manual invalidation on agent configuration changes
- Flush on deployment (optional, for testing)

### 6. Storage Service

**Design Purpose:**
Handles all S3 operations for audio recordings, debug exports, and conversation traces.

**Upload Strategy:**
- Async uploads using Boto3
- Multipart uploads for large files (>5MB)
- Server-side encryption (SSE-S3)
- Content-type headers for proper MIME types
- Unique keys using timestamps and IDs

**Download Strategy:**
- Pre-signed URLs for secure temporary access
- Expiration time: 1 hour for audio playback
- Range requests for partial downloads
- CloudFront integration for CDN (optional)

**Lifecycle Management:**
- Automated transitions to Glacier
- Deletion policies for temporary data
- Versioning for audit trail
- Cross-region replication (optional, for DR)


## Data Models

### Core Domain Models

**User:**
- Unique identifier (UUID)
- Phone number (unique, indexed)
- Creation and update timestamps

**Conversation:**
- Unique identifier (UUID)
- User reference (foreign key)
- Channel type (voice or whatsapp)
- Status (active, completed, failed)
- Start and end timestamps
- Metadata (JSONB for flexible attributes)

**Message:**
- Unique identifier (UUID)
- Conversation reference (foreign key, indexed)
- Direction (inbound or outbound)
- Content (text)
- Agent identifier (for tracking which agent responded)
- Timestamp (indexed for chronological queries)
- Metadata (JSONB for channel-specific data)

**CallLog:**
- Unique identifier (UUID)
- Conversation reference (foreign key)
- Twilio call SID (unique, indexed)
- Duration in seconds
- Recording URL (Twilio CDN)
- S3 audio key (for archived recordings)
- Transcript (full text)
- Creation timestamp

**AgentMetadata:**
- Unique identifier (UUID)
- Agent name (unique)
- Agent type (greeting, classifier, retrieval, executor, fallback)
- Configuration (JSONB for model settings, prompts, etc.)
- Active status (boolean)
- Creation timestamp

**ConversationContext (In-Memory):**
- Conversation identifier
- User identifier
- Channel type
- Message history (last 10 messages)
- Current agent identifier
- Metadata dictionary
- Creation and last activity timestamps

### API Request/Response Models

**WebhookValidationRequest:**
- Signature string
- Timestamp string
- Request body

**TwilioVoiceWebhook:**
- Call SID
- From/To phone numbers
- Call status
- Direction

**WhatsAppWebhook:**
- Object type
- Entry array (nested webhook data)

**SendMessageRequest:**
- Conversation ID
- Content
- Channel type

**SendMessageResponse:**
- Message ID
- Status
- Timestamp

**ConversationHistoryResponse:**
- Conversation ID
- Messages array
- Total count

**HealthCheckResponse:**
- Status string
- Timestamp
- Services dictionary (service name → status)

### Configuration Models

**DatabaseConfig:**
- Host, port, username, password, database name
- Pool size (default: 20)
- Connection string property

**RedisConfig:**
- Host, port, auth token
- SSL enabled flag
- Connection string property

**TwilioConfig:**
- Account SID, auth token, API key
- Phone number

**WhatsAppConfig:**
- API key, webhook verify token
- Phone number ID

**BedrockConfig:**
- Region
- Default model ID
- Temperature and max_tokens defaults

**AmazonQConfig:**
- Region
- Agent IDs for each agent type
- Knowledge base IDs

**AWSConfig:**
- Region
- S3 bucket name
- Secrets prefix

**AppConfig:**
- Environment (development, staging, production)
- Log level
- All service configurations


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: WebSocket Connection Establishment

*For any* valid Twilio voice call initiation, establishing a WebSocket connection should succeed and return a unique connection identifier within the connection timeout period.

**Validates: Requirements 1.1**

### Property 2: Message Normalization Across Channels

*For any* valid webhook payload from either Twilio or WhatsApp, the Channel Abstraction Layer should normalize it into a Message object with all required fields (id, conversation_id, direction, content, timestamp) populated correctly.

**Validates: Requirements 1.2, 19.3**

### Property 3: Conversation Context Isolation

*For any* two different users or two different channels for the same user, their conversation contexts should remain independent such that updates to one context do not affect the other context.

**Validates: Requirements 1.4**

### Property 4: Concurrent Connection Handling

*For any* set of simultaneous connection requests from different users, the platform should successfully establish all connections without interference, maintaining separate state for each connection.

**Validates: Requirements 1.5**

### Property 5: Bedrock API Invocation

*For any* user message processed through an agent using Amazon Bedrock, the platform should invoke the Bedrock API with the message content, conversation history, and agent-specific system prompt.

**Validates: Requirements 3.3**

### Property 6: AI Configuration Persistence Round-Trip

*For any* valid AgentMetadata object containing Bedrock model identifiers or Amazon Q agent IDs, storing it to the database and then retrieving it by agent_name should produce an equivalent AgentMetadata object with all fields preserved.

**Validates: Requirements 3.5, 4.3**

### Property 7: Bedrock API Retry with Exponential Backoff

*For any* Bedrock or Amazon Q API call that fails with a retryable error, the platform should retry up to 3 times with exponentially increasing delays before failing permanently.

**Validates: Requirements 3.6**

### Property 8: Agent Routing Consistency

*For any* incoming message and conversation context, the Multi-Agent Orchestrator should route the message to exactly one agent, and that agent should have a confidence score greater than or equal to all other agents for that message-context pair.

**Validates: Requirements 4.1**

### Property 9: Model Selection Based on Agent Type

*For any* agent processing a message, the selected Bedrock foundation model should be appropriate for the agent type (e.g., Claude 3 Haiku for intent classification, Claude 3 Sonnet for complex tasks).

**Validates: Requirements 4.7**

### Property 10: WebSocket Cleanup and Data Persistence

*For any* active WebSocket connection, when the call ends, the connection should be gracefully closed and all call data (metadata, transcript reference, audio reference) should be persisted to the database before the cleanup completes.

**Validates: Requirements 2.4**

### Property 11: Audio Storage with Unique Identifiers

*For any* completed voice call, the audio recording should be stored in S3 with a unique key that includes the call SID, and the S3 key should be correctly referenced in the call_logs database table.

**Validates: Requirements 2.5, 11.3**

### Property 12: Log Credential Exclusion

*For any* log entry generated by the platform, the log content should not contain any strings matching credential patterns (API keys, passwords, tokens, connection strings with credentials).

**Validates: Requirements 6.3**

### Property 13: Webhook Signature Validation

*For any* webhook request, if the signature is computed correctly using the service's authentication mechanism, validation should pass; if the signature is invalid or missing, validation should fail and return HTTP 401.

**Validates: Requirements 7.1, 19.5**

### Property 14: Rate Limit Enforcement

*For any* user making requests, if the request count within a 1-minute window exceeds 100, subsequent requests should be rejected with HTTP 429 until the window resets.

**Validates: Requirements 7.4, 13.1**

### Property 15: IP Blocking After Repeated Failures

*For any* source IP with repeated webhook validation failures (more than 5 failures within 5 minutes), subsequent requests from that IP should be blocked for 15 minutes.

**Validates: Requirements 7.5**

### Property 16: Structured Log Format

*For any* log entry emitted by the backend, it should be valid JSON containing at minimum the fields: timestamp, level, message, and context (as an object).

**Validates: Requirements 10.1**

### Property 17: Call Data Persistence Completeness

*For any* completed voice call, the database should contain a call_logs record with all required fields populated: conversation_id, twilio_call_sid, duration_seconds, s3_audio_key, and created_at.

**Validates: Requirements 11.1**

### Property 18: Message Persistence and Retrieval

*For any* message sent through WhatsApp or voice channels, storing the message to the database and then querying by conversation_id should return a list containing that message with all fields preserved.

**Validates: Requirements 11.2**

### Property 19: Cache Miss Handling and Population

*For any* conversation_id not present in Redis cache, retrieving conversation context should load it from PostgreSQL, cache it in Redis with 30-minute TTL, and return the context; subsequent retrievals within the TTL should return the cached version without database access.

**Validates: Requirements 12.2, 12.5**

### Property 20: Cache Update After Message Exchange

*For any* conversation with cached context, processing a new message should update the cached context in Redis to include the new message in the message_history and update the last_activity timestamp.

**Validates: Requirements 12.3**

### Property 21: WebSocket Connection Limit Per User

*For any* user with 3 active WebSocket connections, attempting to establish a 4th connection should be rejected with an appropriate error indicating the connection limit has been reached.

**Validates: Requirements 13.3**

### Property 22: API Response Format Consistency

*For any* API endpoint response (success or error), the response should be valid JSON containing the fields: status (string), data (object or null), and error (object or null), where exactly one of data or error is non-null.

**Validates: Requirements 14.4, 14.5**

### Property 23: WebSocket Authentication Enforcement

*For any* WebSocket connection attempt, if authentication credentials are missing or invalid, the connection should be rejected before the upgrade completes; if credentials are valid, the connection should be established successfully.

**Validates: Requirements 14.3**

### Property 24: Message Routing Through Abstraction Layer

*For any* incoming message from any channel, the message should pass through the Channel Abstraction Layer before reaching the Multi-Agent Orchestrator, ensuring channel-specific processing is applied.

**Validates: Requirements 19.4**


## Error Handling

### Error Categories

The platform implements comprehensive error handling across four main categories:

#### 1. Client Errors (4xx)

**400 Bad Request:**
- Invalid request payload format
- Missing required fields
- Invalid enum values
- Malformed JSON

**401 Unauthorized:**
- Invalid webhook signature
- Missing authentication credentials
- Expired authentication tokens

**429 Too Many Requests:**
- Rate limit exceeded (requests per minute)
- Connection limit exceeded
- Concurrent operation limit exceeded

**404 Not Found:**
- Conversation not found
- User not found
- Resource not found

**Response Format:**
All error responses follow consistent JSON structure with status, data (null), and error object containing code, message, and details.

#### 2. Server Errors (5xx)

**500 Internal Server Error:**
- Unhandled exceptions
- Database connection failures
- Redis connection failures
- S3 operation failures
- Bedrock API errors (non-retryable)

**503 Service Unavailable:**
- Database unavailable
- Redis unavailable
- External service (Twilio/WhatsApp/Bedrock/Q) unavailable

**504 Gateway Timeout:**
- Database query timeout
- External API timeout
- WebSocket operation timeout
- Bedrock model invocation timeout

#### 3. WebSocket Errors

**Connection Errors:**
- Authentication failure → Close with code 4001
- Rate limit exceeded → Close with code 4029
- Invalid message format → Close with code 4400
- Internal error → Close with code 4500

**Error Message Format:**
WebSocket errors sent as JSON with event type "error", error code, and descriptive message.

#### 4. External Service Errors

**Twilio Errors:**
- API authentication failure → Retry with exponential backoff
- Rate limit → Respect Retry-After header
- Invalid phone number → Return 400 to client
- Network timeout → Retry up to 3 times

**WhatsApp Errors:**
- Message delivery failure → Log and notify
- Media download failure → Retry up to 3 times
- Invalid recipient → Return 400 to client

**Amazon Bedrock Errors:**
- Throttling → Retry with exponential backoff
- Model not found → Fall back to default model
- Invalid request → Return 400 to client
- Service unavailable → Use cached response or return 503

**Amazon Q Errors:**
- Agent not found → Fall back to Bedrock-based agent
- Knowledge base unavailable → Use Bedrock without knowledge base
- Session expired → Create new session
- Service unavailable → Return 503

### Error Handling Strategies

**Retry Logic:**
- Maximum 3 attempts for retryable errors
- Initial delay: 1 second
- Exponential backoff with base 2
- Maximum delay: 30 seconds
- Jitter enabled to prevent thundering herd
- Retryable exceptions: Network errors, throttling, transient failures

**Circuit Breaker:**
- Failure threshold: 5 consecutive failures
- Recovery timeout: 60 seconds
- States: closed (normal), open (failing), half-open (testing recovery)
- Applied to: Bedrock API, Amazon Q API, external webhooks

**Graceful Degradation:**

When services are unavailable:
- **Redis Unavailable**: Fall back to database for context, disable rate limiting (log warning), continue with reduced performance
- **S3 Unavailable**: Queue audio uploads for retry, store temporary references in database, continue call processing
- **Database Unavailable**: Return 503, reject new connections, maintain existing WebSocket connections, log all errors
- **Bedrock Unavailable**: Use cached responses if available, fall back to simple rule-based responses, notify monitoring
- **Amazon Q Unavailable**: Fall back to Bedrock-based agents, continue operation with reduced capabilities

### Error Logging

All errors logged to CloudWatch with structured JSON format including:
- Timestamp (ISO 8601)
- Log level (ERROR, WARN, INFO)
- Error message
- Context object (error type, operation, IDs, duration, retry attempt)
- Stack trace (for exceptions)
- Request ID (for tracing)

### Error Monitoring and Alerting

**CloudWatch Alarms:**
- Error rate > 5% over 5 minutes → Page on-call
- 5xx errors > 10 per minute → Alert team
- Database connection failures → Page on-call
- Redis connection failures → Alert team
- WebSocket disconnection rate > 20% → Alert team
- Bedrock API errors > 10 per minute → Alert team
- Amazon Q failures > 5 per minute → Alert team


## Testing Strategy

### Dual Testing Approach

The platform requires both unit testing and property-based testing for comprehensive coverage. These approaches are complementary:

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties across all inputs

Together, they provide comprehensive coverage where unit tests catch concrete bugs and property tests verify general correctness.

### Property-Based Testing

**Framework:** Hypothesis (Python)

**Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test references its design document property
- Tag format: `# Feature: aws-voice-whatsapp-ai-platform, Property {number}: {property_text}`

**Test Categories:**

**Channel Abstraction Tests:**
- Property 2: Message normalization across channels
- Property 24: Message routing through abstraction layer
- Generate random webhook payloads for both Twilio and WhatsApp
- Verify normalized message structure

**Agent Orchestration Tests:**
- Property 8: Agent routing consistency
- Property 9: Model selection based on agent type
- Generate random messages and conversation contexts
- Verify agent selection logic and model assignment

**Data Persistence Tests:**
- Property 6: AI configuration round-trip
- Property 17: Call data persistence completeness
- Property 18: Message persistence and retrieval
- Generate random data objects and verify round-trip integrity

**Caching Tests:**
- Property 19: Cache miss handling and population
- Property 20: Cache update after message exchange
- Generate random conversation IDs and verify cache behavior

**Security Tests:**
- Property 12: Log credential exclusion
- Property 13: Webhook signature validation
- Generate random log entries and webhook requests
- Verify security properties hold

**Rate Limiting Tests:**
- Property 14: Rate limit enforcement
- Property 15: IP blocking after repeated failures
- Property 21: WebSocket connection limit per user
- Generate random request patterns and verify limits

**Custom Generators:**
- Conversation context generator (random messages, agents, metadata)
- Webhook payload generator (valid and invalid signatures)
- Message generator (various content types and lengths)
- Agent configuration generator (different models and settings)

### Unit Testing

**Framework:** pytest with pytest-asyncio

**Test Categories:**

#### 1. Component Tests

Test individual components in isolation with mocked dependencies:

**Channel Adapters:**
- Test Twilio signature validation with known signatures
- Test WhatsApp message parsing with sample payloads
- Test TwiML generation for call control
- Test media download handling

**Bedrock Service:**
- Test model invocation with mocked Boto3 client
- Test request body formatting for different models
- Test response parsing for Claude, Titan, Llama
- Test streaming response handling
- Test retry logic with simulated failures

**Amazon Q Service:**
- Test agent creation with mocked client
- Test agent invocation with session management
- Test citation extraction
- Test trace logging

**Agent Orchestrator:**
- Test agent selection with known confidence scores
- Test conversation context loading and saving
- Test agent transition logic
- Test fallback agent routing

**WebSocket Manager:**
- Test connection acceptance and registration
- Test audio buffer accumulation
- Test connection cleanup
- Test connection limit enforcement

#### 2. Integration Tests

Test interactions between components:

**End-to-End Message Flow:**
- Webhook → Channel Adapter → Orchestrator → Agent → Bedrock → Response
- Verify message transformations at each step
- Verify database and cache updates
- Verify logging at each stage

**Voice Call Flow:**
- WebSocket connection → Audio processing → Speech-to-text → Orchestrator → Bedrock → Text-to-speech → Audio response
- Verify real-time processing
- Verify call log persistence
- Verify audio storage in S3

**Multi-Agent Conversation:**
- Send sequence of messages requiring agent transitions
- Verify agent selection changes appropriately
- Verify conversation context maintained across transitions
- Verify Bedrock model changes with agent type

#### 3. Edge Case Tests

Test boundary conditions and edge cases:

**Rate Limiting Boundaries:**
- Test exactly at limit (100 requests)
- Test just over limit (101 requests)
- Test limit reset after window expires

**Empty and Invalid Inputs:**
- Empty message content
- Malformed webhook payloads
- Invalid phone numbers
- Missing required fields

**Connection Limits:**
- Test exactly 3 WebSocket connections per user
- Test 4th connection rejection
- Test connection cleanup and re-establishment

**Cache Expiration:**
- Test context retrieval after TTL expiration
- Test cache warming on first access
- Test cache invalidation

**Bedrock Edge Cases:**
- Test with very long prompts (near token limit)
- Test with empty conversation history
- Test with unsupported model IDs
- Test with malformed responses

#### 4. Error Handling Tests

Test error conditions and recovery:

**Service Failures:**
- Database connection failure handling
- Redis connection failure handling
- S3 upload failure handling
- Bedrock API failure handling
- Amazon Q API failure handling

**Retry Logic:**
- Test exponential backoff timing
- Test maximum retry attempts
- Test jitter application
- Test retry on specific error types

**Circuit Breaker:**
- Test circuit opening after threshold failures
- Test circuit half-open state
- Test circuit closing after successful recovery
- Test timeout behavior

**Graceful Degradation:**
- Test operation with Redis unavailable
- Test operation with S3 unavailable
- Test operation with Bedrock unavailable
- Verify fallback behaviors

### Test Infrastructure

**Test Fixtures:**
- Test database with schema (create/teardown)
- Test Redis instance (flush between tests)
- Mocked S3 client (using moto library)
- Mocked Bedrock client (using moto or custom mocks)
- Mocked Amazon Q client
- Sample webhook payloads (Twilio and WhatsApp)
- Sample conversation contexts

**Test Configuration:**
- Separate test environment configuration
- Test database and Redis URLs
- Mock AWS credentials
- Disabled external API calls (mocked)
- Fast timeouts for quick test execution

### Test Coverage Goals

- **Line coverage:** Minimum 80%
- **Branch coverage:** Minimum 75%
- **Property test coverage:** All 24 correctness properties implemented
- **Critical paths:** 100% coverage (authentication, data persistence, error handling, Bedrock/Q integration)

### Continuous Integration

Tests run automatically on:
- Every pull request
- Every commit to main branch
- Nightly builds (extended property test suite with 1000 iterations)

**CI Pipeline:**
1. Lint and type checking (mypy, ruff)
2. Unit tests (parallel execution)
3. Property tests (100 iterations)
4. Integration tests (with test database and Redis)
5. Coverage report generation
6. Security scanning (bandit, safety)
7. Dependency vulnerability scanning

