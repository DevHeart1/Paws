# PAWS – Tech Stack

## MVP Stack (Phase 1)
**Frontend:**
- Next.js 14 (React, App Router)
- Tailwind CSS
- Framer Motion
- React Query / SWR
- TypeScript

**Backend (BaaS):**
- Supabase
  - PostgreSQL Database
  - Supabase Auth
  - Supabase Storage (images)
  - Row Level Security for DB access control
- Supabase Edge Functions for server-side logic

**Payments:**
- Stripe (international)
- Paystack (Nigeria)

**Hosting:**
- Vercel (Next.js frontend)
- Supabase Cloud (backend)

**Monitoring & Analytics:**
- Sentry (error tracking)
- Google Analytics (traffic insights)

---

## Future Stack (Phase 2+)
**Backend:**
- Django + Django REST Framework
- PostgreSQL (self-hosted on AWS RDS or Railway)
- Celery + Redis (background tasks)
- AWS S3 (media storage)
- Dockerized deployment

**Frontend:**
- Continue with Next.js (API calls switch to Django endpoints)

**Infrastructure:**
- AWS EC2 / Render / Railway for hosting
- Cloudflare for CDN and security

---

## Reasoning for Stack Choice
- Supabase offers a managed PostgreSQL backend with minimal ops cost.
- Next.js provides SEO-friendly, fast-loading e-commerce pages.
- The stack is migration-friendly to Django + PostgreSQL when scaling.
