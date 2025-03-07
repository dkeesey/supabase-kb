# Supabase Edge Functions

Edge Functions are serverless TypeScript functions that run on Supabase's global edge network. They provide a way to execute server-side code for your applications without managing servers.

## Key Concepts

### What Are Edge Functions?

Edge Functions:
- Are written in TypeScript/JavaScript
- Run in a Deno runtime environment
- Deploy globally to edge locations
- Have access to the full Deno standard library
- Can directly access your Supabase database and auth

### When to Use Edge Functions

Use Edge Functions when you need to:

1. **Process sensitive data**: Handle API keys and secrets securely
2. **Integrate with third-party APIs**: Connect to external services
3. **Transform data**: Process data before storing or returning
4. **Implement custom logic**: Execute complex business logic
5. **Extend Supabase functionality**: Add capabilities beyond built-in features
6. **Create webhooks**: Handle incoming webhook requests

## Creating and Deploying Edge Functions

### Prerequisites

Before working with Edge Functions, you need:

1. **Supabase CLI**: Install with `npm install -g supabase`
2. **Project Linking**: Link to your remote project with `supabase link --project-ref your-project-ref`

### Creating a New Function

```bash
supabase functions new my-function-name
```

This creates a new folder in the `supabase/functions/` directory with a `index.ts` file:

```typescript
// supabase/functions/my-function-name/index.ts

import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  const { name } = await req.json()
  const data = { message: `Hello ${name || 'World'}!` }

  return new Response(
    JSON.stringify(data),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

### Testing Locally

You can test functions locally with:

```bash
supabase functions serve my-function-name
```

This runs a local development server where you can test your function before deployment.

### Deploying the Function

```bash
supabase functions deploy my-function-name
```

This deploys your function to the Supabase platform, making it accessible at:

```
https://<project-ref>.supabase.co/functions/v1/my-function-name
```

## Anatomy of an Edge Function

### Basic Structure

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  // 1. Parse the request
  const { data } = await req.json()
  
  // 2. Process the data
  const result = await processData(data)
  
  // 3. Return a response
  return new Response(
    JSON.stringify({ result }),
    { headers: { 'Content-Type': 'application/json' } }
  )
})

async function processData(data) {
  // Your business logic here
  return { processed: data }
}
```

### Handling Different HTTP Methods

```typescript
serve(async (req) => {
  // Get the HTTP method
  const method = req.method
  
  switch (method) {
    case 'GET':
      return handleGet(req)
    case 'POST':
      return handlePost(req)
    case 'PUT':
      return handlePut(req)
    case 'DELETE':
      return handleDelete(req)
    default:
      return new Response(
        JSON.stringify({ error: 'Method not allowed' }),
        { status: 405, headers: { 'Content-Type': 'application/json' } }
      )
  }
})

async function handleGet(req) {
  // Handle GET request
}

async function handlePost(req) {
  // Handle POST request
}

// etc.
```

### Handling Query Parameters

```typescript
serve(async (req) => {
  // Get URL from request
  const url = new URL(req.url)
  
  // Get query parameters
  const name = url.searchParams.get('name')
  const limit = parseInt(url.searchParams.get('limit') || '10')
  
  // Process with parameters
  const data = await getData(name, limit)
  
  return new Response(
    JSON.stringify(data),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

### Handling Path Parameters

```typescript
serve(async (req) => {
  const url = new URL(req.url)
  const path = url.pathname
  
  // Extract path parameters
  // For example, if path is /functions/v1/users/123
  const pathParts = path.split('/')
  const resourceId = pathParts[pathParts.length - 1]
  
  // Use the resourceId
  const data = await getResourceById(resourceId)
  
  return new Response(
    JSON.stringify(data),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

## Connecting to Supabase Services

### Accessing the Database

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Create a Supabase client with the project URL and anon key
  const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
  const supabaseKey = Deno.env.get('SUPABASE_ANON_KEY') || ''
  const supabase = createClient(supabaseUrl, supabaseKey)
  
  // Query the database
  const { data, error } = await supabase
    .from('table_name')
    .select('*')
    .limit(10)
  
  if (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
  
  return new Response(
    JSON.stringify({ data }),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

### Using Authentication in Edge Functions

#### Admin-level Access

When you need to bypass RLS policies:

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Create a Supabase client with the service role key (admin privileges)
  const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
  const supabaseServiceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || ''
  const supabase = createClient(supabaseUrl, supabaseServiceKey)
  
  // This query will bypass RLS
  const { data, error } = await supabase
    .from('private_table')
    .select('*')
  
  // Handle response...
})
```

#### Using the User's Auth Context

To maintain RLS policies:

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Get the JWT from the Authorization header
  const authHeader = req.headers.get('Authorization')
  
  if (!authHeader) {
    return new Response(
      JSON.stringify({ error: 'No authorization header' }),
      { status: 401, headers: { 'Content-Type': 'application/json' } }
    )
  }
  
  // Create a Supabase client with the Authorization header
  const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
  const supabaseAnonKey = Deno.env.get('SUPABASE_ANON_KEY') || ''
  const supabase = createClient(supabaseUrl, supabaseAnonKey, {
    global: {
      headers: {
        Authorization: authHeader
      }
    }
  })
  
  // This query will respect RLS policies
  const { data, error } = await supabase
    .from('protected_table')
    .select('*')
  
  // Handle response...
})
```

### Working with Storage

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Create a Supabase client
  const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
  const supabaseServiceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || ''
  const supabase = createClient(supabaseUrl, supabaseServiceKey)
  
  // Generate a signed URL for a private file
  const { data, error } = await supabase
    .storage
    .from('private-bucket')
    .createSignedUrl('path/to/file.pdf', 60) // URL valid for 60 seconds
  
  if (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
  
  return new Response(
    JSON.stringify({ url: data.signedUrl }),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

## Working with Secrets and Environment Variables

### Setting Secrets

To set secrets for your Edge Functions:

```bash
supabase secrets set MY_SECRET=mysecretvalue OTHER_SECRET=anothervalue
```

Or via the Supabase dashboard under Settings > API > Edge Functions.

### Accessing Secrets

```typescript
serve(async (req) => {
  // Access the secret
  const mySecret = Deno.env.get('MY_SECRET')
  
  // Use the secret (e.g., to authenticate with an external API)
  const apiResponse = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${mySecret}`
    }
  })
  
  // Process and return response...
})
```

## Common Integration Patterns

### Integrating with OpenAI (or Azure OpenAI)

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  try {
    const { prompt } = await req.json()
    
    // Get API key from environment variable
    const apiKey = Deno.env.get('OPENAI_API_KEY')
    
    // Call OpenAI API
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 500
      })
    })
    
    const data = await response.json()
    
    return new Response(
      JSON.stringify({ 
        content: data.choices[0].message.content,
        usage: data.usage
      }),
      { headers: { 'Content-Type': 'application/json' } }
    )
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

### Azure OpenAI Integration

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  try {
    const { prompt } = await req.json()
    
    // Get Azure OpenAI credentials from environment variables
    const apiKey = Deno.env.get('AZURE_OPENAI_API_KEY')
    const endpoint = Deno.env.get('AZURE_OPENAI_ENDPOINT')
    const deploymentName = Deno.env.get('AZURE_OPENAI_DEPLOYMENT_NAME')
    const apiVersion = '2023-05-15' // Update to current version
    
    // Call Azure OpenAI API
    const url = `${endpoint}/openai/deployments/${deploymentName}/chat/completions?api-version=${apiVersion}`
    
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'api-key': apiKey
      },
      body: JSON.stringify({
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 500
      })
    })
    
    const data = await response.json()
    
    return new Response(
      JSON.stringify({
        content: data.choices[0].message.content,
        usage: data.usage
      }),
      { headers: { 'Content-Type': 'application/json' } }
    )
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

### Processing Stripe Webhooks

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
import Stripe from 'https://esm.sh/stripe@12.5.0?target=deno'

serve(async (req) => {
  // Get Stripe webhook secret
  const stripeWebhookSecret = Deno.env.get('STRIPE_WEBHOOK_SECRET')
  
  // Get the stripe signature from the headers
  const signature = req.headers.get('stripe-signature')
  
  if (!signature) {
    return new Response(
      JSON.stringify({ error: 'No signature provided' }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    )
  }
  
  try {
    // Get the raw request body
    const body = await req.text()
    
    // Initialize Stripe
    const stripe = new Stripe(Deno.env.get('STRIPE_SECRET_KEY'), {
      apiVersion: '2023-08-16', // Update to current version
    })
    
    // Verify and construct the webhook event
    const event = stripe.webhooks.constructEvent(
      body,
      signature,
      stripeWebhookSecret
    )
    
    // Initialize Supabase client
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
    const supabaseServiceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || ''
    const supabase = createClient(supabaseUrl, supabaseServiceKey)
    
    // Handle different event types
    switch (event.type) {
      case 'checkout.session.completed':
        // Handle successful payment
        const session = event.data.object
        
        // Update user subscription status
        await supabase
          .from('subscriptions')
          .upsert({
            user_id: session.client_reference_id,
            stripe_customer_id: session.customer,
            subscription_id: session.subscription,
            status: 'active',
            plan: session.metadata.plan,
            current_period_end: new Date(session.subscription_data.trial_end * 1000).toISOString()
          })
        
        break
        
      case 'customer.subscription.updated':
        // Handle subscription update
        const subscription = event.data.object
        
        // Update subscription in database
        // ...
        
        break
        
      case 'customer.subscription.deleted':
        // Handle subscription cancellation
        // ...
        
        break
        
      default:
        console.log(`Unhandled event type: ${event.type}`)
    }
    
    return new Response(
      JSON.stringify({ received: true }),
      { headers: { 'Content-Type': 'application/json' } }
    )
    
  } catch (error) {
    console.error('Error:', error.message)
    return new Response(
      JSON.stringify({ error: `Webhook error: ${error.message}` }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

### Sending Emails with Resend

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  try {
    const { to, subject, content } = await req.json()
    
    // Get Resend API key
    const resendApiKey = Deno.env.get('RESEND_API_KEY')
    
    // Call Resend API to send email
    const response = await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${resendApiKey}`
      },
      body: JSON.stringify({
        from: 'Your App <onboarding@resend.dev>',
        to: to,
        subject: subject,
        html: content
      })
    })
    
    const data = await response.json()
    
    return new Response(
      JSON.stringify({ data }),
      { headers: { 'Content-Type': 'application/json' } }
    )
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

## Advanced Techniques

### CORS Handling

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

// Define CORS headers
const corsHeaders = {
  'Access-Control-Allow-Origin': '*', // Or specific origin
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
  'Access-Control-Max-Age': '86400',
}

serve(async (req) => {
  // Handle CORS preflight requests
  if (req.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders })
  }
  
  try {
    // Process the request...
    const data = { message: 'Success' }
    
    // Return response with CORS headers
    return new Response(
      JSON.stringify(data),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { 
        status: 500, 
        headers: { ...corsHeaders, 'Content-Type': 'application/json' } 
      }
    )
  }
})
```

### Error Handling and Logging

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'

serve(async (req) => {
  try {
    // Attempt to process the request
    const result = await processRequest(req)
    
    return new Response(
      JSON.stringify(result),
      { headers: { 'Content-Type': 'application/json' } }
    )
  } catch (error) {
    // Log the error details (logs are visible in Supabase dashboard)
    console.error('Error processing request:', {
      url: req.url,
      method: req.method,
      error: error.message,
      stack: error.stack
    })
    
    // Return a user-friendly error message
    const statusCode = error.statusCode || 500
    const message = error.userMessage || 'An unexpected error occurred'
    
    return new Response(
      JSON.stringify({ error: message }),
      { 
        status: statusCode, 
        headers: { 'Content-Type': 'application/json' } 
      }
    )
  }
})

async function processRequest(req) {
  // Simulate an error
  if (Math.random() < 0.5) {
    const error = new Error('Something went wrong')
    error.statusCode = 400
    error.userMessage = 'Invalid request parameters'
    throw error
  }
  
  return { success: true, data: 'It worked!' }
}
```

### Handling File Uploads

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Check if request is multipart/form-data
  const contentType = req.headers.get('content-type') || ''
  
  if (!contentType.includes('multipart/form-data')) {
    return new Response(
      JSON.stringify({ error: 'Content-Type must be multipart/form-data' }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    )
  }
  
  try {
    // Parse the form data
    const formData = await req.formData()
    const file = formData.get('file')
    
    if (!file || !(file instanceof File)) {
      return new Response(
        JSON.stringify({ error: 'No file provided' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      )
    }
    
    // Create a Supabase client
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || ''
    const supabaseServiceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || ''
    const supabase = createClient(supabaseUrl, supabaseServiceKey)
    
    // Upload the file to Supabase Storage
    const fileBuffer = await file.arrayBuffer()
    const fileName = `${Date.now()}_${file.name}`
    
    const { data, error } = await supabase
      .storage
      .from('uploads')
      .upload(fileName, fileBuffer, {
        contentType: file.type,
        cacheControl: '3600'
      })
    
    if (error) {
      throw error
    }
    
    // Return the file URL
    const { data: urlData } = supabase
      .storage
      .from('uploads')
      .getPublicUrl(data.path)
    
    return new Response(
      JSON.stringify({ url: urlData.publicUrl }),
      { headers: { 'Content-Type': 'application/json' } }
    )
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

## Best Practices

1. **Keep Functions Small and Focused**: Each function should do one thing well

2. **Use Environment Variables**: Never hardcode secrets

3. **Implement Proper Error Handling**: Always catch errors and return appropriate responses

4. **Add Request Validation**: Validate input before processing

5. **Add Response Types**: Use TypeScript to define response types

6. **Use Service Role Key Carefully**: Only use the service role key when necessary

7. **Set Proper Timeouts**: Consider the function's execution time

8. **Implement Retries for External APIs**: Use retry logic for unreliable services

9. **Log Appropriately**: Include enough information for debugging

10. **Use Dependency Caching**: Import dependencies efficiently

## Limitations and Considerations

- **Execution Time Limit**: 2 seconds (free tier), 60 seconds (paid plan)
- **Memory Limit**: 150MB (free tier), 1GB (paid plan)
- **Cold Starts**: First invocation may be slower
- **Region Restrictions**: Functions run in the same region as your project
- **No Persistent File System**: Files are not persisted between invocations
- **Request Payload Size**: Limited to 6MB
- **Response Payload Size**: Limited to 6MB

## Monitoring and Debugging

You can monitor your Edge Functions from the Supabase dashboard:

1. **Invocation Logs**: View logs for each function invocation
2. **Error Tracking**: See detailed error messages
3. **Performance Metrics**: Monitor execution time and memory usage

Additionally, you can add custom logging:

```typescript
console.log('Debug info:', data)
console.error('Error occurred:', error)
```

These logs will appear in the Supabase dashboard under the Edge Functions section.

By understanding Edge Functions in depth, you can extend your Supabase applications with custom server-side logic and securely integrate with third-party services.
