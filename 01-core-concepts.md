# Supabase Core Concepts

## What is Supabase?

Supabase is an open-source Firebase alternative that provides a suite of tools for building applications. At its core, Supabase is a collection of open-source products working together:

- **PostgreSQL Database**: A powerful, open-source relational database
- **Authentication**: User management and authentication system
- **Storage**: File storage and management
- **Edge Functions**: Serverless function execution
- **Realtime**: Real-time subscriptions and broadcasts
- **Vector**: AI vector embeddings and similarity search

Unlike many "Backend as a Service" (BaaS) offerings, Supabase is built on open-source technologies, with PostgreSQL as its foundation rather than a proprietary NoSQL database.

## Architecture Overview

Supabase follows a client-server architecture:

```
Client Application <---> Supabase Platform <---> PostgreSQL Database
```

The Supabase platform consists of several microservices that handle specific aspects of functionality:

1. **PostgreSQL**: The core database that stores all your data
2. **PostgREST**: Converts your PostgreSQL database into a RESTful API
3. **GoTrue**: Handles user authentication and management
4. **Storage API**: Manages file uploads and serving
5. **Realtime**: Manages WebSocket connections for real-time updates
6. **Edge Functions**: Executes serverless functions in a Deno runtime
7. **pg_graphql**: Provides GraphQL API access to your database
8. **Vector**: Manages vector embeddings for AI applications

## Key Concepts

### Projects

A Supabase Project is a isolated environment that contains your database, authentication rules, storage buckets, and edge functions. Each project has:

- A unique URL (e.g., `https://xyzproject.supabase.co`)
- API keys for authentication
- A dedicated PostgreSQL database
- Independent configuration settings

### Authentication

Supabase Authentication (based on GoTrue) provides:

- Email/password authentication
- Social logins (Google, GitHub, etc.)
- Phone authentication
- Magic link authentication
- JWT-based sessions
- Role-based access control

### Database

The Supabase database is PostgreSQL with additional extensions:

- **Tables**: Store your structured data
- **Views**: Virtual tables defined by queries
- **Functions**: Custom SQL or JavaScript functions
- **Triggers**: Automatic responses to database events
- **Policies**: Row-level security rules
- **Extensions**: Add additional PostgreSQL functionality

### APIs

Supabase automatically generates multiple API endpoints:

- **REST API**: Generated from your database schema
- **GraphQL API**: Optional GraphQL interface to your data
- **Realtime API**: WebSocket-based API for subscribing to changes
- **Storage API**: For file operations
- **Edge Functions API**: For invoking serverless functions

### Security

Security in Supabase is primarily handled through:

1. **API Keys**: Control access to your project APIs
   - `anon` key: For unauthenticated public access
   - `service_role` key: For administrative access (use with caution)

2. **Row Level Security (RLS)**: Policies that control access to database rows
   - Define which users can read/write specific rows
   - Create complex access rules using SQL conditions

3. **Auth Policies**: Control user authentication and session behavior

## Client Libraries

Supabase provides official client libraries for:

- JavaScript/TypeScript
- Flutter/Dart
- Python
- Swift
- Kotlin
- C#
- Go
- Java
- Ruby
- Rust
- PHP

The client libraries abstract away the complexity of working with the underlying APIs, providing a simple, consistent interface for your applications.

## Example: Basic Architecture

Here's how these concepts work together in a typical application:

1. **Authentication**: Users sign in through Supabase Auth
2. **JWT Token**: Successful auth provides a JWT token
3. **Database Access**: The token enables access to database rows according to RLS policies
4. **API Requests**: Client makes API requests with the JWT token
5. **RLS Enforcement**: PostgreSQL enforces access control based on the token
6. **Response**: Data is returned based on the user's permissions

## Typical Project Setup Flow

Setting up a new Supabase project typically follows this process:

1. Create a new project in the Supabase dashboard
2. Design and create your database schema
3. Configure authentication providers
4. Set up Row Level Security policies
5. Create storage buckets (if needed)
6. Deploy Edge Functions (if needed)
7. Connect your client application using a client library

## Local Development

Supabase can be run locally for development using:

- **Supabase CLI**: Command-line tool for local development
- **Docker**: Container-based local development environment

Local development allows you to:
- Develop without internet connectivity
- Test changes before deploying to production
- Set up CI/CD pipelines with database migrations

## Key Differences from Firebase

If you're familiar with Firebase, these are the key differences:

1. **Relational Database**: PostgreSQL vs. Firebase's NoSQL
2. **SQL**: Full SQL capabilities vs. Firebase's query limitations
3. **Open Source**: All components are open source
4. **Client Flexibility**: Direct database queries are possible
5. **Deployment Options**: Self-hosting is possible
6. **Pricing Model**: Resource-based rather than operation-based

## Limitations and Considerations

- **Cold Starts**: Edge Functions may experience cold starts
- **Database Size Limits**: Tier-based limits on database size
- **Connection Limits**: Tier-based limits on concurrent connections
- **Bandwidth Limits**: Tier-based limits on egress bandwidth
- **Regional Deployment**: Projects are deployed to specific regions

Understanding these core concepts will provide a solid foundation for working with Supabase across all its features.
