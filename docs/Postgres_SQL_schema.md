-- PAWS - SQL Schema (Postgres / Supabase-friendly)
-- Assumes Postgres >= 12 and pgcrypto extension for gen_random_uuid()

/* ---------------------------
   Extensions
   --------------------------- */
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "citext"; -- optional: case-insensitive text for email

/* ---------------------------
   Helper types & enums
   --------------------------- */
CREATE TYPE user_role AS ENUM ('customer', 'breeder', 'admin');
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded');
CREATE TYPE product_category AS ENUM ('food', 'clothes', 'health', 'accessory', 'toy', 'other');
CREATE TYPE dog_gender AS ENUM ('male','female','unknown');

/* ---------------------------
   users
   --------------------------- */
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT,
  email CITEXT UNIQUE NOT NULL,
  password_hash TEXT, -- nullable if using external auth (Supabase Auth)
  phone TEXT,
  role user_role NOT NULL DEFAULT 'customer',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);

/* trigger to auto-update updated_at */
CREATE OR REPLACE FUNCTION trigger_set_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE PROCEDURE trigger_set_timestamp();

/* ---------------------------
   breeders
   --------------------------- */
CREATE TABLE breeders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  business_name TEXT,
  license_number TEXT,
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_breeders_user ON breeders (user_id);

/* ---------------------------
   products
   --------------------------- */
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE, -- for SEO-friendly URLs
  category product_category NOT NULL DEFAULT 'other',
  price NUMERIC(12,2) NOT NULL CHECK (price >= 0),
  description TEXT,
  stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_name ON products USING gin (to_tsvector('english', name || ' ' || coalesce(description,'')));
CREATE INDEX idx_products_category ON products (category);

/* ---------------------------
   dogs
   --------------------------- */
CREATE TABLE dogs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  breeder_id UUID REFERENCES breeders(id) ON DELETE SET NULL,
  name TEXT,
  breed TEXT NOT NULL,
  age INTEGER CHECK (age >= 0), -- in months or years - decide on unit in app
  age_unit TEXT DEFAULT 'months', -- 'months' or 'years'
  gender dog_gender DEFAULT 'unknown',
  price NUMERIC(12,2) CHECK (price >= 0),
  description TEXT,
  health_certificate_url TEXT,
  vaccinated BOOLEAN DEFAULT FALSE,
  is_available BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dogs_breed ON dogs (breed);
CREATE INDEX idx_dogs_text ON dogs USING gin (to_tsvector('english', coalesce(name,'') || ' ' || coalesce(breed,'') || ' ' || coalesce(description,'')));

/* ---------------------------
   media (images/documents)
   --------------------------- */
CREATE TABLE media (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_type TEXT NOT NULL, -- 'product' | 'dog' | 'breeder_document' | 'user_avatar' | ...
  owner_id UUID NOT NULL, -- references depending on owner_type (enforced in app)
  url TEXT NOT NULL,
  filename TEXT,
  mime_type TEXT,
  size BIGINT,
  is_primary BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_owner ON media (owner_type, owner_id);

/* ---------------------------
   orders & order_items
   --------------------------- */
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
  total_amount NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
  status order_status NOT NULL DEFAULT 'pending',
  currency TEXT DEFAULT 'NGN',
  shipping_address JSONB, -- store address snapshot
  billing_address JSONB,
  payment_provider TEXT, -- 'stripe', 'paystack', etc.
  payment_reference TEXT,
  placed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_user ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);

/* order_items - can reference either a product or a dog */
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE SET NULL,
  dog_id UUID REFERENCES dogs(id) ON DELETE SET NULL,
  quantity INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
  unit_price NUMERIC(12,2) NOT NULL CHECK (unit_price >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ensure either product_id OR dog_id is set (but not both null)
ALTER TABLE order_items
ADD CONSTRAINT order_items_product_or_dog CHECK (
  (product_id IS NOT NULL AND dog_id IS NULL) OR
  (product_id IS NULL AND dog_id IS NOT NULL)
);

/* ---------------------------
   reviews
   --------------------------- */
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  dog_id UUID REFERENCES dogs(id) ON DELETE CASCADE,
  rating SMALLINT NOT NULL CHECK (rating >= 1 AND rating <= 5),
  title TEXT,
  comment TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reviews_product ON reviews (product_id);
CREATE INDEX idx_reviews_dog ON reviews (dog_id);

/* ---------------------------
   breeder_documents (licenses, vet certs)
   --------------------------- */
CREATE TABLE breeder_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  breeder_id UUID NOT NULL REFERENCES breeders(id) ON DELETE CASCADE,
  doc_type TEXT, -- 'license', 'vet_certificate', 'id', etc.
  media_id UUID REFERENCES media(id) ON DELETE SET NULL,
  uploaded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  verified BOOLEAN DEFAULT FALSE,
  verified_at TIMESTAMPTZ
);

CREATE INDEX idx_breeder_documents_breeder ON breeder_documents (breeder_id);

/* ---------------------------
   shipping / fulfillment (basic)
   --------------------------- */
CREATE TABLE shipments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  provider TEXT, -- logistics provider name
  tracking_number TEXT,
  status TEXT, -- e.g., 'label_created', 'in_transit', 'delivered'
  shipped_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

/* ---------------------------
   payments (audit)
   --------------------------- */
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id) ON DELETE SET NULL,
  provider TEXT NOT NULL,
  provider_payment_id TEXT,
  amount NUMERIC(12,2) NOT NULL CHECK (amount >= 0),
  currency TEXT NOT NULL DEFAULT 'NGN',
  status TEXT, -- 'initiated','success','failed','refunded'
  raw_response JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

/* ---------------------------
   Audit & misc
   --------------------------- */
-- simple search materialized view example (optional)
-- CREATE MATERIALIZED VIEW product_search AS
-- SELECT id, name, to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,'')) AS document FROM products;
-- CREATE INDEX product_search_idx ON product_search USING gin(document);

-- Trigger to auto-update updated_at for tables that have it (products, dogs, orders etc.)
CREATE TRIGGER trg_products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE PROCEDURE trigger_set_timestamp();

CREATE TRIGGER trg_dogs_updated_at
BEFORE UPDATE ON dogs
FOR EACH ROW
EXECUTE PROCEDURE trigger_set_timestamp();

CREATE TRIGGER trg_orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW
EXECUTE PROCEDURE trigger_set_timestamp();

-- Add other triggers as needed

/* ---------------------------
   Notes & next steps
   --------------------------- 
1. If you use Supabase Auth, you may not store password_hash in users — instead map Supabase user_id to users.id.
2. Enable Row Level Security (RLS) in Supabase and add policies for:
   - users can only edit their own row
   - breeders can edit their own breeder rows & dog/product listings
   - admins can access everything
3. You may prefer to add a `slug` generator function for products/dogs for SEO-friendly URLs.
4. Consider adding full-text search GIN indexes or integrating Algolia for fast search later.
5. For images, Supabase Storage is ideal; store URLs in media.url.
6. Back up schema and data regularly; use SQL dumps for migration to Django later.
*/
