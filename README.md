# Curate Backend

A secure, scalable backend for the Curate content curation app built on Supabase. This backend provides user authentication, content storage, collections, and tagging functionality with row-level security.

## Features

- 🔐 **Secure Authentication** - Built on Supabase Auth with automatic profile creation
- 📦 **Content Storage** - Save articles, videos, tweets, and other web content
- 📁 **Collections** - Organize content into custom collections
- 🏷️ **Tagging System** - Categorize items with user-created tags
- 🔒 **Row Level Security** - Users can only access their own data
- 🚀 **Optimized Queries** - Indexed for common access patterns

## Tech Stack

- **Database**: PostgreSQL (via Supabase)
- **Authentication**: Supabase Auth
- **Security**: Row Level Security (RLS)
- **Language**: SQL

## Database Schema

### Tables

#### `profiles`
Extends Supabase `auth.users` with additional profile information.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key, references `auth.users.id` |
| `email` | TEXT | User email address |
| `name` | TEXT | User display name |
| `avatar_url` | TEXT | URL to user avatar (nullable) |
| `created_at` | TIMESTAMPTZ | Profile creation timestamp |

**Relationships**: One-to-one with `auth.users`

---

#### `items`
Stores all saved content items (articles, videos, tweets, etc.).

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `user_id` | UUID | References `profiles.id` |
| `url` | TEXT | Original content URL |
| `title` | TEXT | Content title |
| `content` | TEXT | Full extracted text (nullable) |
| `summary` | TEXT | AI-generated summary (nullable) |
| `source_type` | TEXT | Type: `article`, `video`, `tweet`, `other` |
| `thumbnail_url` | TEXT | Preview image URL (nullable) |
| `metadata` | JSONB | Additional metadata (author, duration, etc.) |
| `created_at` | TIMESTAMPTZ | When item was saved |
| `updated_at` | TIMESTAMPTZ | Last update timestamp |

**Relationships**: 
- Belongs to one `profile`
- Can belong to many `collections` (via `collection_items`)
- Can have many `tags` (via `item_tags`)

**Metadata Example**:
```json
{
  "author": "John Doe",
  "publication_date": "2025-01-15",
  "duration": "5:30",
  "word_count": 1500,
  "domain": "example.com"
}
```

---

#### `collections`
User-created collections/folders for organizing items.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `user_id` | UUID | References `profiles.id` |
| `name` | TEXT | Collection name |
| `description` | TEXT | Optional description (nullable) |
| `created_at` | TIMESTAMPTZ | Creation timestamp |

**Relationships**:
- Belongs to one `profile`
- Can contain many `items` (via `collection_items`)

---

#### `collection_items`
Many-to-many join table between collections and items.

| Column | Type | Description |
|--------|------|-------------|
| `collection_id` | UUID | References `collections.id` |
| `item_id` | UUID | References `items.id` |
| `added_at` | TIMESTAMPTZ | When item was added to collection |

**Primary Key**: `(collection_id, item_id)`

**Cascade Behavior**: Deleting a collection or item automatically removes the association.

---

#### `tags`
User-created tags for categorizing items.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `user_id` | UUID | References `profiles.id` |
| `name` | TEXT | Tag name |
| `created_at` | TIMESTAMPTZ | Creation timestamp |

**Constraints**: Tag names must be unique per user

**Relationships**:
- Belongs to one `profile`
- Can be applied to many `items` (via `item_tags`)

---

#### `item_tags`
Many-to-many join table between items and tags.

| Column | Type | Description |
|--------|------|-------------|
| `item_id` | UUID | References `items.id` |
| `tag_id` | UUID | References `tags.id` |

**Primary Key**: `(item_id, tag_id)`

**Cascade Behavior**: Deleting an item or tag automatically removes the association.

---

## Security

### Row Level Security (RLS)

All tables have RLS enabled to ensure data isolation between users. The policies enforce:

- **Profiles**: Users can only view/edit their own profile
- **Items**: Users can only access items they created
- **Collections**: Users can only access their own collections
- **Tags**: Users can only access their own tags
- **Collection Items**: Users can only link their own items to their own collections
- **Item Tags**: Users can only tag their own items with their own tags

### Authentication Flow

1. User signs up via Supabase Auth
2. `handle_new_user()` trigger automatically creates a profile
3. All subsequent queries are filtered by `auth.uid()` via RLS policies

---

## Setup Instructions

### Prerequisites

- [Supabase CLI](https://supabase.com/docs/guides/cli) installed
- Supabase project created (already done)
- Connection details in `.env.local`

### Installation

1. **Install Supabase CLI** (if not already installed):
   ```bash
   npm install -g supabase
   ```

2. **Link to your Supabase project**:
   ```bash
   supabase link --project-ref <your-project-ref>
   ```
   
   Your project ref can be found in the `.env.local` file or in your Supabase dashboard URL.

3. **Run migrations**:
   ```bash
   supabase db push
   ```
   
   This will apply all migrations in the `supabase/migrations/` directory to your database.

### Verify Setup

After running migrations, verify in your Supabase dashboard:

1. Go to **Table Editor** - you should see 6 tables
2. Go to **Authentication** → **Policies** - each table should have RLS policies
3. Try signing up a test user - a profile should be auto-created

---

## Running Migrations

### Apply All Migrations
```bash
supabase db push
```

### Create a New Migration
```bash
supabase migration new <migration_name>
```

### Reset Database (⚠️ Destructive)
```bash
supabase db reset
```

This will drop all tables and re-run all migrations from scratch.

---

## Common Queries

### Get all items for the current user
```sql
SELECT * FROM items
WHERE user_id = auth.uid()
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### Get items in a specific collection
```sql
SELECT i.* FROM items i
JOIN collection_items ci ON i.id = ci.item_id
WHERE ci.collection_id = $1 AND i.user_id = auth.uid()
ORDER BY ci.added_at DESC;
```

### Get items with a specific tag
```sql
SELECT i.* FROM items i
JOIN item_tags it ON i.id = it.item_id
WHERE it.tag_id = $1 AND i.user_id = auth.uid()
ORDER BY i.created_at DESC;
```

### Get all tags for an item
```sql
SELECT t.* FROM tags t
JOIN item_tags it ON t.id = it.tag_id
WHERE it.item_id = $1 AND t.user_id = auth.uid();
```

### Search items by title or content
```sql
SELECT * FROM items
WHERE user_id = auth.uid()
AND (title ILIKE '%search_term%' OR content ILIKE '%search_term%')
ORDER BY created_at DESC;
```

### Get items by source type
```sql
SELECT * FROM items
WHERE user_id = auth.uid() AND source_type = 'article'
ORDER BY created_at DESC;
```

---

## Environment Variables

The `.env.local` file contains your Supabase connection details:

```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

**⚠️ Security Notes**:
- Never commit `.env.local` to version control
- Use `SUPABASE_ANON_KEY` in client applications
- Only use `SUPABASE_SERVICE_ROLE_KEY` in secure server environments
- The service role key bypasses RLS - use with extreme caution

---

## Database Indexes

The schema includes optimized indexes for common query patterns:

- **Items**: `user_id`, `created_at`, `user_id + created_at`, `source_type`
- **Collections**: `user_id`, `created_at`
- **Tags**: `user_id`, `name`
- **Collection Items**: `collection_id`, `item_id`, `added_at`
- **Item Tags**: `item_id`, `tag_id`

---

## Automatic Features

### Profile Creation
When a user signs up, a profile is automatically created via the `handle_new_user()` trigger.

### Updated Timestamps
The `items` table automatically updates the `updated_at` column on every update via a trigger.

---

## File Structure

```
backend/
├── .env.local                    # Supabase connection details (do not commit)
├── .env.example                  # Example environment file
├── .gitignore                    # Git ignore rules
├── README.md                     # This file
└── supabase/
    ├── migrations/
    │   └── 001_initial_schema.sql  # Initial database schema
    └── schema.sql                   # Complete schema reference
```

---

## Next Steps

After setting up the backend:

1. **Test Authentication**: Sign up a test user and verify profile creation
2. **Test Permissions**: Try accessing data from different users to verify RLS works
3. **Build Frontend**: Connect your frontend to the Supabase client
4. **Add Features**: Consider adding full-text search, sharing, or AI summarization

---

## Troubleshooting

### Migration fails to apply
- Ensure you're linked to the correct project: `supabase projects list`
- Check for syntax errors in SQL files
- Verify you have the correct permissions

### RLS policies not working
- Ensure RLS is enabled on all tables
- Check that `auth.uid()` returns a valid user ID
- Verify the user is authenticated in your application

### Cannot insert data
- Ensure the user is authenticated
- Check that foreign key references exist (e.g., profile must exist before creating items)
- Verify RLS policies allow the operation

---

## Support

For issues related to:
- **Supabase Platform**: [Supabase Documentation](https://supabase.com/docs)
- **PostgreSQL**: [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- **Supabase CLI**: [CLI Documentation](https://supabase.com/docs/guides/cli)

---

## License

This backend is part of the Curate content curation app.
#   C u r a t e - B a c k e n d  
 