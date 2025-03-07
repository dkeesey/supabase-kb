# Common Integration Patterns

This document covers common patterns for integrating Supabase with third-party services and APIs. These patterns focus on Edge Functions as the secure bridge between your Supabase application and external services.

## General Integration Pattern

Most third-party integrations follow this general pattern:

1. **Store API credentials** securely as Supabase secrets
2. **Create an Edge Function** to interact with the third-party API
3. **Implement proper authentication** in the Edge Function
4. **Call the Edge Function** from your client application
5. **Store relevant data** in your Supabase database

## AI Integrations

### OpenAI Integration

```typescript
// supabase/functions/openai/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  // Handle CORS preflight requests
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }
  
  try {
    // Get request body
    const { prompt, model = 'gpt-4' } = await req.json();
    
    // Get OpenAI API key from environment variable
    const apiKey = Deno.env.get('OPENAI_API_KEY');
    
    if (!apiKey) {
      throw new Error('OpenAI API key not found');
    }
    
    // Get user information from the request
    const authHeader = req.headers.get('Authorization');
    
    if (!authHeader) {
      throw new Error('Missing Authorization header');
    }
    
    // Create Supabase client
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || '';
    const supabaseKey = Deno.env.get('SUPABASE_ANON_KEY') || '';
    const supabase = createClient(supabaseUrl, supabaseKey, {
      global: {
        headers: {
          Authorization: authHeader
        }
      }
    });
    
    // Check if user is authenticated
    const { data: { user }, error: userError } = await supabase.auth.getUser();
    
    if (userError || !user) {
      throw new Error('Unauthorized');
    }
    
    // Optional: Log the request for billing/tracking
    await supabase
      .from('ai_requests')
      .insert({
        user_id: user.id,
        model: model,
        prompt_tokens: prompt.length // This is approximate; use proper token counting for accuracy
      });
    
    // Call OpenAI API
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        model: model,
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 1000
      })
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`OpenAI API error: ${error.error?.message || response.statusText}`);
    }
    
    const data = await response.json();
    
    // Store the completion for reference
    await supabase
      .from('ai_completions')
      .insert({
        user_id: user.id,
        prompt: prompt,
        completion: data.choices[0].message.content,
        model: model,
        usage: data.usage
      });
    
    // Return the completion
    return new Response(
      JSON.stringify({
        completion: data.choices[0].message.content,
        usage: data.usage
      }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

### Client-Side Usage

```typescript
const generateCompletion = async (prompt: string) => {
  try {
    const { data, error } = await supabase.functions.invoke('openai', {
      body: { prompt }
    });
    
    if (error) throw error;
    
    return data.completion;
  } catch (error) {
    console.error('Error calling OpenAI:', error);
    throw error;
  }
};
```

### Azure OpenAI Integration

```typescript
// supabase/functions/azure-openai/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  // Handle CORS preflight requests
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }
  
  try {
    // Get request body
    const { prompt, model = 'gpt-4' } = await req.json();
    
    // Get Azure OpenAI credentials from environment variables
    const apiKey = Deno.env.get('AZURE_OPENAI_API_KEY');
    const endpoint = Deno.env.get('AZURE_OPENAI_ENDPOINT');
    const deploymentName = Deno.env.get('AZURE_OPENAI_DEPLOYMENT_NAME');
    const apiVersion = '2023-05-15'; // Update as needed
    
    if (!apiKey || !endpoint || !deploymentName) {
      throw new Error('Azure OpenAI credentials not found');
    }
    
    // Get user information from the request
    const authHeader = req.headers.get('Authorization');
    
    if (!authHeader) {
      throw new Error('Missing Authorization header');
    }
    
    // Create Supabase client
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || '';
    const supabaseKey = Deno.env.get('SUPABASE_ANON_KEY') || '';
    const supabase = createClient(supabaseUrl, supabaseKey, {
      global: {
        headers: {
          Authorization: authHeader
        }
      }
    });
    
    // Check if user is authenticated
    const { data: { user }, error: userError } = await supabase.auth.getUser();
    
    if (userError || !user) {
      throw new Error('Unauthorized');
    }
    
    // Call Azure OpenAI API
    const url = `${endpoint}/openai/deployments/${deploymentName}/chat/completions?api-version=${apiVersion}`;
    
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'api-key': apiKey
      },
      body: JSON.stringify({
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 1000
      })
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`Azure OpenAI API error: ${error.error?.message || response.statusText}`);
    }
    
    const data = await response.json();
    
    // Return the completion
    return new Response(
      JSON.stringify({
        completion: data.choices[0].message.content,
        usage: data.usage
      }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

## Payment Integrations

### Stripe Integration

```typescript
// supabase/functions/stripe-create-checkout/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import Stripe from 'https://esm.sh/stripe@12.5.0?target=deno';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  // Handle CORS preflight requests
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }
  
  try {
    // Get request body
    const { priceId, successUrl, cancelUrl } = await req.json();
    
    // Get Stripe API key from environment variable
    const stripeKey = Deno.env.get('STRIPE_SECRET_KEY');
    
    if (!stripeKey) {
      throw new Error('Stripe API key not found');
    }
    
    // Get user information from the request
    const authHeader = req.headers.get('Authorization');
    
    if (!authHeader) {
      throw new Error('Missing Authorization header');
    }
    
    // Create Supabase client
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || '';
    const supabaseKey = Deno.env.get('SUPABASE_ANON_KEY') || '';
    const supabase = createClient(supabaseUrl, supabaseKey, {
      global: {
        headers: {
          Authorization: authHeader
        }
      }
    });
    
    // Check if user is authenticated
    const { data: { user }, error: userError } = await supabase.auth.getUser();
    
    if (userError || !user) {
      throw new Error('Unauthorized');
    }
    
    // Initialize Stripe
    const stripe = new Stripe(stripeKey, {
      apiVersion: '2023-08-16', // Update as needed
      httpClient: Stripe.createFetchHttpClient(),
    });
    
    // Create checkout session
    const session = await stripe.checkout.sessions.create({
      line_items: [
        {
          price: priceId,
          quantity: 1,
        },
      ],
      mode: 'subscription',
      success_url: successUrl,
      cancel_url: cancelUrl,
      client_reference_id: user.id, // Link to Supabase user
    });
    
    // Return the session URL
    return new Response(
      JSON.stringify({ url: session.url }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
    
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

### Stripe Webhook Handler

```typescript
// supabase/functions/stripe-webhook/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import Stripe from 'https://esm.sh/stripe@12.5.0?target=deno';

serve(async (req) => {
  try {
    // Get Stripe webhook secret and API key
    const stripeWebhookSecret = Deno.env.get('STRIPE_WEBHOOK_SECRET');
    const stripeKey = Deno.env.get('STRIPE_SECRET_KEY');
    
    if (!stripeWebhookSecret || !stripeKey) {
      throw new Error('Stripe credentials not found');
    }
    
    // Get the signature from the headers
    const signature = req.headers.get('stripe-signature');
    
    if (!signature) {
      throw new Error('No signature provided');
    }
    
    // Get the raw request body
    const body = await req.text();
    
    // Initialize Stripe
    const stripe = new Stripe(stripeKey, {
      apiVersion: '2023-08-16', // Update as needed
      httpClient: Stripe.createFetchHttpClient(),
    });
    
    // Verify and construct the webhook event
    const event = stripe.webhooks.constructEvent(
      body,
      signature,
      stripeWebhookSecret
    );
    
    // Create Supabase admin client (bypasses RLS)
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || '';
    const supabaseServiceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || '';
    const supabase = createClient(supabaseUrl, supabaseServiceKey);
    
    // Handle different event types
    switch (event.type) {
      case 'checkout.session.completed':
        await handleCheckoutCompleted(event.data.object, supabase);
        break;
        
      case 'customer.subscription.created':
        await handleSubscriptionCreated(event.data.object, supabase);
        break;
        
      case 'customer.subscription.updated':
        await handleSubscriptionUpdated(event.data.object, supabase);
        break;
        
      case 'customer.subscription.deleted':
        await handleSubscriptionCanceled(event.data.object, supabase);
        break;
        
      case 'invoice.payment_succeeded':
        await handleInvoicePaymentSucceeded(event.data.object, supabase);
        break;
        
      case 'invoice.payment_failed':
        await handleInvoicePaymentFailed(event.data.object, supabase);
        break;
        
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
    
    return new Response(
      JSON.stringify({ received: true }),
      { headers: { 'Content-Type': 'application/json' } }
    );
    
  } catch (error) {
    console.error('Error:', error.message);
    return new Response(
      JSON.stringify({ error: `Webhook error: ${error.message}` }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

// Handler functions for different Stripe events
async function handleCheckoutCompleted(session, supabase) {
  // Get user ID from client reference ID
  const userId = session.client_reference_id;
  
  if (!userId) {
    console.error('No user ID found in session');
    return;
  }
  
  // Store Stripe customer ID with user
  await supabase
    .from('customers')
    .upsert({
      user_id: userId,
      stripe_customer_id: session.customer,
      email: session.customer_details.email
    });
}

async function handleSubscriptionCreated(subscription, supabase) {
  // Get customer ID
  const stripeCustomerId = subscription.customer;
  
  // Get user from customer ID
  const { data: customer } = await supabase
    .from('customers')
    .select('user_id')
    .eq('stripe_customer_id', stripeCustomerId)
    .single();
  
  if (!customer) {
    console.error('No customer found for subscription');
    return;
  }
  
  // Add subscription record
  await supabase
    .from('subscriptions')
    .upsert({
      id: subscription.id,
      user_id: customer.user_id,
      status: subscription.status,
      price_id: subscription.items.data[0].price.id,
      quantity: subscription.items.data[0].quantity,
      cancel_at_period_end: subscription.cancel_at_period_end,
      current_period_start: new Date(subscription.current_period_start * 1000).toISOString(),
      current_period_end: new Date(subscription.current_period_end * 1000).toISOString(),
      created_at: new Date(subscription.created * 1000).toISOString(),
      ended_at: subscription.ended_at ? new Date(subscription.ended_at * 1000).toISOString() : null
    });
  
  // Update user with subscription status
  await supabase
    .from('profiles')
    .update({
      subscription_status: subscription.status,
      subscription_tier: getPlanFromPriceId(subscription.items.data[0].price.id)
    })
    .eq('id', customer.user_id);
}

async function handleSubscriptionUpdated(subscription, supabase) {
  // Update subscription record
  await supabase
    .from('subscriptions')
    .update({
      status: subscription.status,
      price_id: subscription.items.data[0].price.id,
      quantity: subscription.items.data[0].quantity,
      cancel_at_period_end: subscription.cancel_at_period_end,
      current_period_start: new Date(subscription.current_period_start * 1000).toISOString(),
      current_period_end: new Date(subscription.current_period_end * 1000).toISOString(),
      ended_at: subscription.ended_at ? new Date(subscription.ended_at * 1000).toISOString() : null
    })
    .eq('id', subscription.id);
  
  // Get user from subscription
  const { data: subscriptionData } = await supabase
    .from('subscriptions')
    .select('user_id')
    .eq('id', subscription.id)
    .single();
  
  if (subscriptionData) {
    // Update user profile
    await supabase
      .from('profiles')
      .update({
        subscription_status: subscription.status,
        subscription_tier: getPlanFromPriceId(subscription.items.data[0].price.id)
      })
      .eq('id', subscriptionData.user_id);
  }
}

async function handleSubscriptionCanceled(subscription, supabase) {
  // Update subscription
  await supabase
    .from('subscriptions')
    .update({
      status: subscription.status,
      cancel_at_period_end: subscription.cancel_at_period_end,
      ended_at: subscription.ended_at ? new Date(subscription.ended_at * 1000).toISOString() : null
    })
    .eq('id', subscription.id);
  
  // Get user from subscription
  const { data: subscriptionData } = await supabase
    .from('subscriptions')
    .select('user_id')
    .eq('id', subscription.id)
    .single();
  
  if (subscriptionData) {
    // Update user profile
    await supabase
      .from('profiles')
      .update({
        subscription_status: subscription.status
      })
      .eq('id', subscriptionData.user_id);
  }
}

async function handleInvoicePaymentSucceeded(invoice, supabase) {
  // Handle successful invoice payment
  // You might want to log this or update usage records
}

async function handleInvoicePaymentFailed(invoice, supabase) {
  // Handle failed invoice payment
  // You might want to alert the user or update status
  
  // Get customer ID
  const stripeCustomerId = invoice.customer;
  
  // Get user from customer ID
  const { data: customer } = await supabase
    .from('customers')
    .select('user_id')
    .eq('stripe_customer_id', stripeCustomerId)
    .single();
  
  if (!customer) {
    console.error('No customer found for invoice');
    return;
  }
  
  // Update user profile to indicate payment failure
  await supabase
    .from('profiles')
    .update({
      payment_status: 'failed'
    })
    .eq('id', customer.user_id);
}

// Helper function to determine plan from price ID
function getPlanFromPriceId(priceId) {
  // Map your price IDs to plan names
  const pricePlans = {
    'price_1234567890': 'basic',
    'price_0987654321': 'premium',
    'price_5432167890': 'enterprise'
  };
  
  return pricePlans[priceId] || 'unknown';
}
```

## Scheduled Tasks

### Cron Job Integration

```typescript
// supabase/functions/scheduled-task/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

// This function will be triggered by a scheduled cron job
serve(async (req) => {
  try {
    // Verify the request is from the Supabase scheduler
    const authorization = req.headers.get('Authorization');
    const supabaseAdminKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY');
    
    if (!authorization || !supabaseAdminKey || authorization !== `Bearer ${supabaseAdminKey}`) {
      throw new Error('Unauthorized');
    }
    
    // Create Supabase admin client (bypasses RLS)
    const supabaseUrl = Deno.env.get('SUPABASE_URL') || '';
    const supabase = createClient(supabaseUrl, supabaseAdminKey);
    
    // Example: Clean up expired sessions
    const { error: deleteError } = await supabase
      .from('sessions')
      .delete()
      .lt('expires_at', new Date().toISOString());
    
    if (deleteError) {
      throw new Error(`Error cleaning up sessions: ${deleteError.message}`);
    }
    
    // Example: Send reminder emails
    const { data: reminders, error: reminderError } = await supabase
      .from('reminders')
      .select('id, user_id, message')
      .eq('status', 'pending')
      .lt('send_at', new Date().toISOString());
    
    if (reminderError) {
      throw new Error(`Error fetching reminders: ${reminderError.message}`);
    }
    
    // Process each reminder
    for (const reminder of reminders) {
      // Mark as processing
      await supabase
        .from('reminders')
        .update({ status: 'processing' })
        .eq('id', reminder.id);
      
      try {
        // Get user email
        const { data: user } = await supabase
          .from('profiles')
          .select('email')
          .eq('id', reminder.user_id)
          .single();
        
        if (user && user.email) {
          // Send email (using another function or service)
          const emailResponse = await fetch(`${supabaseUrl}/functions/v1/send-email`, {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${supabaseAdminKey}`
            },
            body: JSON.stringify({
              to: user.email,
              subject: 'Reminder',
              text: reminder.message
            })
          });
          
          if (!emailResponse.ok) {
            throw new Error(`Failed to send email: ${emailResponse.statusText}`);
          }
          
          // Mark as completed
          await supabase
            .from('reminders')
            .update({ status: 'completed' })
            .eq('id', reminder.id);
        }
      } catch (error) {
        // Mark as failed
        await supabase
          .from('reminders')
          .update({
            status: 'failed',
            error_message: error.message
          })
          .eq('id', reminder.id);
      }
    }
    
    return new Response(
      JSON.stringify({ success: true, processed: reminders.length }),
      { headers: { 'Content-Type': 'application/json' } }
    );
    
  } catch (error) {
    console.error('Error:', error.message);
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});
```

### Setting Up Scheduled Functions in Supabase

To set up scheduled Edge Functions, use the Supabase CLI:

```bash
# Create a new function
supabase functions new scheduled-task

# Deploy the function
supabase functions deploy scheduled-task

# Create a schedule (every day at midnight)
supabase schedules create --cron "0 0 * * *" --function scheduled-task
```

## Best Practices for Edge Function Integrations

1. **Error Handling**
   - Always implement proper error handling
   - Return appropriate HTTP status codes
   - Log errors for debugging

2. **Authentication**
   - Verify user authentication for user-specific operations
   - Use the `service_role` key only when necessary
   - Implement proper permission checks

3. **Rate Limiting**
   - Implement rate limiting for public endpoints
   - Consider API provider limits

4. **Logging**
   - Log important events and errors
   - Include enough context for debugging
   - Don't log sensitive information

5. **Testing**
   - Test functions locally before deployment
   - Create test cases for different scenarios
   - Simulate error conditions

6. **Security**
   - Never expose API keys or secrets
   - Validate and sanitize all user inputs
   - Follow the principle of least privilege

7. **Performance**
   - Keep functions small and focused
   - Use caching where appropriate
   - Minimize database calls

8. **Maintenance**
   - Document your functions
   - Keep dependencies updated
   - Monitor function performance

## Common Pitfalls

1. **Exceeding Function Limits**
   - Functions have execution time limits
   - Use background processing for long-running tasks

2. **Webhook Verification**
   - Always verify signatures for webhook endpoints
   - Protect against replay attacks

3. **Missing CORS Headers**
   - Include proper CORS headers for browser access
   - Handle OPTIONS requests

4. **Excessive Database Queries**
   - Optimize database access patterns
   - Use bulk operations where possible

5. **Ignoring Cold Starts**
   - Edge Functions have cold starts
   - Design for performance with cold starts in mind

6. **Poor Error Handling**
   - Always handle and log errors
   - Provide meaningful error messages

7. **Forgetting Content-Type Headers**
   - Always set the Content-Type header
   - Use the correct content type for the response

By following these integration patterns and best practices, you can create secure, reliable, and efficient connections between your Supabase application and third-party services.
