# Supabase Database

Supabase provides a full PostgreSQL database with additional extensions and tools to simplify development. This document covers database design, querying, optimization, and security.

## Core Database Concepts

### PostgreSQL Overview

Supabase uses PostgreSQL, a powerful open-source relational database system, offering:

- Robust ACID compliance
- Advanced query capabilities
- JSON/JSONB support
- Extensive extension ecosystem
- Strong data integrity

### Database Structure

A Supabase database contains:

- **Tables**: Store structured data with columns and rows
- **Views**: Virtual tables defined by queries
- **Functions**: Custom SQL or JavaScript routines
- **Triggers**: Automatic responses to database events
- **Types**: Custom data types
- **Extensions**: Add functionality to PostgreSQL

### PostgreSQL Extensions

Supabase pre-installs several extensions:

- **postgis**: Geographic information system
- **pg_graphql**: GraphQL API for your database
- **pgjwt**: JWT functions for PostgreSQL
- **pgvector**: Vector similarity search for AI
- **pgsodium**: Encryption and hashing utilities
- **plv8**: JavaScript language for stored procedures

## Database Design Best Practices

### Schema Design

1. **Use a public schema for app data**
   - Default schema for application tables

2. **Create logical separation with additional schemas**
   - Organize related tables into separate schemas
   - Example: `public` for app data, `auth` for authentication, `storage` for file metadata

3. **Define relationships explicitly**
   - Use foreign keys to enforce referential integrity

4. **Consider normalization levels**
   - Balance between normalization and query performance
   - Denormalize when performance requires it

### Table Design

1. **Primary Keys**
   - Use UUID for compatibility with Supabase's auth system
   ```sql
   CREATE TABLE items (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     name TEXT NOT NULL,
     created_at TIMESTAMPTZ DEFAULT NOW()
   );
   ```

2. **Foreign Keys**
   - Define relationships with foreign keys
   ```sql
   CREATE TABLE orders (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     user_id UUID REFERENCES auth.users(id) NOT NULL,
     status TEXT NOT NULL,
     created_at TIMESTAMPTZ DEFAULT NOW()
   );
   ```

3. **Timestamps**
   - Add `created_at` and `updated_at` columns
   ```sql
   CREATE TABLE products (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     name TEXT NOT NULL,
     price DECIMAL(10,2) NOT NULL,
     created_at TIMESTAMPTZ DEFAULT NOW(),
     updated_at TIMESTAMPTZ DEFAULT NOW()
   );
   
   -- Add trigger for updated_at
   CREATE TRIGGER set_updated_at
   BEFORE UPDATE ON products
   FOR EACH ROW
   EXECUTE PROCEDURE set_updated_at_timestamp();
   ```

4. **Use appropriate data types**
   - TEXT for variable-length strings
   - INTEGER or BIGINT for numbers
   - BOOLEAN for true/false values
   - TIMESTAMPTZ for timestamps (with timezone)
   - JSONB for JSON data
   - UUID for unique identifiers

### Example Schema Design

```sql
-- Users profile information
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  username TEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  website TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Products table
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  inventory INTEGER NOT NULL DEFAULT 0,
  is_available BOOLEAN DEFAULT true,
  category_id UUID REFERENCES categories(id),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Categories table
CREATE TABLE categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT UNIQUE NOT NULL,
  parent_id UUID REFERENCES categories(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Orders table
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) NOT NULL,
  status TEXT NOT NULL,
  total_amount DECIMAL(10,2) NOT NULL,
  shipping_address JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Order items join table
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id) NOT NULL,
  product_id UUID REFERENCES products(id) NOT NULL,
  quantity INTEGER NOT NULL,
  price_at_purchase DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Working with Supabase Database

### Using the Dashboard

The Supabase dashboard provides these database tools:

1. **Table Editor**: Create and edit tables with a visual interface
2. **SQL Editor**: Write and execute SQL queries
3. **Database Schema**: View and manage your database schema
4. **Extensions**: Enable and configure PostgreSQL extensions
5. **Backups**: Manage database backups
6. **Replication**: Set up database replication

### SQL Queries

#### Basic CRUD Operations

```sql
-- Create (Insert)
INSERT INTO products (name, description, price, category_id)
VALUES ('Product Name', 'Product Description', 19.99, '123e4567-e89b-12d3-a456-426614174000')
RETURNING *;

-- Read (Select)
SELECT * FROM products WHERE price < 50.00 ORDER BY price ASC;

-- Update
UPDATE products 
SET price = 24.99, updated_at = NOW() 
WHERE id = '123e4567-e89b-12d3-a456-426614174000'
RETURNING *;

-- Delete
DELETE FROM products 
WHERE id = '123e4567-e89b-12d3-a456-426614174000'
RETURNING *;
```

#### Advanced Queries

```sql
-- Joins
SELECT 
  p.id, 
  p.name, 
  p.price, 
  c.name AS category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_available = true;

-- Aggregations
SELECT 
  category_id, 
  COUNT(*) AS product_count, 
  AVG(price) AS avg_price,
  MIN(price) AS min_price,
  MAX(price) AS max_price
FROM products
GROUP BY category_id;

-- Common Table Expressions (CTEs)
WITH revenue_by_month AS (
  SELECT 
    DATE_TRUNC('month', created_at) AS month,
    SUM(total_amount) AS revenue
  FROM orders
  WHERE status = 'completed'
  GROUP BY month
)
SELECT 
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
  (revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (ORDER BY month) * 100 AS growth_percent
FROM revenue_by_month
ORDER BY month;

-- JSON/JSONB Queries
SELECT 
  id, 
  shipping_address->>'street' AS street,
  shipping_address->>'city' AS city,
  shipping_address->>'country' AS country
FROM orders;

-- Array Operations
SELECT 
  product_id,
  ARRAY_AGG(order_id) AS order_ids
FROM order_items
GROUP BY product_id;
```

### Using PostgreSQL Functions

```sql
-- Create a function to calculate order total
CREATE OR REPLACE FUNCTION calculate_order_total(order_id UUID)
RETURNS DECIMAL AS $$
DECLARE
  total DECIMAL(10,2);
BEGIN
  SELECT SUM(price_at_purchase * quantity)
  INTO total
  FROM order_items
  WHERE order_id = calculate_order_total.order_id;
  
  RETURN total;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT calculate_order_total('123e4567-e89b-12d3-a456-426614174000');
```

### Using Database Triggers

```sql
-- Create a function for the trigger
CREATE OR REPLACE FUNCTION set_updated_at_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create a trigger on a table
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE PROCEDURE set_updated_at_timestamp();

-- Create a trigger for inventory tracking
CREATE OR REPLACE FUNCTION update_inventory_after_order()
RETURNS TRIGGER AS $$
BEGIN
  -- Decrease inventory when a new order item is created
  UPDATE products
  SET inventory = inventory - NEW.quantity
  WHERE id = NEW.product_id;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER decrease_inventory
AFTER INSERT ON order_items
FOR EACH ROW
EXECUTE PROCEDURE update_inventory_after_order();
```

## Client-Side Database Access

### JavaScript/TypeScript Client

```typescript
// Fetch all products in a category
const fetchProductsByCategory = async (categoryId) => {
  const { data, error } = await supabase
    .from('products')
    .select(`
      id,
      name,
      description,
      price,
      inventory,
      is_available,
      categories (
        id,
        name
      )
    `)
    .eq('category_id', categoryId)
    .order('name');
    
  if (error) {
    console.error('Error fetching products:', error);
    return null;
  }
  
  return data;
};

// Insert a new product
const createProduct = async (product) => {
  const { data, error } = await supabase
    .from('products')
    .insert([product])
    .select();
    
  if (error) {
    console.error('Error creating product:', error);
    return null;
  }
  
  return data[0];
};

// Update a product
const updateProduct = async (id, updates) => {
  const { data, error } = await supabase
    .from('products')
    .update(updates)
    .eq('id', id)
    .select();
    
  if (error) {
    console.error('Error updating product:', error);
    return null;
  }
  
  return data[0];
};

// Delete a product
const deleteProduct = async (id) => {
  const { error } = await supabase
    .from('products')
    .delete()
    .eq('id', id);
    
  if (error) {
    console.error('Error deleting product:', error);
    return false;
  }
  
  return true;
};
```

### Using Filters and Modifiers

```typescript
// Using filters
const getActiveProducts = async () => {
  const { data, error } = await supabase
    .from('products')
    .select('*')
    .eq('is_available', true)
    .gt('inventory', 0)
    .lt('price', 100);
    
  return data;
};

// Using OR conditions
const searchProducts = async (query) => {
  const { data, error } = await supabase
    .from('products')
    .select('*')
    .or(`name.ilike.%${query}%,description.ilike.%${query}%`);
    
  return data;
};

// Using IN condition
const getProductsByIds = async (ids) => {
  const { data, error } = await supabase
    .from('products')
    .select('*')
    .in('id', ids);
    
  return data;
};

// Pagination
const getProductsWithPagination = async (page = 1, pageSize = 10) => {
  const { data, count, error } = await supabase
    .from('products')
    .select('*', { count: 'exact' })
    .range((page - 1) * pageSize, page * pageSize - 1);
    
  return {
    data,
    count,
    totalPages: Math.ceil(count / pageSize),
    currentPage: page
  };
};
```

### Executing Raw SQL Queries

When you need more control:

```typescript
const executeComplexQuery = async () => {
  const { data, error } = await supabase.rpc('execute_sql', {
    query_text: `
      WITH monthly_sales AS (
        SELECT 
          DATE_TRUNC('month', o.created_at) AS month,
          p.category_id,
          SUM(oi.quantity * oi.price_at_purchase) AS total_revenue
        FROM orders o
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE o.status = 'completed'
        GROUP BY month, p.category_id
      )
      SELECT 
        c.name AS category_name,
        TO_CHAR(ms.month, 'Month YYYY') AS month,
        ms.total_revenue
      FROM monthly_sales ms
      JOIN categories c ON ms.category_id = c.id
      ORDER BY ms.month, ms.total_revenue DESC
    `
  });
  
  return data;
};
```

## Database Security

### Row Level Security (RLS)

Row Level Security (RLS) allows you to restrict access to rows in a table based on the user making the request.

#### Enabling RLS

```sql
-- Enable RLS on a table
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
```

#### Creating Policies

```sql
-- Allow users to view all products
CREATE POLICY "Products are viewable by everyone" 
ON products FOR SELECT 
USING (true);

-- Allow users to insert their own orders
CREATE POLICY "Users can create their own orders" 
ON orders FOR INSERT 
WITH CHECK (auth.uid() = user_id);

-- Allow users to view their own orders
CREATE POLICY "Users can view their own orders" 
ON orders FOR SELECT 
USING (auth.uid() = user_id);

-- Allow users to update their own orders with specific statuses
CREATE POLICY "Users can update their pending orders" 
ON orders FOR UPDATE 
USING (auth.uid() = user_id AND status = 'pending')
WITH CHECK (auth.uid() = user_id AND status IN ('pending', 'canceled'));

-- Allow users to delete their own orders
CREATE POLICY "Users can delete their own orders" 
ON orders FOR DELETE 
USING (auth.uid() = user_id AND status = 'pending');
```

#### Admin Access Policy

```sql
-- Allow admin users to perform all operations
CREATE POLICY "Admin users have full access"
ON products
USING (
  EXISTS (
    SELECT 1 FROM user_roles ur
    JOIN roles r ON ur.role_id = r.id
    WHERE ur.user_id = auth.uid() AND r.name = 'admin'
  )
);
```

### Using Stored Procedures for Security

For operations that require elevated privileges:

```sql
-- Create a stored procedure for canceling orders
CREATE OR REPLACE FUNCTION cancel_order(order_id UUID)
RETURNS BOOLEAN AS $$
DECLARE
  order_user_id UUID;
  order_status TEXT;
BEGIN
  -- Get the order's user_id and status
  SELECT user_id, status INTO order_user_id, order_status
  FROM orders
  WHERE id = order_id;
  
  -- Check if the order belongs to the current user and is in a cancellable state
  IF order_user_id = auth.uid() AND order_status IN ('pending', 'processing') THEN
    -- Update the order
    UPDATE orders
    SET status = 'canceled', updated_at = NOW()
    WHERE id = order_id;
    
    -- Return success
    RETURN TRUE;
  ELSE
    -- Return failure
    RETURN FALSE;
  END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Usage from client
const cancelOrder = async (orderId) => {
  const { data, error } = await supabase
    .rpc('cancel_order', { order_id: orderId });
    
  return data; // true or false
};
```

## Database Migrations

### Creating Migrations

Supabase CLI allows you to create and manage database migrations:

```bash
# Create a new migration
supabase migration new create_products_table

# This creates a file in supabase/migrations/TIMESTAMP_create_products_table.sql
```

### Example Migration File

```sql
-- Migration: create_products_table
-- Created at: 2023-05-15 14:30:00
-- supabase/migrations/20230515143000_create_products_table.sql

-- Create products table
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  inventory INTEGER NOT NULL DEFAULT 0,
  is_available BOOLEAN DEFAULT true,
  category_id UUID REFERENCES categories(id),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Create policies
CREATE POLICY "Products are viewable by everyone" 
ON products FOR SELECT 
USING (true);

CREATE POLICY "Admin users can manage products"
ON products
USING (
  EXISTS (
    SELECT 1 FROM user_roles ur
    JOIN roles r ON ur.role_id = r.id
    WHERE ur.user_id = auth.uid() AND r.name = 'admin'
  )
);

-- Create updated_at trigger
CREATE TRIGGER set_products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE PROCEDURE set_updated_at_timestamp();
```

### Running Migrations

```bash
# Apply migrations locally
supabase db reset

# Push migrations to remote project
supabase db push
```

## Performance Optimization

### Indexing

```sql
-- Create an index on a column
CREATE INDEX idx_products_category_id ON products(category_id);

-- Create a composite index
CREATE INDEX idx_order_items_order_product ON order_items(order_id, product_id);

-- Create a unique index
CREATE UNIQUE INDEX idx_profiles_username ON profiles(username);

-- Create a partial index
CREATE INDEX idx_products_available ON products(name) WHERE is_available = true;

-- Create a text search index
CREATE INDEX idx_products_text_search ON products USING GIN (to_tsvector('english', name || ' ' || coalesce(description, '')));
```

### Query Optimization

1. **Use EXPLAIN ANALYZE to understand query execution**

```sql
EXPLAIN ANALYZE
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.price < 50.00
ORDER BY p.price;
```

2. **Optimize joins**
   - Join on indexed columns
   - Use appropriate join types (INNER, LEFT, RIGHT)
   - Filter early to reduce the dataset

3. **Use query planning**
   - CTE (Common Table Expressions) for complex queries
   - Subqueries when appropriate
   - Materialized views for expensive calculations

4. **Pagination**
   - Always use LIMIT and OFFSET for large datasets
   - Consider keyset pagination for better performance

```sql
-- Traditional pagination (slower for large offsets)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 100;

-- Keyset pagination (faster)
SELECT * FROM products
WHERE id > '123e4567-e89b-12d3-a456-426614174000'
ORDER BY id LIMIT 10;
```

### Database Maintenance

1. **Regular VACUUM**
   - Supabase automatically handles VACUUM operations
   - Consider manual VACUUM ANALYZE for performance tuning

2. **Monitoring Table Bloat**
   - Large, heavily-updated tables may need attention

3. **Removing Unused Indexes**
   - Indexes improve read performance but slow down writes
   - Remove indexes that aren't being used

## Common Patterns

### Implementing Soft Delete

```sql
-- Add columns for soft delete
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;

-- Modify policies to exclude deleted items
CREATE POLICY "Exclude deleted products" 
ON products FOR SELECT 
USING (deleted_at IS NULL);

-- Create function for soft delete
CREATE OR REPLACE FUNCTION soft_delete_product(product_id UUID)
RETURNS BOOLEAN AS $$
BEGIN
  UPDATE products
  SET deleted_at = NOW()
  WHERE id = product_id;
  
  RETURN FOUND;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Implementing Full-Text Search

```sql
-- Create a function for text search
CREATE OR REPLACE FUNCTION search_products(search_query TEXT)
RETURNS SETOF products AS $$
BEGIN
  RETURN QUERY
  SELECT p.*
  FROM products p
  WHERE to_tsvector('english', p.name || ' ' || coalesce(p.description, '')) @@ to_tsquery('english', search_query)
  ORDER BY ts_rank(to_tsvector('english', p.name), to_tsquery('english', search_query)) DESC;
END;
$$ LANGUAGE plpgsql;

-- Example usage
SELECT * FROM search_products('organic vegetables');
```

### Implementing Hierarchical Data (Tree Structures)

```sql
-- For categories with parent-child relationships
WITH RECURSIVE category_tree AS (
  -- Base case: select root categories
  SELECT id, name, parent_id, 0 AS level, ARRAY[name] AS path
  FROM categories
  WHERE parent_id IS NULL
  
  UNION ALL
  
  -- Recursive case: select children categories
  SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || c.name
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT id, name, parent_id, level, path
FROM category_tree
ORDER BY path;
```

### Implementing Versioning/History

```sql
-- Create a history table
CREATE TABLE products_history (
  id UUID,
  product_id UUID,
  name TEXT,
  description TEXT,
  price DECIMAL(10,2),
  inventory INTEGER,
  is_available BOOLEAN,
  category_id UUID,
  metadata JSONB,
  changed_at TIMESTAMPTZ DEFAULT NOW(),
  changed_by UUID,
  operation CHAR(1)  -- 'I' = insert, 'U' = update, 'D' = delete
);

-- Create trigger function
CREATE OR REPLACE FUNCTION log_product_changes()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO products_history (
      id, product_id, name, description, price, inventory, 
      is_available, category_id, metadata, changed_by, operation
    )
    VALUES (
      gen_random_uuid(), NEW.id, NEW.name, NEW.description, NEW.price, 
      NEW.inventory, NEW.is_available, NEW.category_id, NEW.metadata, 
      auth.uid(), 'I'
    );
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO products_history (
      id, product_id, name, description, price, inventory, 
      is_available, category_id, metadata, changed_by, operation
    )
    VALUES (
      gen_random_uuid(), NEW.id, NEW.name, NEW.description, NEW.price, 
      NEW.inventory, NEW.is_available, NEW.category_id, NEW.metadata, 
      auth.uid(), 'U'
    );
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO products_history (
      id, product_id, name, description, price, inventory, 
      is_available, category_id, metadata, changed_by, operation
    )
    VALUES (
      gen_random_uuid(), OLD.id, OLD.name, OLD.description, OLD.price, 
      OLD.inventory, OLD.is_available, OLD.category_id, OLD.metadata, 
      auth.uid(), 'D'
    );
    RETURN OLD;
  END IF;
END;
$$ LANGUAGE plpgsql;

-- Create triggers
CREATE TRIGGER log_product_insert
AFTER INSERT ON products
FOR EACH ROW
EXECUTE PROCEDURE log_product_changes();

CREATE TRIGGER log_product_update
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE PROCEDURE log_product_changes();

CREATE TRIGGER log_product_delete
AFTER DELETE ON products
FOR EACH ROW
EXECUTE PROCEDURE log_product_changes();
```

## Common Pitfalls and Solutions

1. **Not using Row Level Security**
   - Always implement RLS for tables with sensitive data
   - Check RLS policies thoroughly

2. **Exposing secret data**
   - Use views or functions to filter sensitive data
   - Avoid storing secrets in regular columns

3. **N+1 query problems**
   - Use `.select('*')` instead of multiple separate queries
   - Utilize JOINs when possible

4. **Large transaction volumes**
   - Use connection pooling
   - Consider breaking large operations into smaller batches

5. **Complex query performance**
   - Create materialized views for complex reports
   - Optimize with proper indexing
   - Consider denormalization for read-heavy operations

By understanding these database concepts and best practices, you can create efficient, secure, and performant applications with Supabase.
