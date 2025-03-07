# Supabase Authentication

Supabase Authentication provides a complete user management system that works seamlessly with PostgreSQL's Row Level Security. This document covers implementation patterns, best practices, and common pitfalls.

## Authentication Methods

Supabase offers several authentication methods:

1. **Email and Password**
   - Traditional email/password login
   - Optional email verification
   - Password reset functionality

2. **Magic Link**
   - Passwordless authentication via email links
   - Simplifies user experience
   - Reduces password management overhead

3. **Social Providers**
   - Google
   - GitHub
   - Facebook
   - Twitter/X
   - Discord
   - Spotify
   - Apple
   - Microsoft
   - And more...

4. **Phone Authentication**
   - SMS-based verification
   - One-time passwords (OTP)

5. **Single Sign-On (SSO)**
   - Enterprise-grade SAML 2.0 authentication
   - Supports major identity providers

## Implementation Patterns

### Basic Email/Password Authentication

```javascript
// Sign up a new user
const signUp = async (email, password) => {
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
  });
  
  if (error) {
    console.error('Error signing up:', error.message);
    return null;
  }
  
  return data;
};

// Sign in an existing user
const signIn = async (email, password) => {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });
  
  if (error) {
    console.error('Error signing in:', error.message);
    return null;
  }
  
  return data;
};

// Sign out
const signOut = async () => {
  const { error } = await supabase.auth.signOut();
  
  if (error) {
    console.error('Error signing out:', error.message);
  }
};
```

### Magic Link Authentication

```javascript
const sendMagicLink = async (email) => {
  const { data, error } = await supabase.auth.signInWithOtp({
    email,
    options: {
      emailRedirectTo: 'https://example.com/auth/callback',
    },
  });
  
  if (error) {
    console.error('Error sending magic link:', error.message);
    return false;
  }
  
  return true;
};
```

### Social Authentication

```javascript
const signInWithGoogle = async () => {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: 'https://example.com/auth/callback',
    },
  });
  
  if (error) {
    console.error('Error signing in with Google:', error.message);
    return null;
  }
  
  return data;
};
```

### Phone Authentication

```javascript
// Send OTP to phone
const sendOTP = async (phone) => {
  const { data, error } = await supabase.auth.signInWithOtp({
    phone,
  });
  
  if (error) {
    console.error('Error sending OTP:', error.message);
    return false;
  }
  
  return true;
};

// Verify OTP
const verifyOTP = async (phone, token) => {
  const { data, error } = await supabase.auth.verifyOtp({
    phone,
    token,
    type: 'sms',
  });
  
  if (error) {
    console.error('Error verifying OTP:', error.message);
    return null;
  }
  
  return data;
};
```

## User Management

### Get Current User

```javascript
const getCurrentUser = async () => {
  const { data: { user } } = await supabase.auth.getUser();
  return user;
};
```

### Update User

```javascript
const updateUserEmail = async (newEmail) => {
  const { data, error } = await supabase.auth.updateUser({
    email: newEmail,
  });
  
  if (error) {
    console.error('Error updating email:', error.message);
    return false;
  }
  
  return true;
};

const updateUserPassword = async (newPassword) => {
  const { data, error } = await supabase.auth.updateUser({
    password: newPassword,
  });
  
  if (error) {
    console.error('Error updating password:', error.message);
    return false;
  }
  
  return true;
};

const updateUserMetadata = async (metadata) => {
  const { data, error } = await supabase.auth.updateUser({
    data: metadata,
  });
  
  if (error) {
    console.error('Error updating user metadata:', error.message);
    return false;
  }
  
  return true;
};
```

### Reset Password

```javascript
const resetPassword = async (email) => {
  const { data, error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: 'https://example.com/reset-password',
  });
  
  if (error) {
    console.error('Error resetting password:', error.message);
    return false;
  }
  
  return true;
};
```

## Session Management

### Get Session

```javascript
const getSession = async () => {
  const { data: { session }, error } = await supabase.auth.getSession();
  
  if (error) {
    console.error('Error getting session:', error.message);
    return null;
  }
  
  return session;
};
```

### Refresh Session

```javascript
const refreshSession = async () => {
  const { data, error } = await supabase.auth.refreshSession();
  
  if (error) {
    console.error('Error refreshing session:', error.message);
    return null;
  }
  
  return data;
};
```

### Listen to Auth Changes

```javascript
const setupAuthListener = () => {
  return supabase.auth.onAuthStateChange((event, session) => {
    console.log('Auth event:', event);
    console.log('Session:', session);
    
    // Handle different auth events
    switch (event) {
      case 'SIGNED_IN':
        // Handle sign in
        break;
      case 'SIGNED_OUT':
        // Handle sign out
        break;
      case 'USER_UPDATED':
        // Handle user update
        break;
      case 'PASSWORD_RECOVERY':
        // Handle password recovery
        break;
      default:
        // Handle other events
        break;
    }
  });
};

// To unsubscribe from the listener
const unsubscribe = setupAuthListener();
// Later when no longer needed
unsubscribe.subscription.unsubscribe();
```

## Custom User Properties with User Profiles

A common pattern is to create a `profiles` table to store additional user information:

```sql
-- Create a table for public profiles
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  username TEXT UNIQUE,
  full_name TEXT,
  avatar_url TEXT,
  website TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc', NOW()),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc', NOW())
);

-- Enable Row Level Security
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Create a policy that allows users to view all profiles
CREATE POLICY "Profiles are viewable by everyone" 
  ON profiles FOR SELECT 
  USING (true);

-- Create a policy that allows users to update only their own profile
CREATE POLICY "Users can update their own profile" 
  ON profiles FOR UPDATE 
  USING (auth.uid() = id);

-- Create a profile when a new user signs up
CREATE OR REPLACE FUNCTION public.handle_new_user() 
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, username, full_name, avatar_url)
  VALUES (
    NEW.id, 
    NEW.raw_user_meta_data->>'username', 
    NEW.raw_user_meta_data->>'full_name', 
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger the function every time a user is created
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();
```

## Role-Based Access Control (RBAC)

You can implement role-based access control by:

1. **Creating a roles table and assigning roles to users:**

```sql
CREATE TABLE roles (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE
);

CREATE TABLE user_roles (
  user_id UUID REFERENCES auth.users(id) NOT NULL,
  role_id INTEGER REFERENCES roles(id) NOT NULL,
  PRIMARY KEY (user_id, role_id)
);

-- Insert basic roles
INSERT INTO roles (name) VALUES ('admin'), ('moderator'), ('user');
```

2. **Creating a function to check user roles:**

```sql
CREATE OR REPLACE FUNCTION public.user_has_role(role_name TEXT)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM user_roles ur
    JOIN roles r ON ur.role_id = r.id
    WHERE ur.user_id = auth.uid() AND r.name = role_name
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

3. **Using the function in RLS policies:**

```sql
CREATE POLICY "Admins can update any content"
  ON content FOR UPDATE
  USING (user_has_role('admin'));
```

## Client-Side Implementation Best Practices

### Protecting Routes in Frontend Frameworks

#### React with React Router

```jsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from './your-auth-hook';

const ProtectedRoute = () => {
  const { user, loading } = useAuth();
  
  if (loading) {
    return <div>Loading...</div>;
  }
  
  if (!user) {
    return <Navigate to="/login" replace />;
  }
  
  return <Outlet />;
};

// Usage in router
// <Route element={<ProtectedRoute />}>
//   <Route path="/dashboard" element={<Dashboard />} />
// </Route>
```

#### React Custom Auth Hook

```jsx
import { useState, useEffect, createContext, useContext } from 'react';
import { supabase } from './supabaseClient';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Get initial session
    const getInitialSession = async () => {
      const { data: { session } } = await supabase.auth.getSession();
      setUser(session?.user ?? null);
      setLoading(false);
    };
    
    getInitialSession();
    
    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (_event, session) => {
        setUser(session?.user ?? null);
        setLoading(false);
      }
    );
    
    return () => subscription.unsubscribe();
  }, []);
  
  const value = {
    user,
    loading,
    signIn: (email, password) => supabase.auth.signInWithPassword({ email, password }),
    signUp: (email, password) => supabase.auth.signUp({ email, password }),
    signOut: () => supabase.auth.signOut(),
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  return useContext(AuthContext);
};
```

## Server-Side Authentication

### Verifying JWT Tokens in Edge Functions

```typescript
// Inside an Edge Function
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

Deno.serve(async (req) => {
  // Create a Supabase client with the Auth context of the logged in user
  const supabaseClient = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_ANON_KEY') ?? '',
    {
      global: {
        headers: { Authorization: req.headers.get('Authorization')! },
      },
    }
  );
  
  // Get the session from the request
  const { data: { session }, error } = await supabaseClient.auth.getSession();
  
  if (error || !session) {
    return new Response(
      JSON.stringify({ error: 'Unauthorized' }),
      { status: 401, headers: { 'Content-Type': 'application/json' } }
    );
  }

  // You now have access to the user's identity
  const userId = session.user.id;
  
  // Continue with your function...
});
```

## Common Pitfalls and Solutions

### 1. Email Verification Requirements

**Pitfall**: Users not receiving verification emails or being unable to log in.

**Solution**: 
- Check your email verification settings in the Supabase dashboard
- Temporarily disable email confirmation for testing
- Check spam folders
- Use a proper email delivery service for production

### 2. CORS Issues with OAuth

**Pitfall**: Social authentication fails due to CORS errors.

**Solution**: 
- Ensure your redirect URLs are correctly set in the Supabase dashboard
- Add your site domain to the allowed domains list
- Check for protocol mismatches (http vs https)

### 3. Token Expiration

**Pitfall**: User sessions unexpectedly ending.

**Solution**:
- Implement token refresh logic
- Adjust token expiration settings in the Supabase dashboard
- Handle session refreshes gracefully in the UI

### 4. Missing User Data After Sign-Up

**Pitfall**: User profile data not being created properly.

**Solution**:
- Ensure your database triggers are properly set up
- Add error handling to your user creation functions
- Use transactions to ensure atomicity

### 5. Securing the Service Role Key

**Pitfall**: Exposing the `service_role` key in client-side code.

**Solution**:
- Never use the `service_role` key in client-side code
- Use Edge Functions for operations requiring elevated privileges
- Implement proper Row Level Security policies

## Security Best Practices

1. **Use RLS Policies**: Always use Row Level Security policies to protect your data

2. **Minimal Permissions**: Grant the minimum permissions necessary for each operation

3. **Validate Input**: Always validate user input on both client and server side

4. **Secure Password Policies**: Enforce strong password requirements

5. **Rate Limiting**: Implement rate limiting for authentication attempts

6. **Audit Logs**: Monitor authentication events and set up alerts for suspicious activity

7. **Refresh Tokens**: Implement proper refresh token rotation

8. **HTTPS Only**: Only use HTTPS for all authentication operations

9. **Secure Cookie Settings**: Use secure and httpOnly flags for cookies

10. **Regular Security Audits**: Periodically review your authentication configuration

## Advanced Topics

### Custom JWT Claims

You can add custom claims to JWT tokens using PostgreSQL functions:

```sql
-- Create a function to add custom claims to the JWT
CREATE OR REPLACE FUNCTION auth.jwt() RETURNS jsonb
  LANGUAGE sql STABLE
  SECURITY DEFINER
  SET search_path = auth
  AS $$
  DECLARE
    role text;
    ready boolean;
  BEGIN
    -- Check if this user has a role
    role := (
      SELECT r.name
      FROM public.user_roles ur
      JOIN public.roles r ON ur.role_id = r.id
      WHERE ur.user_id = auth.uid()
      LIMIT 1
    );
    
    -- Add the role as a custom claim
    RETURN jsonb_build_object(
      'role', COALESCE(role, 'user'),
      'aud', current_setting('request.jwt.aud', true),
      'sub', auth.uid()::text
    );
  END;
$$;
```

### Multi-Factor Authentication (MFA)

```javascript
// Enable TOTP (Time-based One-Time Password) for a user
const enableMFA = async () => {
  const { data, error } = await supabase.auth.mfa.enroll();
  
  if (error) {
    console.error('Error enabling MFA:', error.message);
    return null;
  }
  
  // Return the TOTP URI for QR code generation
  return data.totp.uri;
};

// Verify the TOTP
const verifyMFA = async (code) => {
  const { data, error } = await supabase.auth.mfa.challenge({ 
    factorId: 'totp',
    code 
  });
  
  if (error) {
    console.error('Error verifying MFA:', error.message);
    return false;
  }
  
  return true;
};
```

By understanding these authentication patterns and best practices, you can implement secure, scalable authentication in your Supabase applications.
