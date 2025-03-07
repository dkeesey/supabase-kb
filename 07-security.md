# Supabase Security Best Practices

This document outlines security best practices when working with Supabase, with a focus on protecting your data and implementing proper access controls.

## Core Security Concepts

### Security Architecture

Supabase's security model consists of several layers:

1. **API Gateway**: Controls access to your project's endpoints
2. **Authentication**: Verifies user identity
3. **Row Level Security (RLS)**: Enforces data access rules
4. **Database Encryption**: Protects data at rest
5. **Network Security**: Secures data in transit

### API Keys

Supabase provides two types of API keys:

1. **`anon` (Public) Key**: Used in client applications
   - Limited permissions
   - Meant to be public
   - Access controlled by RLS policies

2. **`service_role` (Secret) Key**: Used for administrative tasks
   - Bypasses RLS policies
   - Has full database access
   - Must be kept secure and never exposed to clients

## Row Level Security (RLS)

Row Level Security is the cornerstone of Supabase's security model.

### RLS Principles

1. **Default deny**: Start with no access, then explicitly grant permissions
2. **Fine-grained control**: Policies apply at the row level, not just tables
3. **Context-aware**: Policies can use the current user's identity and other context
4. **Dynamic evaluation**: Policies are evaluated at query time

### Enabling RLS

```sql
-- Enable RLS on a table
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Make sure RLS is enforced for all users (including table owners)
ALTER TABLE profiles FORCE ROW LEVEL SECURITY;
```

### Policy Structure

```sql
CREATE POLICY policy_name
ON table_name
FOR operation
TO role
USING (using_expression)
WITH CHECK (check_expression);
```

Where:
- `policy_name`: Descriptive name for the policy
- `table_name`: The table this policy applies to
- `operation`: SELECT, INSERT, UPDATE, DELETE, or ALL
- `role`: Specific database role(s) or PUBLIC
- `using_expression`: Controls which rows are visible (for SELECT, UPDATE, DELETE)
- `check_expression`: Controls which values can be inserted/updated (for INSERT, UPDATE)

### Common RLS Patterns

#### Read-Only Access

```sql
-- Allow anyone to read all products
CREATE POLICY "Public read access for products"
ON products
FOR SELECT
USING (true);
```

#### Owner-Based Access

```sql
-- Users can only see their own data
CREATE POLICY "Users can see their own profiles"
ON profiles
FOR SELECT
USING (auth.uid() = id);

-- Users can update their own data
CREATE POLICY "Users can update their own profiles"
ON profiles
FOR UPDATE
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);
```

#### Role-Based Access

```sql
-- Create a function to check if a user has a specific role
CREATE OR REPLACE FUNCTION public.user_has_role(required_role TEXT)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM user_roles ur
    JOIN roles r ON ur.role_id = r.id
    WHERE ur.user_id = auth.uid() AND r.name = required_role
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Admin users can perform all operations
CREATE POLICY "Admin users have full access"
ON products
USING (user_has_role('admin'));

-- Moderator users can update but not delete
CREATE POLICY "Moderators can update products"
ON products
FOR UPDATE
USING (user_has_role('moderator'));
```

#### Tenant Isolation

```sql
-- Multi-tenant application where users can only access their organization's data
CREATE POLICY "Organization-based access"
ON organization_data
USING (
  EXISTS (
    SELECT 1
    FROM organization_members
    WHERE user_id = auth.uid() 
    AND organization_id = organization_data.organization_id
  )
);
```

#### Time-Based Policies

```sql
-- Users can only see active content
CREATE POLICY "Only show active content"
ON content
FOR SELECT
USING (
  published_at <= NOW() 
  AND (expires_at IS NULL OR expires_at > NOW())
);
```

#### Data-Based Policies

```sql
-- Users can only see public or their own private items
CREATE POLICY "Show public items or own private items"
ON items
FOR SELECT
USING (
  is_public = true 
  OR (is_public = false AND owner_id = auth.uid())
);
```

### Testing RLS Policies

Always test RLS policies from both authenticated and anonymous contexts:

```sql
-- Test as anonymous user
SELECT * FROM profiles;

-- Test as specific user
SELECT * FROM profiles;  -- After logging in as a specific user
```

You can also explicitly test with specific user IDs:

```sql
-- Set a specific user context for testing
SET LOCAL ROLE authenticated;
SET LOCAL request.jwt.claim.sub = 'user-uuid-here';

-- Run query to test policy
SELECT * FROM profiles;

-- Reset role
RESET ROLE;
```

## Secure Coding Patterns

### Using Security Definer Functions

When you need to perform operations with elevated privileges:

```sql
-- Create a function to perform privileged operations
CREATE OR REPLACE FUNCTION admin_update_user_role(
  user_id UUID,
  new_role TEXT
)
RETURNS BOOLEAN AS $$
BEGIN
  -- Check if the requesting user has admin privileges
  IF NOT user_has_role('admin') THEN
    RAISE EXCEPTION 'Permission denied';
  END IF;
  
  -- Perform the privileged operation
  INSERT INTO user_roles (user_id, role_id)
  SELECT user_id, id FROM roles WHERE name = new_role;
  
  RETURN TRUE;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

The `SECURITY DEFINER` attribute makes the function run with the privileges of the user who created it (typically the database owner), rather than the calling user.

### Edge Function Security

For operations requiring the `service_role` key:

```typescript
// In an Edge Function
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

// Create a Supabase client with the service role key
// This bypasses RLS policies
const supabaseAdmin = createClient(
  Deno.env.get('SUPABASE_URL') ?? '',
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
);

// Perform operations with elevated privileges
const { data, error } = await supabaseAdmin
  .from('private_table')
  .select('*');
```

Always verify user permissions in Edge Functions that use the `service_role` key:

```typescript
// First, verify the user's identity and permissions
const authHeader = req.headers.get('Authorization');
if (!authHeader) {
  return new Response(
    JSON.stringify({ error: 'Unauthorized' }),
    { status: 401, headers: { 'Content-Type': 'application/json' } }
  );
}

// Create a client with the user's Auth context
const supabaseClient = createClient(
  Deno.env.get('SUPABASE_URL') ?? '',
  Deno.env.get('SUPABASE_ANON_KEY') ?? '',
  { global: { headers: { Authorization: authHeader } } }
);

// Get the user's session
const { data: { user }, error } = await supabaseClient.auth.getUser();

if (error || !user) {
  return new Response(
    JSON.stringify({ error: 'Unauthorized' }),
    { status: 401, headers: { 'Content-Type': 'application/json' } }
  );
}

// Check user permissions (e.g., if they're an admin)
const { data: permissions, error: permissionsError } = await supabaseClient
  .from('user_roles')
  .select('roles(name)')
  .eq('user_id', user.id)
  .single();

const isAdmin = permissions?.roles?.name === 'admin';

if (!isAdmin) {
  return new Response(
    JSON.stringify({ error: 'Forbidden' }),
    { status: 403, headers: { 'Content-Type': 'application/json' } }
  );
}

// Only now perform privileged operations with the admin client
const { data, error: adminError } = await supabaseAdmin
  .from('private_table')
  .select('*');
```

### Client-Side Security

Never expose the `service_role` key in client-side code. Instead:

1. Use Edge Functions for privileged operations
2. Implement proper RLS policies
3. Use secure authentication methods

Frontend code should always use the `anon` key:

```javascript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  'https://your-project.supabase.co',
  'your-anon-key'
);
```

## Secrets Management

### Environment Variables

Store sensitive information in environment variables, not in code:

```typescript
// Edge Function
const apiKey = Deno.env.get('THIRD_PARTY_API_KEY');
```

### Supabase Secrets

Use Supabase's secrets management for Edge Functions:

```bash
# Set a secret
supabase secrets set MY_SECRET=my-secret-value

# Set multiple secrets
supabase secrets set MY_SECRET=my-secret-value OTHER_SECRET=other-value
```

Secrets can also be managed through the Supabase dashboard under:
Settings > API > Edge Functions > Secrets

### Handling API Keys

For integrating with third-party services:

1. Store API keys as Supabase secrets
2. Access services only from Edge Functions, never from the client
3. Implement rate limiting to prevent abuse

```typescript
// Edge Function for OpenAI integration
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';

serve(async (req) => {
  try {
    const { prompt } = await req.json();
    
    // Get the secret API key
    const apiKey = Deno.env.get('OPENAI_API_KEY');
    
    // Call the OpenAI API
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }]
      })
    });
    
    const data = await response.json();
    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' }
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
});
```

## Data Security

### Handling Sensitive Data

1. **Encrypting Sensitive Fields**

```sql
-- Create an extension for encryption
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Create a function to encrypt data
CREATE OR REPLACE FUNCTION encrypt_data(data TEXT, key TEXT)
RETURNS TEXT AS $$
BEGIN
  RETURN encode(
    pgp_sym_encrypt(
      data,
      key,
      'cipher-algo=aes256'
    ),
    'base64'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create a function to decrypt data
CREATE OR REPLACE FUNCTION decrypt_data(encrypted_data TEXT, key TEXT)
RETURNS TEXT AS $$
BEGIN
  RETURN pgp_sym_decrypt(
    decode(encrypted_data, 'base64'),
    key,
    'cipher-algo=aes256'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Example usage
INSERT INTO users (id, encrypted_ssn)
VALUES (
  '123e4567-e89b-12d3-a456-426614174000',
  encrypt_data('123-45-6789', current_setting('app.encryption_key'))
);
```

2. **Masking Data in Responses**

```sql
-- Create a view that masks sensitive data
CREATE VIEW public_users AS
SELECT 
  id,
  email,
  CASE 
    WHEN auth.uid() = id OR user_has_role('admin') THEN phone_number
    ELSE CONCAT(LEFT(phone_number, 3), '****', RIGHT(phone_number, 4))
  END AS phone_number,
  created_at
FROM users;
```

### Audit Logging

Track important security events:

```sql
-- Create an audit log table
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  action TEXT NOT NULL,
  table_name TEXT NOT NULL,
  record_id UUID,
  old_data JSONB,
  new_data JSONB,
  ip_address TEXT,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Only admins can read audit logs
CREATE POLICY "Only admins can view audit logs"
ON audit_logs
FOR SELECT
USING (user_has_role('admin'));

-- Create a function to log changes
CREATE OR REPLACE FUNCTION log_audit_event()
RETURNS TRIGGER AS $$
DECLARE
  client_info JSONB;
BEGIN
  -- Get client info from Supabase context
  client_info := current_setting('request.jwt.claims', true)::jsonb;
  
  INSERT INTO audit_logs (
    user_id,
    action,
    table_name,
    record_id,
    old_data,
    new_data,
    ip_address,
    user_agent
  )
  VALUES (
    CASE WHEN auth.uid() IS NOT NULL THEN auth.uid() ELSE NULL END,
    TG_OP,
    TG_TABLE_NAME,
    CASE 
      WHEN TG_OP = 'DELETE' THEN OLD.id
      ELSE NEW.id
    END,
    CASE WHEN TG_OP = 'INSERT' THEN NULL ELSE row_to_json(OLD) END,
    CASE WHEN TG_OP = 'DELETE' THEN NULL ELSE row_to_json(NEW) END,
    client_info->'sub'->'ip'::TEXT,
    current_setting('request.headers', true)::jsonb->>'user-agent'
  );
  
  RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Add a trigger to a table
CREATE TRIGGER products_audit_log
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE PROCEDURE log_audit_event();
```

## WebAuthn and 2FA Security

For applications requiring higher security:

```sql
-- Enable MFA for all users
UPDATE auth.config
SET enable_mfa = true;

-- Create a trigger to enforce MFA for specific user roles
CREATE OR REPLACE FUNCTION enforce_mfa_for_admins()
RETURNS TRIGGER AS $$
BEGIN
  -- Check if the user has admin role but no MFA
  IF EXISTS (
    SELECT 1
    FROM user_roles ur
    JOIN roles r ON ur.role_id = r.id
    WHERE ur.user_id = NEW.id AND r.name = 'admin'
  ) AND NOT EXISTS (
    SELECT 1
    FROM auth.mfa_factors
    WHERE user_id = NEW.id AND status = 'verified'
  ) THEN
    -- Force MFA enrollment on next login
    UPDATE auth.users
    SET require_mfa = true
    WHERE id = NEW.id;
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER enforce_admin_mfa
AFTER INSERT OR UPDATE ON auth.users
FOR EACH ROW EXECUTE PROCEDURE enforce_mfa_for_admins();
```

## Security Best Practices Checklist

### Database Security

- [ ] Enable RLS on all tables
- [ ] Implement proper RLS policies for each table
- [ ] Use `SECURITY DEFINER` functions for privileged operations
- [ ] Create views to mask sensitive data
- [ ] Implement audit logging for sensitive operations
- [ ] Use encryption for sensitive fields
- [ ] Regularly review and test RLS policies

### API and Service Security

- [ ] Never expose the `service_role` key in client code
- [ ] Use Edge Functions for operations requiring elevated privileges
- [ ] Implement proper authentication checks in Edge Functions
- [ ] Store all secrets using Supabase secrets management
- [ ] Implement rate limiting for API endpoints
- [ ] Validate and sanitize all user inputs
- [ ] Use parameterized queries to prevent SQL injection

### Authentication Security

- [ ] Configure proper password policies
- [ ] Implement MFA for sensitive operations or admin roles
- [ ] Set up proper email verification and recovery workflows
- [ ] Use JWT tokens with appropriate expiration times
- [ ] Implement proper session management
- [ ] Create secure roles and permissions system

### Client-Side Security

- [ ] Only use the `anon` key in client applications
- [ ] Implement proper client-side validation
- [ ] Handle authentication state securely
- [ ] Don't store sensitive information in localStorage
- [ ] Use HTTPS for all API communications
- [ ] Implement proper CORS settings

## Handling Security Incidents

If a security incident occurs:

1. **Rotate API Keys**: Immediately generate new API keys in the Supabase dashboard
2. **Audit Access Logs**: Review database and authentication logs
3. **Review RLS Policies**: Check for any misconfigurations
4. **Reset User Passwords**: Force password resets if necessary
5. **Update Vulnerable Code**: Fix any security issues in your application
6. **Contact Supabase Support**: For assistance with serious incidents

## Common Security Pitfalls

1. **Using `service_role` key in client code**:
   - Never expose this key in frontend applications
   - Use Edge Functions instead

2. **Missing RLS policies**:
   - Always enable RLS on all tables
   - Start with restrictive policies and open access as needed

3. **Inadequate permission checks**:
   - Always verify permissions before performing privileged operations
   - Use role-based checks, not just authentication

4. **Insecure direct object references**:
   - Don't use predictable IDs or allow users to manipulate IDs
   - Always verify ownership through RLS

5. **SQL injection vulnerabilities**:
   - Use parameterized queries
   - Avoid string concatenation in SQL

6. **Insufficient logging**:
   - Implement comprehensive audit logging
   - Monitor for suspicious activities

7. **Not encrypting sensitive data**:
   - Encrypt personal or financial information
   - Use proper key management

8. **Relying solely on client-side validation**:
   - Always validate data on the server side
   - Treat all client input as untrusted

By following these security best practices, you can build secure applications with Supabase that protect your users' data and maintain the integrity of your system.
