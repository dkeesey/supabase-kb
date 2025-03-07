# Model Context Protocol (MCP) Configuration for Supabase

This guide covers how to configure Supabase with various AI coding assistants using Model Context Protocol (MCP).

## What is Model Context Protocol (MCP)?

MCP is a protocol that enables AI coding assistants to access your development environment, including files, directories, and code. When properly configured, AI tools like Claude Desktop, Cursor, or Windsurf can:

1. Read your Supabase project files
2. Execute Supabase CLI commands
3. Create and modify Supabase configuration files
4. Access your Supabase databases for context

## MCP Configuration for Supabase in Different Tools

### Claude Desktop Configuration

Claude Desktop allows you to connect to Supabase through its MCP interface. Here's how to set it up:

1. Open Claude Desktop settings
2. Navigate to the "Connections" tab
3. Add a new connection for Supabase
4. Configure the connection with these details:

```json
{
  "name": "supabase",
  "type": "filesystem",
  "paths": [
    "/path/to/your/supabase/project"
  ],
  "capabilities": [
    "read",
    "write"
  ]
}
```

Additionally, you can set up the Supabase CLI connection:

```json
{
  "name": "supabase-cli",
  "type": "terminal",
  "working_directory": "/path/to/your/supabase/project",
  "capabilities": [
    "execute"
  ]
}
```

### Cursor Configuration

Cursor has built-in support for Supabase projects. To use it effectively:

1. Open your Supabase project in Cursor
2. Enable AI features in Cursor settings
3. Set up a .cursorignore file to exclude node_modules and other large directories
4. Make sure Cursor has the proper permissions to access your project directory

For enhanced functionality, add Supabase-specific context to Cursor:

1. Go to Settings > AI > Custom Context
2. Add Supabase context:

```
You are working on a Supabase project. The project uses PostgreSQL for the database, 
with Row Level Security (RLS) policies to control access to data. The project also 
uses Supabase Auth for authentication and Edge Functions for serverless API endpoints.

When generating database code, follow PostgreSQL best practices and always consider 
security implications like SQL injection and proper RLS policies.

When working with Edge Functions, use TypeScript and follow the Deno runtime conventions.
```

### Windsurf Configuration

To set up Supabase in Windsurf:

1. Open Windsurf and go to Settings
2. Navigate to Integrations > File System
3. Add your Supabase project directory
4. Enable read and write permissions

Additionally, you can add terminal access:

1. Go to Integrations > Terminal
2. Enable terminal access with appropriate permissions
3. Set the default working directory to your Supabase project

## General MCP Guidelines for Supabase

For the best experience with any MCP-enabled AI assistant working with Supabase:

1. **Organize Your Project**: Keep a well-structured project with clear separation of concerns:
   - `/supabase`: For Supabase configuration
   - `/components`: For UI components
   - `/lib/supabase`: For client-side Supabase utilities

2. **Include Documentation Comments**: Add comments to your code explaining Supabase-specific patterns:

```typescript
/**
 * Supabase client configuration
 * Uses environment variables for URLs and keys
 * Avoids exposing service role key on the client side
 */
export const supabaseClient = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

3. **Create Reference Files**: Add files like `supabase-types.ts` that define your database schema:

```typescript
/**
 * Database schema types
 * Generated using Supabase CLI:
 * supabase gen types typescript --local > lib/supabase/types.ts
 */
export type Database = {
  public: {
    Tables: {
      profiles: {
        Row: {
          id: string;
          username: string;
          avatar_url: string | null;
          created_at: string;
        };
        Insert: {
          id: string;
          username: string;
          avatar_url?: string | null;
          created_at?: string;
        };
        Update: {
          id?: string;
          username?: string;
          avatar_url?: string | null;
          created_at?: string;
        };
      };
      // Other tables...
    };
    // Views, functions, etc...
  };
};
```

4. **Include Local Supabase Setup**: Add a README or documentation on how to set up Supabase locally:

```markdown
## Local Supabase Development

1. Install Supabase CLI: `npm install -g supabase`
2. Start local Supabase: `supabase start`
3. Generate types: `supabase gen types typescript --local > lib/supabase/types.ts`
4. Apply migrations: `supabase db reset`
5. Stop local Supabase: `supabase stop`
```

## Using MCP Effectively with Supabase

### Best Practices for AI-Assisted Supabase Development

1. **Provide Context**: When asking about a specific Supabase feature, reference relevant files or code
2. **Specify Version Information**: Mention which version of Supabase you're using
3. **Reference Documentation**: Link to official Supabase documentation for complex features
4. **Show Examples**: Include examples of what you're trying to accomplish
5. **Be Specific About Requirements**: Clearly explain security, performance, and functional requirements

### Example AI Prompts with MCP

When working with an MCP-enabled AI assistant, you can reference files and directories:

#### Analyzing Database Schema

```
Please analyze my Supabase schema in `/supabase/migrations` and suggest improvements for performance and security.
```

#### Enhancing RLS Policies

```
Review the RLS policies in `/supabase/migrations/20230501_create_posts_table.sql` and suggest how to improve them to ensure users can only access their own data.
```

#### Debugging Edge Functions

```
I'm getting this error with my Edge Function in `/supabase/functions/process-payment/index.ts`: [paste error]. Can you help me fix it?
```

#### Generating TypeScript Types

```
Based on my database schema in `/supabase/migrations`, can you generate TypeScript types for my Supabase tables?
```

## Troubleshooting MCP with Supabase

If you encounter issues with MCP and Supabase:

1. **Check Permissions**: Ensure the AI tool has proper read/write access to your project directory
2. **Verify CLI Installation**: Make sure Supabase CLI is installed and in your PATH
3. **Check Configuration Files**: Ensure `.env` files are properly set up with Supabase credentials
4. **Limit Context Size**: Large Supabase projects may exceed context limits; focus on specific files
5. **Update Tools**: Keep your AI tools and Supabase CLI updated to the latest versions

By properly configuring MCP for your preferred AI coding assistant, you can greatly enhance your productivity when working with Supabase projects.
