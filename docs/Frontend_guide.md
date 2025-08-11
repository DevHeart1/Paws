# PAWS – Frontend Guidelines

## 1. Framework & Tools
- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS
- **State Management:** Zustand or React Query (for server state)
- **Animations:** Framer Motion
- **Form Handling:** React Hook Form + Zod validation

---

## 2. File Structure
/src
/app # Next.js App Router pages
/components # Reusable UI components
/features # Feature-specific components (cart, products, breeds)
/hooks # Custom hooks
/lib # Utility functions (API client, constants)
/styles # Tailwind config & global styles
/types # TypeScript types/interfaces


---

## 3. UI/UX Guidelines
- **Mobile-first design** — test all breakpoints.
- **Accessible components** (aria-labels, proper contrast ratios).
- Consistent spacing using Tailwind's spacing scale.
- Buttons: rounded, high-contrast for CTAs.
- Product cards: image + name + price + quick-view option.
- Use **Skeleton Loading** for image-heavy pages.
- Keep navigation minimal with clear “Shop”, “Breeds”, “About”, and “Cart” sections.

---

## 4. API Interaction
- Use a central `apiClient.ts` for Supabase and REST calls.
- All API calls should be wrapped in error handling with user-friendly messages.
- Use SWR or React Query for data fetching and caching.

---

## 5. SEO & Performance
- Use `generateMetadata()` in Next.js for dynamic SEO titles/descriptions.
- Optimize all images using Next.js `Image` component.
- Prefetch product detail pages for faster navigation.

