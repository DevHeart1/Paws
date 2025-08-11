# PAWS – Backend Structure

## 1. Database Schema (PostgreSQL)
**Tables:**
- **users**
  - id (UUID, PK)
  - name
  - email (unique)
  - password_hash
  - role (customer, breeder, admin)
  - created_at

- **breeders**
  - id (UUID, PK)
  - user_id (FK to users)
  - license_number
  - verified (boolean)
  - created_at

- **dogs**
  - id (UUID, PK)
  - breeder_id (FK to breeders)
  - breed
  - age
  - gender
  - price
  - description
  - health_cert_url
  - images (array of URLs)
  - created_at

- **products**
  - id (UUID, PK)
  - seller_id (FK to users)
  - name
  - category (food, clothes, health, accessories)
  - price
  - description
  - stock
  - images (array of URLs)
  - created_at

- **orders**
  - id (UUID, PK)
  - user_id (FK to users)
  - total_amount
  - status (pending, paid, shipped, delivered)
  - created_at

- **order_items**
  - id (UUID, PK)
  - order_id (FK to orders)
  - product_id (nullable FK to products)
  - dog_id (nullable FK to dogs)
  - quantity
  - price

- **reviews**
  - id (UUID, PK)
  - user_id (FK to users)
  - product_id (nullable)
  - dog_id (nullable)
  - rating (1-5)
  - comment
  - created_at

---

## 2. Supabase Setup
- Enable Row Level Security (RLS) for all tables.
- Policies:
  - Users can only edit their own data.
  - Breeders can only edit their listings.
  - Admin can edit all data.
- Storage buckets:
  - `dog-images`
  - `product-images`
  - `documents` (certificates, breeder licenses)

---

## 3. API Structure (Next.js API Routes)
/api
/auth
login.ts
register.ts
/dogs
list.ts
create.ts
[id].ts
/products
list.ts
create.ts
[id].ts
/orders
create.ts
list.ts
update-status.ts
/breeders
verify.ts


---

## 4. Future Django Migration
- Directly import Supabase PostgreSQL schema into Django.
- Replace Supabase auth/storage with Django auth + S3.
- Keep frontend API endpoints consistent to reduce breakage.
