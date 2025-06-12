# Database Table Design Strategy Guide

A comprehensive framework for designing robust database schemas by systematically identifying and organizing your application's data requirements.

## Overview

Effective database design requires a methodical approach to identifying what data your application manages and how different pieces of information relate to each other. This guide provides a systematic process for translating application requirements into a well-structured database schema.

## The Complete Design Process

### 1. Foundation Tables - Common Application Features

**Purpose:** Establish the core infrastructure that most applications require, using industry-standard naming conventions and column structures.

#### User Authentication & Management
```sql
-- Users table (primary authentication)
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email_verified_at TIMESTAMP NULL,
    remember_token VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL -- soft deletes
);

-- Password resets
CREATE TABLE password_resets (
    email VARCHAR(255) NOT NULL,
    token VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_token (token)
);

-- Email verification tokens
CREATE TABLE email_verifications (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    token VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

#### User Profiles & Settings
```sql
-- Extended user profile information
CREATE TABLE user_profiles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNIQUE NOT NULL,
    bio TEXT,
    avatar_url VARCHAR(500),
    phone VARCHAR(20),
    date_of_birth DATE,
    timezone VARCHAR(50) DEFAULT 'UTC',
    language VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- User preferences and settings
CREATE TABLE user_settings (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    setting_key VARCHAR(100) NOT NULL,
    setting_value JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_setting (user_id, setting_key)
);
```

#### Comments System
```sql
-- Polymorphic comments (can attach to any resource)
CREATE TABLE comments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    commentable_type VARCHAR(50) NOT NULL, -- 'posts', 'articles', etc.
    commentable_id BIGINT NOT NULL,
    parent_id BIGINT NULL, -- for nested comments
    content TEXT NOT NULL,
    is_approved BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id) REFERENCES comments(id) ON DELETE CASCADE,
    INDEX idx_commentable (commentable_type, commentable_id),
    INDEX idx_parent (parent_id)
);
```

#### Media & File Management
```sql
-- File uploads and media storage
CREATE TABLE media (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    disk VARCHAR(50) DEFAULT 'local', -- storage disk
    alt_text VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- Polymorphic relationship for attaching media to any resource
CREATE TABLE mediables (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    media_id BIGINT NOT NULL,
    mediable_type VARCHAR(50) NOT NULL,
    mediable_id BIGINT NOT NULL,
    purpose VARCHAR(50) DEFAULT 'attachment', -- 'avatar', 'gallery', 'attachment'
    sort_order INT DEFAULT 0,
    FOREIGN KEY (media_id) REFERENCES media(id) ON DELETE CASCADE,
    INDEX idx_mediable (mediable_type, mediable_id)
);
```

#### Notifications System
```sql
-- User notifications
CREATE TABLE notifications (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    type VARCHAR(100) NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT,
    data JSON, -- additional notification data
    read_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_unread (user_id, read_at)
);
```

#### Activity Logging & Audits
```sql
-- System activity logs
CREATE TABLE activity_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    action VARCHAR(100) NOT NULL,
    subject_type VARCHAR(50),
    subject_id BIGINT,
    description TEXT,
    ip_address VARCHAR(45),
    user_agent TEXT,
    properties JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_user (user_id),
    INDEX idx_subject (subject_type, subject_id)
);
```

### 2. Resource-Specific Tables - Core Business Entities

**Purpose:** Create dedicated tables for each distinct type of resource or entity that your application manages.

#### Identification Strategy
Ask yourself these questions to identify resources:
- What are the main "things" users interact with in your app?
- What entities do users create, read, update, or delete?
- What would you put after "I want to see all my..." or "Show me the..."?

#### Example Resource Tables

**Blog/Content Application:**
```sql
-- Posts/Articles
CREATE TABLE posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    excerpt TEXT,
    content LONGTEXT NOT NULL,
    status ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    featured_image_id BIGINT,
    published_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (featured_image_id) REFERENCES media(id) ON DELETE SET NULL,
    INDEX idx_status_published (status, published_at),
    INDEX idx_slug (slug)
);

-- Categories for organizing content
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    parent_id BIGINT,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL
);

-- Tags for flexible content labeling
CREATE TABLE tags (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**E-commerce Application:**
```sql
-- Products
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    short_description VARCHAR(500),
    sku VARCHAR(100) UNIQUE,
    price DECIMAL(10,2) NOT NULL,
    sale_price DECIMAL(10,2),
    stock_quantity INT DEFAULT 0,
    manage_stock BOOLEAN DEFAULT TRUE,
    status ENUM('active', 'inactive', 'out_of_stock') DEFAULT 'active',
    weight DECIMAL(8,2),
    dimensions VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_sku (sku)
);

-- Orders
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    shipping_amount DECIMAL(10,2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    billing_address JSON,
    shipping_address JSON,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_user (user_id),
    INDEX idx_status (status)
);
```

**Project Management Application:**
```sql
-- Projects
CREATE TABLE projects (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id BIGINT NOT NULL,
    status ENUM('planning', 'active', 'on_hold', 'completed', 'cancelled') DEFAULT 'planning',
    priority ENUM('low', 'medium', 'high', 'urgent') DEFAULT 'medium',
    start_date DATE,
    due_date DATE,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Tasks
CREATE TABLE tasks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    project_id BIGINT NOT NULL,
    assigned_to BIGINT,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status ENUM('todo', 'in_progress', 'review', 'done') DEFAULT 'todo',
    priority ENUM('low', 'medium', 'high', 'urgent') DEFAULT 'medium',
    due_date DATETIME,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_to) REFERENCES users(id) ON DELETE SET NULL
);
```

### 3. Relationship Tables - Connections Between Resources

**Purpose:** Model complex relationships, ownership patterns, and many-to-many associations between your core resources.

#### Identifying Relationships
Look for these patterns in your application requirements:
- **Ownership:** "Users own posts", "Companies have employees"
- **Association:** "Posts belong to categories", "Users follow other users"  
- **Many-to-Many:** "Posts can have multiple tags", "Users can belong to multiple teams"
- **Hierarchical:** "Comments can have replies", "Categories can have subcategories"
- **Temporal:** "Users can like posts", "Products can be in multiple orders"

#### Common Relationship Patterns

**Many-to-Many Relationships:**
```sql
-- Posts and Tags (many-to-many)
CREATE TABLE post_tags (
    post_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (post_id, tag_id),
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

-- Users and Roles (many-to-many with additional data)
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    granted_by BIGINT,
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (granted_by) REFERENCES users(id) ON DELETE SET NULL
);

-- Team memberships with roles
CREATE TABLE team_members (
    team_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    role ENUM('member', 'admin', 'owner') DEFAULT 'member',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (team_id, user_id),
    FOREIGN KEY (team_id) REFERENCES teams(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

**Social Interactions:**
```sql
-- Likes/Reactions (polymorphic)
CREATE TABLE likes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    likeable_type VARCHAR(50) NOT NULL,
    likeable_id BIGINT NOT NULL,
    reaction_type ENUM('like', 'love', 'laugh', 'angry', 'sad') DEFAULT 'like',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_like (user_id, likeable_type, likeable_id),
    INDEX idx_likeable (likeable_type, likeable_id)
);

-- Following/Friendship relationships
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(id) ON DELETE CASCADE,
    CHECK (follower_id != following_id)
);

-- Bookmarks/Favorites
CREATE TABLE bookmarks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    bookmarkable_type VARCHAR(50) NOT NULL,
    bookmarkable_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_bookmark (user_id, bookmarkable_type, bookmarkable_id),
    INDEX idx_bookmarkable (bookmarkable_type, bookmarkable_id)
);
```

**Business Process Relationships:**
```sql
-- Order items (products in orders)
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);

-- Project collaborators
CREATE TABLE project_collaborators (
    project_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    role ENUM('viewer', 'contributor', 'admin') DEFAULT 'contributor',
    permissions JSON, -- specific permissions
    invited_by BIGINT,
    invited_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    joined_at TIMESTAMP NULL,
    PRIMARY KEY (project_id, user_id),
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (invited_by) REFERENCES users(id) ON DELETE SET NULL
);
```

**Permission & Access Control:**
```sql
-- Roles and permissions
CREATE TABLE roles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE permissions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
);

-- Resource-specific permissions
CREATE TABLE resource_permissions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id BIGINT NOT NULL,
    permission VARCHAR(50) NOT NULL,
    granted_by BIGINT,
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (granted_by) REFERENCES users(id) ON DELETE SET NULL,
    UNIQUE KEY unique_resource_permission (user_id, resource_type, resource_id, permission)
);
```

## Design Best Practices

### Naming Conventions
- **Tables:** Use plural nouns (`users`, `posts`, `order_items`)
- **Columns:** Use snake_case (`first_name`, `created_at`, `is_active`)
- **Foreign Keys:** Use `{table_name}_id` format (`user_id`, `category_id`)
- **Junction Tables:** Combine table names (`post_tags`, `user_roles`)

### Standard Column Patterns
- **Primary Keys:** Always use `id BIGINT PRIMARY KEY AUTO_INCREMENT`
- **Timestamps:** Include `created_at` and `updated_at` on most tables
- **Soft Deletes:** Add `deleted_at TIMESTAMP NULL` when needed
- **Status Fields:** Use ENUMs for predefined states
- **Foreign Keys:** Always include proper constraints and indexes

### Relationship Design Guidelines
- **One-to-Many:** Store foreign key in the "many" table
- **Many-to-Many:** Create junction/pivot tables with composite primary keys
- **Polymorphic:** Use `{resource}_type` and `{resource}_id` columns
- **Self-Referencing:** Use `parent_id` for hierarchical relationships

### Performance Considerations
- **Indexes:** Add indexes on foreign keys, frequently queried columns, and unique constraints
- **Constraints:** Use appropriate ON DELETE and ON UPDATE actions
- **Data Types:** Choose appropriate types and sizes for your data
- **Normalization:** Balance normalization with query performance needs

## Validation Checklist

Before finalizing your database design, verify:

**✅ Foundation Requirements**
- [ ] User authentication and profile management
- [ ] Common features like comments, media, notifications
- [ ] Activity logging and audit trails
- [ ] Proper indexing on authentication tables

**✅ Resource Coverage**
- [ ] Every major application feature has a dedicated table
- [ ] Each resource has appropriate status/state management
- [ ] Proper foreign key relationships to users
- [ ] Adequate metadata and timestamps

**✅ Relationship Integrity**
- [ ] All many-to-many relationships have junction tables
- [ ] Ownership and permission relationships are clear
- [ ] Polymorphic relationships are properly structured
- [ ] Cascade behaviors are appropriate for each relationship

**✅ Technical Standards**
- [ ] Consistent naming conventions throughout
- [ ] Appropriate data types and constraints
- [ ] Performance indexes on frequently queried columns
- [ ] Proper foreign key constraints with appropriate actions

This systematic approach ensures your database design scales with your application and maintains data integrity while supporting efficient queries and operations.
