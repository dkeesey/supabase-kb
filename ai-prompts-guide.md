# Supabase AI Prompts Guide

This document contains effective prompts for working with Supabase using AI coding assistants such as Claude, ChatGPT, Cursor, or Windsurf.

## General Supabase Prompts

These prompts help you generate standard Supabase implementations:

### Bootstrap a Next.js App with Supabase Auth

```
Create a new Next.js 14 app with Supabase authentication. Include:
1. Email/password and social authentication
2. Protected routes
3. User profile management
4. Proper TypeScript types
5. Environment setup instructions
```

### Create a Database Schema with Relationships

```
Design a Supabase database schema for a [describe your application type, e.g., "task management app"]. Include:
1. Users table extending auth.users
2. [List your main data tables]
3. Proper relationships and foreign keys
4. Timestamp fields for created_at and updated_at
5. Appropriate indexes for performance
6. RLS policies for each table
```

### Set Up Row Level Security (RLS)

```
Write RLS policies for a Supabase [table name] table with these requirements:
1. Users can only view their own records
2. Admin users can view all records
3. Users can only update their own records
4. Records are automatically assigned to the authenticated user on insert
```

### Create a PostgreSQL Function

```
Write a PostgreSQL function for Supabase that [describe what the function should do]. The function should:
1. Accept parameters for [list parameters]
2. Return [describe return value]
3. Include proper error handling
4. Be secure with SECURITY DEFINER/INVOKER as appropriate
5. Include comments explaining the logic
```

### Generate a Database Migration

```
Create a Supabase migration script to:
1. Add a new table called [table name] with fields [list fields and types]
2. Add foreign key relationships to [related tables]
3. Create RLS policies for the table
4. Add indexes for [list columns that need indexing]
5. Insert initial seed data
```

## Edge Functions Prompts

### Create a Basic Edge Function

```
Write a Supabase Edge Function that [describe what the function should do]. Include:
1. Proper error handling
2. Authentication checks
3. CORS configuration
4. TypeScript types
5. Comments explaining key parts of the code
```

### Create an OpenAI Integration Edge Function

```
Create a Supabase Edge Function that integrates with OpenAI API to [describe functionality]. The function should:
1. Securely handle API keys using environment variables
2. Validate user authentication and permissions
3. Handle API rate limits and errors gracefully
4. Format and return responses consistently
5. Include TypeScript types
```

### Create a Stripe Payment Integration

```
Write a Supabase Edge Function for processing payments with Stripe. Include:
1. Creating checkout sessions
2. Handling webhooks securely
3. Updating user subscription status in the database
4. Error handling and logging
5. TypeScript types for all operations
```

### Create an Edge Function for File Processing

```
Create a Supabase Edge Function that processes file uploads for [describe purpose]. The function should:
1. Validate file types and sizes
2. Process the file using [describe processing]
3. Store metadata in the database
4. Return appropriate success/error responses
4. Include proper TypeScript types
```

## Advanced Prompts

### Full-Stack Feature Implementation

```
Implement a complete [feature name] feature using Next.js frontend and Supabase backend. Include:
1. Database schema changes
2. RLS policies
3. Client-side data fetching and caching
4. React components with proper state management
5. Form validation and error handling
6. Unit tests for key functionality
```

### Implement Real-Time Features

```
Create a real-time [feature name] using Supabase's real-time capabilities. Include:
1. Database setup with needed tables and RLS
2. Client-side subscription setup
3. UI components that react to real-time updates
4. Proper error handling and reconnection logic
5. Performance optimization considerations
```

### Implement Vector Search

```
Set up pgvector in Supabase for [describe use case]. Include:
1. Vector column setup
2. Indexing configuration
3. Generation of embeddings using [model/technique]
4. Search functions with similarity metrics
5. Client-side integration to display results
```

## Style Guides

### PostgreSQL SQL Style Guide

Follow these best practices when writing SQL for Supabase:

#### Naming Conventions
- Use snake_case for all identifiers (tables, columns, functions)
- Use plural for table names (users, posts, comments)
- Use singular for column names (user_id, post_id)
- Prefix primary keys with the table name (e.g., user_id in users table)
- Use consistent prefixes for foreign keys (e.g., user_id, post_id)

#### Formatting
- Capitalize SQL keywords (SELECT, INSERT, UPDATE)
- Use 2 or 4 spaces for indentation (be consistent)
- Place each column on its own line for readability
- Left-align keywords

#### Example:

```sql
SELECT
  users.id AS user_id,
  users.name AS user_name,
  posts.title AS post_title,
  posts.created_at AS post_created_at
FROM
  users
LEFT JOIN
  posts ON users.id = posts.user_id
WHERE
  users.is_active = true
  AND posts.created_at > NOW() - INTERVAL '7 days'
ORDER BY
  posts.created_at DESC
LIMIT 10;
```

#### Functions and Procedures
- Include parameter types and return types
- Add comments explaining purpose and usage
- Handle errors gracefully
- Validate inputs
- Use appropriate security context (SECURITY DEFINER vs INVOKER)

#### RLS Policies
- Use descriptive names that indicate what the policy allows
- Create separate policies for different operations
- Keep policy expressions as simple as possible
- Test policies thoroughly

Remember, in Supabase, all SQL runs on PostgreSQL, so you have access to all PostgreSQL features and syntax.
