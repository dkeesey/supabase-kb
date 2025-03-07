# Supabase Knowledge Base

This knowledge base provides a comprehensive guide to Supabase, focusing on practical implementation patterns and best practices for developers. It's designed to be a reference when working with AI coding assistants like Claude Desktop, Cursor, or Windsurf.

## Official Resources

- [Supabase Documentation](https://supabase.com/docs)
- [Supabase GitHub](https://github.com/supabase/supabase)
- [Supabase Blog](https://supabase.com/blog)
- [Supabase Dashboard](https://app.supabase.com/)

## Contents

1. [Core Concepts](01-core-concepts.md) - Overview of Supabase architecture and components
2. [Authentication](02-authentication.md) - Implementing user authentication and authorization
3. [Database](03-database.md) - PostgreSQL database management and best practices
4. [Edge Functions](04-edge-functions.md) - Serverless function development and deployment
5. [Storage](05-storage.md) - File storage implementation and security
6. [Realtime](06-realtime.md) - Real-time data synchronization
7. [Security Best Practices](07-security.md) - Row Level Security and other security considerations
8. [Common Integration Patterns](08-integration-patterns.md) - Patterns for integrating with third-party services
9. [AI Prompts Guide](ai-prompts-guide.md) - Effective prompts for Supabase development with AI assistants
10. [MCP Configuration](mcp-configuration.md) - Setting up Model Context Protocol for Cursor, Windsurf, and Claude Desktop
11. [Performance Optimization](09-performance.md) - Database and API performance tuning
12. [Deployment and CI/CD](10-deployment.md) - Setting up deployment workflows
13. [Troubleshooting](11-troubleshooting.md) - Common issues and their solutions

## How to Use This Knowledge Base

This knowledge base is designed to be both a learning resource and a practical reference. Each section contains:

- Conceptual overviews
- Code examples
- Implementation patterns
- Best practices
- Common pitfalls to avoid

When working with AI coding assistants, you can reference specific sections of this knowledge base to provide context for your questions or to guide the AI in generating accurate, best-practice code for Supabase-related functionality.

## Getting Started

If you're new to Supabase, start with the [Core Concepts](01-core-concepts.md) guide to understand the fundamental architecture and capabilities. Then, explore specific areas based on your immediate needs.

For setting up a new project, follow this general sequence:

1. Read [Core Concepts](01-core-concepts.md)
2. Set up [Authentication](02-authentication.md)
3. Design your [Database](03-database.md) schema
4. Implement [Security Best Practices](07-security.md)
5. Add additional features as needed (Storage, Edge Functions, Realtime)

## Supabase CLI Installation and Common Commands

### Installation (macOS)

```bash
# Install via Homebrew
brew install supabase/tap/supabase

# Verify installation
supabase --version
```

### Local Development Commands

```bash
# Initialize a new project
supabase init

# Start Supabase locally
supabase start

# Check status of local services
supabase status

# Stop local services
supabase stop
```

### Database Management Commands

```bash
# Create a new migration
supabase migration new my_migration_name

# Apply migrations
supabase db reset

# Generate types based on your schema
supabase gen types typescript --local > types/supabase.ts
```

### Edge Functions Commands

```bash
# Create a new Edge Function
supabase functions new my-function-name

# Deploy an Edge Function to production
supabase functions deploy my-function-name

# Serve Edge Functions locally for testing
supabase functions serve
```

### Project Management Commands

```bash
# Link to a remote project
supabase link --project-ref your-project-ref

# Get project configuration
supabase config get

# List all remote projects
supabase projects list
```

### Secrets Management Commands

```bash
# Set secret for Edge Functions
supabase secrets set MY_SECRET=value

# List all secrets
supabase secrets list
```
