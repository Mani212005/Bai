# Design Document: AWS-Deployed Voice + WhatsApp AI Platform

## Overview

This design document specifies the architecture and implementation details for a production-grade, cloud-native AI communication platform deployed on AWS. The system enables multi-channel interactions through voice calls (Twilio) and WhatsApp messaging, featuring real-time WebSocket support, multi-agent orchestration, and comprehensive observability.

### Architecture Philosophy

The platform follows cloud-native principles with:
- **Containerized microservices** on ECS Fargate for scalability and isolation
- **Managed AWS services** for reduced operational overhead
- **Event-driven architecture** for real-time communication
- **Infrastructure as Code** for reproducible deployments
- **Defense in depth** security model
- **Observability-first** design for production operations

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
   ├─→ RDS PostgreSQL (Call logs, transcripts, metadata)
   ├─→ ElastiCache Redis (Sessions, cache, rate limits)
   ├─→ S3 (Audio recordings, debug exports)
   ├─→ Secrets Manager (Credentials)
   └─→ CloudWatch (Logs, metrics, alarms)
```

### Technology Stack

**Backend:**
- FastAPI (Python 3.11+) - High-performance async web framework
- Uvicorn - ASGI server with WebSocket support
- SQLAlchemy - ORM for PostgreSQL
- Redis-py - Redis client with connection pooling
- Boto3 - AWS SDK for Python
- Pydantic - Data validation and settings management

**Infrastructure:**
- AWS ECS Fargate - Serverless container compute
- AWS ALB - Layer 7 load balancing with WebSocket support
- AWS RDS PostgreSQL 15 - Relational database
- AWS ElastiCache Redis 7 - In-memory cache
- AWS S3 - Object storage
- AWS CloudWatch - Monitoring and logging
- AWS Secrets Manager - Credential management
- AWS IAM - Access control

**External Services:**
- Twilio Voice API - Voice call handling
- Twilio Media Streams - Real-time audio streaming
- WhatsApp Business API - Messaging integration


## Architecture

### System Components

#### 1. API Gateway Layer (ALB)

The Application Load Balancer serves as the entry point for all traffic:

**Responsibilities:**
- SSL/TLS termination using ACM certificates
- HTTP to HTTPS redirection
- WebSocket connection upgrade handling
- Health check routing to backend instances
- Request routing based on path patterns
- Connection draining during deployments

**Configuration:**
- Listener on port 443 (HTTPS) with ACM certificate
- Listener on port 80 (HTTP) redirecting to 443
- Target group pointing to ECS Fargate tasks
- Health check path: `/health` (30-second interval)
- Deregistration delay: 30 seconds for graceful shutdown
- Sticky sessions disabled (stateless design)

#### 2. Compute Layer (ECS Fargate)

Containerized FastAPI application running on serverless compute:

**Container Specifications:**
- Base image: python:3.11-slim
- CPU: 1 vCPU (1024 units)
- Memory: 2 GB
- Port mapping: 8000 (container) → 8000 (host)
- Health check: HTTP GET /health every 30 seconds

**Auto-scaling Configuration:**
- Minimum tasks: 2 (across 2 availability zones)
- Maximum tasks: 10
- Target CPU utilization: 70%
- Scale-up cooldown: 60 seconds
- Scale-down cooldown: 300 seconds

**Task Definition:**
- Network mode: awsvpc (required for Fargate)
- Task role: ECS task execution role with permissions
- Environment variables: Loaded from Secrets Manager
- Logging: CloudWatch Logs with awslogs driver

#### 3. Data Layer

**RDS PostgreSQL:**
- Engine: PostgreSQL 15.x
- Instance class: db.t4g.medium (2 vCPU, 4 GB RAM)
- Storage: 100 GB gp3 with autoscaling to 500 GB
- Multi-AZ: Enabled for high availability
- Backup retention: 7 days
- Encryption: At rest (KMS) and in transit (SSL)

**Schema Design:**
```
users
  - id (UUID, primary key)
  - phone_number (VARCHAR, unique, indexed)
  - created_at (TIMESTAMP)
  - updated_at (TIMESTAMP)

conversations
  - id (UUID, primary key)
  - user_id (UUID, foreign key)
  - channel_type (ENUM: voice, whatsapp)
  - status (ENUM: active, completed, failed)
  - started_at (TIMESTAMP)
  - ended_at (TIMESTAMP)
  - metadata (JSONB)

messages
  - id (UUID, primary key)
  - conversation_id (UUID, foreign key, indexed)
  - direction (ENUM: inbound, outbound)
  - content (TEXT)
  - agent_id (VARCHAR, indexed)
  - timestamp (TIMESTAMP, indexed)
  - metadata (JSONB)

call_logs
  - id (UUID, primary key)
  - conversation_id (UUID, foreign key)
  - twilio_call_sid (VARCHAR, unique, indexed)
  - duration_seconds (INTEGER)
  - recording_url (VARCHAR)
  - s3_audio_key (VARCHAR)
  - transcript (TEXT)
  - created_at (TIMESTAMP)

agent_metadata
  - id (UUID, primary key)
  - agent_name (VARCHAR, unique)
  - agent_type (VARCHAR)
  - configuration (JSONB)
  - is_active (BOOLEAN)
  - created_at (TIMESTAMP)
```

**ElastiCache Redis:**
- Engine: Redis 7.x
- Node type: cache.t4g.medium (2 vCPU, 3.09 GB RAM)
- Cluster mode: Enabled with 2 shards
- Replicas: 1 per shard (for failover)
- Encryption: At rest and in transit

**Redis Key Patterns:**
```
session:{user_id}:{channel} → Conversation context (TTL: 30 min)
rate_limit:{user_id}:requests → Request counter (TTL: 1 min)
rate_limit:{user_id}:connections → WebSocket counter (TTL: 1 hour)
agent_state:{conversation_id} → Active agent state (TTL: 30 min)
webhook_nonce:{nonce} → Replay prevention (TTL: 5 min)
```

#### 4. Storage Layer (S3)

**Bucket Structure:**
```
voice-whatsapp-ai-platform-{region}-{account_id}/
  ├── audio-recordings/
  │   └── {year}/{month}/{day}/{call_sid}.wav
  ├── debug-exports/
  │   └── {year}/{month}/{day}/{conversation_id}.json
  └── conversation-traces/
      └── {year}/{month}/{day}/{conversation_id}_trace.json
```

**Lifecycle Policies:**
- Audio recordings: Transition to Glacier after 90 days
- Debug exports: Delete after 30 days
- Conversation traces: Transition to Glacier after 180 days

**Bucket Configuration:**
- Versioning: Enabled
- Encryption: SSE-S3 (AES-256)
- Public access: Blocked
- CORS: Configured for frontend access (if needed)

#### 5. Security Layer

**Secrets Manager:**
Stores sensitive credentials with automatic rotation:
```
/voice-whatsapp-platform/prod/database
  - host, port, username, password, database

/voice-whatsapp-platform/prod/redis
  - host, port, auth_token

/voice-whatsapp-platform/prod/twilio
  - account_sid, auth_token, api_key

/voice-whatsapp-platform/prod/whatsapp
  - api_key, webhook_verify_token, phone_number_id
```

**IAM Roles:**

ECS Task Execution Role:
- Pull images from ECR
- Write logs to CloudWatch
- Read secrets from Secrets Manager

ECS Task Role:
- Read/write to S3 bucket
- Read/write to RDS (via security group)
- Read/write to ElastiCache (via security group)
- Publish metrics to CloudWatch
- Read secrets from Secrets Manager

**Security Groups:**

ALB Security Group:
- Inbound: 443 (HTTPS) from 0.0.0.0/0
- Inbound: 80 (HTTP) from 0.0.0.0/0
- Outbound: 8000 to ECS security group

ECS Security Group:
- Inbound: 8000 from ALB security group
- Outbound: 443 to 0.0.0.0/0 (for AWS API calls)
- Outbound: 5432 to RDS security group
- Outbound: 6379 to ElastiCache security group

RDS Security Group:
- Inbound: 5432 from ECS security group

ElastiCache Security Group:
- Inbound: 6379 from ECS security group


## Components and Interfaces

### 1. Channel Abstraction Layer

Provides a unified interface for handling different communication channels.

**Interface Definition:**

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from enum import Enum

class ChannelType(Enum):
    VOICE = "voice"
    WHATSAPP = "whatsapp"

class MessageDirection(Enum):
    INBOUND = "inbound"
    OUTBOUND = "outbound"

class Message:
    """Normalized message structure across channels"""
    id: str
    conversation_id: str
    user_id: str
    channel: ChannelType
    direction: MessageDirection
    content: str
    timestamp: datetime
    metadata: Dict[str, Any]

class ChannelAdapter(ABC):
    """Abstract base class for channel implementations"""
    
    @abstractmethod
    async def validate_webhook(self, request: Request) -> bool:
        """Validate incoming webhook signature"""
        pass
    
    @abstractmethod
    async def receive_message(self, webhook_data: Dict[str, Any]) -> Message:
        """Parse webhook data into normalized Message"""
        pass
    
    @abstractmethod
    async def send_message(self, message: Message) -> bool:
        """Send message through the channel"""
        pass
    
    @abstractmethod
    async def get_message_status(self, message_id: str) -> str:
        """Query delivery status of sent message"""
        pass
```

**Implementations:**

**TwilioVoiceAdapter:**
- Validates Twilio webhook signatures using X-Twilio-Signature header
- Handles TwiML responses for call control
- Manages WebSocket connections for Media Streams
- Converts audio streams to text using speech-to-text
- Converts AI responses to audio using text-to-speech

**WhatsAppAdapter:**
- Validates WhatsApp webhook signatures
- Parses WhatsApp message formats (text, media, interactive)
- Handles message status callbacks (sent, delivered, read)
- Manages media downloads from WhatsApp CDN
- Formats outbound messages with WhatsApp API requirements

### 2. Multi-Agent Orchestrator

Manages AI agent selection, routing, and conversation flow.

**Core Components:**

```python
class AgentType(Enum):
    GREETING = "greeting"
    INTENT_CLASSIFIER = "intent_classifier"
    INFORMATION_RETRIEVAL = "information_retrieval"
    TASK_EXECUTOR = "task_executor"
    FALLBACK = "fallback"

class Agent:
    """Base agent interface"""
    id: str
    name: str
    agent_type: AgentType
    configuration: Dict[str, Any]
    
    async def process(self, message: Message, context: ConversationContext) -> AgentResponse:
        """Process message and return response"""
        pass
    
    async def can_handle(self, message: Message, context: ConversationContext) -> float:
        """Return confidence score (0-1) for handling this message"""
        pass

class ConversationContext:
    """Maintains conversation state"""
    conversation_id: str
    user_id: str
    channel: ChannelType
    message_history: List[Message]
    current_agent: Optional[str]
    metadata: Dict[str, Any]
    created_at: datetime
    last_activity: datetime

class AgentOrchestrator:
    """Routes messages to appropriate agents"""
    
    def __init__(self, agents: List[Agent]):
        self.agents = agents
    
    async def route_message(self, message: Message, context: ConversationContext) -> Agent:
        """Select best agent for handling message"""
        scores = await asyncio.gather(*[
            agent.can_handle(message, context) for agent in self.agents
        ])
        best_agent_idx = scores.index(max(scores))
        return self.agents[best_agent_idx]
    
    async def process_message(self, message: Message) -> Message:
        """Main orchestration logic"""
        # Load or create conversation context
        context = await self.load_context(message.conversation_id)
        
        # Route to appropriate agent
        agent = await self.route_message(message, context)
        
        # Process message
        response = await agent.process(message, context)
        
        # Update context
        context.message_history.append(message)
        context.current_agent = agent.id
        await self.save_context(context)
        
        return response.to_message()
```

**Agent Selection Strategy:**
1. Check if current agent can continue handling (confidence > 0.7)
2. If not, query all agents for confidence scores
3. Select agent with highest confidence
4. If all scores < 0.5, route to fallback agent
5. Cache agent selection in Redis for performance

### 3. WebSocket Manager

Handles real-time bidirectional communication for Twilio Media Streams.

**Architecture:**

```python
class WebSocketConnection:
    """Represents an active WebSocket connection"""
    connection_id: str
    user_id: str
    call_sid: str
    websocket: WebSocket
    created_at: datetime
    last_activity: datetime
    audio_buffer: List[bytes]

class WebSocketManager:
    """Manages WebSocket lifecycle and message routing"""
    
    def __init__(self):
        self.active_connections: Dict[str, WebSocketConnection] = {}
        self.connection_lock = asyncio.Lock()
    
    async def connect(self, websocket: WebSocket, call_sid: str) -> str:
        """Accept new WebSocket connection"""
        await websocket.accept()
        connection_id = str(uuid.uuid4())
        
        async with self.connection_lock:
            self.active_connections[connection_id] = WebSocketConnection(
                connection_id=connection_id,
                call_sid=call_sid,
                websocket=websocket,
                created_at=datetime.utcnow(),
                last_activity=datetime.utcnow(),
                audio_buffer=[]
            )
        
        return connection_id
    
    async def handle_media_stream(self, connection_id: str):
        """Process incoming Twilio Media Stream messages"""
        connection = self.active_connections[connection_id]
        
        try:
            while True:
                data = await connection.websocket.receive_json()
                
                if data["event"] == "media":
                    # Accumulate audio chunks
                    audio_payload = base64.b64decode(data["media"]["payload"])
                    connection.audio_buffer.append(audio_payload)
                    
                    # Process when buffer reaches threshold
                    if len(connection.audio_buffer) >= 20:  # ~1 second of audio
                        await self.process_audio_buffer(connection)
                
                elif data["event"] == "stop":
                    await self.disconnect(connection_id)
                    break
                
                connection.last_activity = datetime.utcnow()
        
        except WebSocketDisconnect:
            await self.disconnect(connection_id)
    
    async def process_audio_buffer(self, connection: WebSocketConnection):
        """Convert audio to text and process through orchestrator"""
        audio_data = b"".join(connection.audio_buffer)
        connection.audio_buffer.clear()
        
        # Speech-to-text conversion
        text = await self.speech_to_text(audio_data)
        
        # Create message and process
        message = Message(
            conversation_id=connection.call_sid,
            user_id=connection.user_id,
            channel=ChannelType.VOICE,
            direction=MessageDirection.INBOUND,
            content=text,
            timestamp=datetime.utcnow()
        )
        
        # Process through orchestrator
        response = await orchestrator.process_message(message)
        
        # Convert response to audio and send
        audio_response = await self.text_to_speech(response.content)
        await self.send_audio(connection, audio_response)
    
    async def send_audio(self, connection: WebSocketConnection, audio_data: bytes):
        """Send audio back through WebSocket"""
        payload = base64.b64encode(audio_data).decode("utf-8")
        await connection.websocket.send_json({
            "event": "media",
            "streamSid": connection.call_sid,
            "media": {
                "payload": payload
            }
        })
    
    async def disconnect(self, connection_id: str):
        """Clean up connection"""
        async with self.connection_lock:
            if connection_id in self.active_connections:
                connection = self.active_connections[connection_id]
                await connection.websocket.close()
                del self.active_connections[connection_id]
```

### 4. Database Service

Handles all database operations with connection pooling and error handling.

```python
class DatabaseService:
    """Manages PostgreSQL connections and queries"""
    
    def __init__(self, connection_string: str):
        self.engine = create_async_engine(
            connection_string,
            pool_size=20,
            max_overflow=0,
            pool_pre_ping=True,
            echo=False
        )
        self.async_session = sessionmaker(
            self.engine,
            class_=AsyncSession,
            expire_on_commit=False
        )
    
    async def create_conversation(self, user_id: str, channel: ChannelType) -> str:
        """Create new conversation record"""
        async with self.async_session() as session:
            conversation = Conversation(
                id=str(uuid.uuid4()),
                user_id=user_id,
                channel_type=channel.value,
                status="active",
                started_at=datetime.utcnow()
            )
            session.add(conversation)
            await session.commit()
            return conversation.id
    
    async def save_message(self, message: Message):
        """Persist message to database"""
        async with self.async_session() as session:
            db_message = DBMessage(
                id=message.id,
                conversation_id=message.conversation_id,
                direction=message.direction.value,
                content=message.content,
                agent_id=message.metadata.get("agent_id"),
                timestamp=message.timestamp,
                metadata=message.metadata
            )
            session.add(db_message)
            await session.commit()
    
    async def get_conversation_history(
        self, 
        conversation_id: str, 
        limit: int = 50
    ) -> List[Message]:
        """Retrieve recent messages for conversation"""
        async with self.async_session() as session:
            result = await session.execute(
                select(DBMessage)
                .where(DBMessage.conversation_id == conversation_id)
                .order_by(DBMessage.timestamp.desc())
                .limit(limit)
            )
            db_messages = result.scalars().all()
            return [self._to_message(msg) for msg in reversed(db_messages)]
```

### 5. Cache Service

Manages Redis operations for sessions, rate limiting, and caching.

```python
class CacheService:
    """Manages Redis operations"""
    
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(
            redis_url,
            encoding="utf-8",
            decode_responses=True,
            max_connections=50
        )
    
    async def get_conversation_context(self, conversation_id: str) -> Optional[ConversationContext]:
        """Retrieve cached conversation context"""
        key = f"session:{conversation_id}"
        data = await self.redis.get(key)
        if data:
            return ConversationContext.parse_raw(data)
        return None
    
    async def save_conversation_context(self, context: ConversationContext):
        """Cache conversation context with TTL"""
        key = f"session:{context.conversation_id}"
        await self.redis.setex(
            key,
            1800,  # 30 minutes
            context.json()
        )
    
    async def check_rate_limit(self, user_id: str, limit: int = 100) -> bool:
        """Check if user is within rate limits"""
        key = f"rate_limit:{user_id}:requests"
        current = await self.redis.incr(key)
        
        if current == 1:
            await self.redis.expire(key, 60)  # 1 minute window
        
        return current <= limit
    
    async def increment_connection_count(self, user_id: str) -> int:
        """Track active WebSocket connections per user"""
        key = f"rate_limit:{user_id}:connections"
        count = await self.redis.incr(key)
        await self.redis.expire(key, 3600)  # 1 hour
        return count
    
    async def decrement_connection_count(self, user_id: str):
        """Decrease connection count when WebSocket closes"""
        key = f"rate_limit:{user_id}:connections"
        await self.redis.decr(key)
```

### 6. Storage Service

Handles S3 operations for audio recordings and debug exports.

```python
class StorageService:
    """Manages S3 operations"""
    
    def __init__(self, bucket_name: str):
        self.s3_client = boto3.client("s3")
        self.bucket_name = bucket_name
    
    async def upload_audio_recording(
        self, 
        call_sid: str, 
        audio_data: bytes
    ) -> str:
        """Upload audio recording to S3"""
        now = datetime.utcnow()
        key = f"audio-recordings/{now.year}/{now.month:02d}/{now.day:02d}/{call_sid}.wav"
        
        await asyncio.to_thread(
            self.s3_client.put_object,
            Bucket=self.bucket_name,
            Key=key,
            Body=audio_data,
            ContentType="audio/wav",
            ServerSideEncryption="AES256"
        )
        
        return key
    
    async def export_conversation_trace(
        self, 
        conversation_id: str, 
        trace_data: Dict[str, Any]
    ):
        """Export conversation trace for debugging"""
        now = datetime.utcnow()
        key = f"conversation-traces/{now.year}/{now.month:02d}/{now.day:02d}/{conversation_id}_trace.json"
        
        await asyncio.to_thread(
            self.s3_client.put_object,
            Bucket=self.bucket_name,
            Key=key,
            Body=json.dumps(trace_data, indent=2),
            ContentType="application/json",
            ServerSideEncryption="AES256"
        )
```


## Data Models

### Core Domain Models

```python
from datetime import datetime
from typing import Optional, Dict, Any, List
from enum import Enum
from pydantic import BaseModel, Field
import uuid

class ChannelType(str, Enum):
    VOICE = "voice"
    WHATSAPP = "whatsapp"

class MessageDirection(str, Enum):
    INBOUND = "inbound"
    OUTBOUND = "outbound"

class ConversationStatus(str, Enum):
    ACTIVE = "active"
    COMPLETED = "completed"
    FAILED = "failed"

class User(BaseModel):
    """User entity"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    phone_number: str
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

class Conversation(BaseModel):
    """Conversation entity"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    user_id: str
    channel_type: ChannelType
    status: ConversationStatus = ConversationStatus.ACTIVE
    started_at: datetime = Field(default_factory=datetime.utcnow)
    ended_at: Optional[datetime] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)

class Message(BaseModel):
    """Message entity"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    conversation_id: str
    direction: MessageDirection
    content: str
    agent_id: Optional[str] = None
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    metadata: Dict[str, Any] = Field(default_factory=dict)

class CallLog(BaseModel):
    """Call log entity"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    conversation_id: str
    twilio_call_sid: str
    duration_seconds: int
    recording_url: Optional[str] = None
    s3_audio_key: Optional[str] = None
    transcript: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)

class AgentMetadata(BaseModel):
    """Agent metadata entity"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    agent_name: str
    agent_type: str
    configuration: Dict[str, Any] = Field(default_factory=dict)
    is_active: bool = True
    created_at: datetime = Field(default_factory=datetime.utcnow)

class ConversationContext(BaseModel):
    """In-memory conversation context"""
    conversation_id: str
    user_id: str
    channel: ChannelType
    message_history: List[Message] = Field(default_factory=list)
    current_agent: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    last_activity: datetime = Field(default_factory=datetime.utcnow)
```

### API Request/Response Models

```python
class WebhookValidationRequest(BaseModel):
    """Webhook validation request"""
    signature: str
    timestamp: str
    body: str

class TwilioVoiceWebhook(BaseModel):
    """Twilio voice webhook payload"""
    CallSid: str
    From: str
    To: str
    CallStatus: str
    Direction: str

class WhatsAppWebhook(BaseModel):
    """WhatsApp webhook payload"""
    object: str
    entry: List[Dict[str, Any]]

class SendMessageRequest(BaseModel):
    """API request to send message"""
    conversation_id: str
    content: str
    channel: ChannelType

class SendMessageResponse(BaseModel):
    """API response for sent message"""
    message_id: str
    status: str
    timestamp: datetime

class ConversationHistoryResponse(BaseModel):
    """API response for conversation history"""
    conversation_id: str
    messages: List[Message]
    total_count: int

class HealthCheckResponse(BaseModel):
    """Health check response"""
    status: str
    timestamp: datetime
    services: Dict[str, str]  # service_name -> status
```

### Configuration Models

```python
class DatabaseConfig(BaseModel):
    """Database configuration"""
    host: str
    port: int = 5432
    username: str
    password: str
    database: str
    pool_size: int = 20
    
    @property
    def connection_string(self) -> str:
        return f"postgresql+asyncpg://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"

class RedisConfig(BaseModel):
    """Redis configuration"""
    host: str
    port: int = 6379
    auth_token: Optional[str] = None
    ssl: bool = True
    
    @property
    def connection_string(self) -> str:
        protocol = "rediss" if self.ssl else "redis"
        auth = f":{self.auth_token}@" if self.auth_token else ""
        return f"{protocol}://{auth}{self.host}:{self.port}"

class TwilioConfig(BaseModel):
    """Twilio configuration"""
    account_sid: str
    auth_token: str
    api_key: str
    phone_number: str

class WhatsAppConfig(BaseModel):
    """WhatsApp configuration"""
    api_key: str
    webhook_verify_token: str
    phone_number_id: str

class AWSConfig(BaseModel):
    """AWS configuration"""
    region: str
    s3_bucket: str
    secrets_prefix: str = "/voice-whatsapp-platform/prod"

class AppConfig(BaseModel):
    """Application configuration"""
    environment: str = "production"
    log_level: str = "INFO"
    database: DatabaseConfig
    redis: RedisConfig
    twilio: TwilioConfig
    whatsapp: WhatsAppConfig
    aws: AWSConfig
```

