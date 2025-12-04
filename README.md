# ChatApp

A production-ready, microservices-based real-time chat application built with TypeScript and Node.js. This project implements an API Gateway pattern with event-driven architecture, featuring JWT authentication, multi-database persistence, and Redis caching.

## Table of Contents

- [Architecture](#architecture)
- [Services](#services)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Development](#development)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)

## Architecture

The application follows a microservices architecture with the following key patterns:

- **API Gateway Pattern**: All client requests flow through a single gateway service that handles authentication, request validation, and routing to internal services.
- **Event-Driven Architecture**: Services communicate asynchronously through RabbitMQ for loose coupling and eventual consistency.
- **Database per Service**: Each service manages its own database, promoting independence and scalability.

**Request Flow**: Client requests hit the Gateway Service, which validates JWT tokens and forwards requests to the appropriate internal service (Auth, User, or Chat). Services communicate with each other through RabbitMQ events (e.g., user registration events are published by Auth Service and consumed by User Service to sync profile data).

## Services

### Gateway Service (Port 4000)

The public-facing entry point for all API requests. Responsibilities include:

- JWT token validation and user authentication
- Request routing to internal services
- Request/response transformation
- Rate limiting and security headers (Helmet)

### Auth Service (Port 4003)

Manages user authentication and authorization:

- User registration with password hashing (bcrypt)
- Login with credential validation
- JWT access token and refresh token generation
- Token refresh with rotation
- Token revocation
- Publishes user registration events to RabbitMQ

### User Service (Port 4001)

Manages user profile data:

- User profile retrieval
- User search functionality
- Consumes auth events to sync user data from registration

### Chat Service (Port 4002)

Handles real-time messaging features:

- Conversation creation and management
- Message sending and retrieval
- Redis caching for conversation data
- Message pagination with cursor-based navigation

## Technology Stack

| Category              | Technology           |
| --------------------- | -------------------- |
| Runtime               | Node.js 22           |
| Language              | TypeScript (ESM)     |
| Package Manager       | pnpm 10 (workspaces) |
| Web Framework         | Express 5            |
| Auth Service Database | MySQL 8.0            |
| User Service Database | PostgreSQL 16        |
| Chat Service Database | MongoDB 7            |
| Caching               | Redis 7              |
| Message Broker        | RabbitMQ 3           |
| ORM                   | Sequelize 6          |
| Validation            | Zod                  |
| Logging               | Pino                 |
| Containerization      | Docker               |

## Prerequisites

- Node.js 22 or higher
- pnpm 10 or higher
- Docker and Docker Compose
- Git

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/fiston-user/chatapp-yt.git
cd chatapp-yt
```

### 2. Install dependencies

```bash
pnpm install
```

### 3. Configure environment variables

Create a `.env` file in the root directory based on the example:

```bash
cp .env.example .env
```

Update the environment variables as needed.

### 4. Start infrastructure services

```bash
docker compose up -d rabbitmq redis mongo user-db auth-db
```

### 5. Build the common package

```bash
pnpm --filter @chatapp/common build
```

### 6. Start all services (development)

```bash
pnpm dev
```

Or start individual services:

```bash
pnpm --filter gateway-service dev
pnpm --filter @chatapp/auth-service dev
pnpm --filter @chatapp/user-service dev
pnpm --filter chat-service dev
```

### Running with Docker Compose (Production)

To run the entire stack with Docker:

```bash
docker compose up -d
```

This will start all services and their dependencies with health checks and automatic restarts.

## Configuration

### Environment Variables

| Variable                 | Description                       | Default       |
| ------------------------ | --------------------------------- | ------------- |
| `NODE_ENV`               | Environment mode                  | `development` |
| `GATEWAY_PORT`           | Gateway service port              | `4000`        |
| `AUTH_SERVICE_PORT`      | Auth service port                 | `4003`        |
| `USER_SERVICE_PORT`      | User service port                 | `4001`        |
| `CHAT_SERVICE_PORT`      | Chat service port                 | `4002`        |
| `JWT_SECRET`             | Secret for signing access tokens  | -             |
| `JWT_EXPIRES_IN`         | Access token expiry               | `1d`          |
| `JWT_REFRESH_SECRET`     | Secret for signing refresh tokens | -             |
| `JWT_REFRESH_EXPIRES_IN` | Refresh token expiry              | `30d`         |
| `INTERNAL_API_TOKEN`     | Token for inter-service auth      | -             |
| `AUTH_DB_URL`            | MySQL connection URL              | -             |
| `USER_DB_URL`            | PostgreSQL connection URL         | -             |
| `MONGO_URL`              | MongoDB connection URL            | -             |
| `REDIS_URL`              | Redis connection URL              | -             |
| `RABBITMQ_URL`           | RabbitMQ connection URL           | -             |

### Default Infrastructure Ports

| Service             | Port  |
| ------------------- | ----- |
| RabbitMQ            | 5672  |
| RabbitMQ Management | 15672 |
| Redis               | 6379  |
| MongoDB             | 27017 |
| PostgreSQL          | 5432  |
| MySQL               | 3306  |

## API Reference

### Authentication Endpoints

All authentication endpoints are accessed through the gateway service at `http://localhost:4000`.

#### Register User

```http
POST /auth/register
Content-Type: application/json

{
    "email": "user@example.com",
    "displayName": "John Doe",
    "password": "SecurePassword123!"
}
```

**Response** (201 Created):

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "John Doe",
    "createdAt": "2024-01-01T00:00:00.000Z"
  }
}
```

#### Login

```http
POST /auth/login
Content-Type: application/json

{
    "email": "user@example.com",
    "password": "SecurePassword123!"
}
```

**Response** (200 OK):

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### Refresh Tokens

```http
POST /auth/refresh
Content-Type: application/json

{
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### Revoke Tokens

```http
POST /auth/revoke
Content-Type: application/json

{
    "userId": "uuid"
}
```

### Conversation Endpoints

All conversation endpoints require authentication via Bearer token.

#### Create Conversation

```http
POST /conversations
Authorization: Bearer <access_token>
Content-Type: application/json

{
    "title": "Project Discussion",
    "participantIds": ["user-id-1", "user-id-2"]
}
```

#### List Conversations

```http
GET /conversations
Authorization: Bearer <access_token>
```

#### Get Conversation

```http
GET /conversations/:id
Authorization: Bearer <access_token>
```

### Message Endpoints

#### Send Message

```http
POST /conversations/:id/messages
Authorization: Bearer <access_token>
Content-Type: application/json

{
    "body": "Hello, world!"
}
```

#### List Messages

```http
GET /conversations/:id/messages?limit=50&after=<message_id>
Authorization: Bearer <access_token>
```

### Health Checks

Each service exposes a health endpoint:

```http
GET /health
```

**Response**:

```json
{
  "status": "ok",
  "service": "<service-name>"
}
```

## Development

### Scripts

| Command          | Description                            |
| ---------------- | -------------------------------------- |
| `pnpm dev`       | Start all services in development mode |
| `pnpm build`     | Build all packages and services        |
| `pnpm lint`      | Run ESLint on all packages             |
| `pnpm format`    | Check code formatting with Prettier    |
| `pnpm typecheck` | Run TypeScript type checking           |

### Code Quality

The project uses:

- ESLint for code linting
- Prettier for code formatting
- TypeScript strict mode for type safety

## Project Structure

This is a monorepo managed with pnpm workspaces. The root contains `docker-compose.yml` for infrastructure, along with shared TypeScript and workspace configs.

**packages/common**: Shared library containing error classes, event types, HTTP middleware, validation utilities, and logging setup. All services depend on this package.

**services/gateway-service**: The API gateway. Contains controllers that proxy requests to internal services, JWT auth middleware, route definitions, and Zod validation schemas.

**services/auth-service**: Handles authentication. Contains Sequelize models for user credentials and refresh tokens, JWT token utilities, and a RabbitMQ publisher that emits events when users register.

**services/user-service**: Manages user profiles. Contains a RabbitMQ consumer that listens for auth events to sync user data, plus repositories and services for user CRUD operations.

**services/chat-service**: Handles messaging. Uses MongoDB for storing conversations and messages, Redis for caching, and follows a repository pattern for data access.

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the ISC License.
