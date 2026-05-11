# DevOverFlow - Comprehensive Code Audit Report

> This document contains all identified issues from a thorough code audit of the DevOverFlow repository.
> Each section represents a separate issue that should be created in the GitHub issue tracker.

---

## Issue #1: Bug - Account PUT Endpoint Passes SafeParseResult Instead of Validated Data

**Priority:** Critical  
**Labels:** bug, api, data-integrity  
**Affected Files:** `src/app/api/accounts/[id]/route.ts`

### Description
In the PUT handler, the entire `validatedData` (Zod SafeParseResult object) is passed to `findByIdAndUpdate` instead of `validatedData.data`.

### Current Behavior
```typescript
const updatedAccount = await Account.findByIdAndUpdate(id, validatedData, { new: true });
```
MongoDB receives `{ success: true, data: {...}, error: undefined }` instead of the actual update fields.

### Expected Behavior
```typescript
const updatedAccount = await Account.findByIdAndUpdate(id, validatedData.data, { new: true });
```

### Steps to Reproduce
1. Send a PUT request to `/api/accounts/:id` with valid update data
2. Observe that the document is not updated correctly or gets corrupted

---

## Issue #2: Security - Wildcard Hostname in Next.js Image Configuration

**Priority:** High  
**Labels:** security, configuration  
**Affected Files:** `next.config.ts`

### Description
The `next.config.ts` has a wildcard `hostname: "*"` in the `images.remotePatterns` array, which allows loading images from ANY external domain.

### Current Behavior
```typescript
images: {
  remotePatterns: [
    // ... specific patterns ...
    { protocol: "https", hostname: "*", port: "" }, // DANGEROUS
  ],
}
```

### Expected Behavior
Only explicitly trusted domains should be whitelisted. Remove the wildcard entry and add specific domains as needed (e.g., `flagsapi.com` for job cards, employer logo domains).

### Security Impact
- Enables SSRF attacks through image optimization
- Can be used to proxy arbitrary content
- Violates principle of least privilege

---

## Issue #3: Security - API Key Exposed via NEXT_PUBLIC Environment Variable

**Priority:** Critical  
**Labels:** security, api-key-leak  
**Affected Files:** `src/lib/actions/job.action.ts`

### Description
The RapidAPI key is stored in `NEXT_PUBLIC_RAPID_API_KEY`, which means it's exposed to the client-side JavaScript bundle.

### Current Behavior
```typescript
const headers = {
  "X-RapidAPI-Key": process.env.NEXT_PUBLIC_RAPID_API_KEY ?? "",
  "X-RapidAPI-Host": "jsearch.p.rapidapi.com",
};
```

### Expected Behavior
The API key should be stored as a server-only environment variable (without `NEXT_PUBLIC_` prefix) since this function runs server-side. Use `RAPID_API_KEY` instead.

### Security Impact
- Anyone can extract the API key from the browser bundle
- Key can be abused for unauthorized API calls
- May result in unexpected billing

---

## Issue #4: Security - Unprotected API Routes Allow Unauthorized Data Access

**Priority:** High  
**Labels:** security, api, authentication  
**Affected Files:** `src/app/api/users/route.ts`, `src/app/api/accounts/route.ts`, `src/app/api/users/[id]/route.ts`, `src/app/api/accounts/[id]/route.ts`

### Description
All REST API routes (GET, DELETE, PUT) lack authentication/authorization checks. Any unauthenticated user can list all users, delete accounts, or modify user data.

### Current Behavior
- `GET /api/users` returns ALL user records without authentication
- `GET /api/accounts` returns ALL account records (including password hashes)
- `DELETE /api/users/:id` allows anyone to delete any user
- `DELETE /api/accounts/:id` allows anyone to delete any account

### Expected Behavior
- All mutating endpoints (POST, PUT, DELETE) should require authentication
- Admin-only endpoints should check for admin role
- User-specific operations should verify the requesting user owns the resource
- Account data should never expose password hashes

### Suggested Solution
Add auth middleware or check `auth()` session at the top of each handler.

---

## Issue #5: Bug - Duplicate Tag Action Files Causing Confusion

**Priority:** Medium  
**Labels:** bug, code-quality, duplicate-code  
**Affected Files:** `src/lib/actions/tag.action.ts`, `src/lib/actions/tag.actions.ts`

### Description
There are two tag action files with overlapping functionality:
- `tag.action.ts` - contains `getTags` and `getTagQuestions`
- `tag.actions.ts` - contains `getTags` and `getTopTags`

Both export a `getTags` function with nearly identical implementations. The RightSidebar imports from `tag.actions.ts` while the Tags page imports from `tag.action.ts`.

### Expected Behavior
Consolidate into a single file with all tag-related actions.

### Steps to Reproduce
1. Search for `getTags` in the codebase
2. Note two different files export the same function name

---

## Issue #6: Bug - getUserStats Crashes When User Has No Questions or Answers

**Priority:** High  
**Labels:** bug, runtime-error  
**Affected Files:** `src/lib/actions/user.action.ts`

### Description
In `getUserStats`, the aggregation pipeline results are destructured without null checks. If a user has no questions or answers, the aggregation returns an empty array, and destructuring `[questionStats]` yields `undefined`.

### Current Behavior
```typescript
const [questionStats] = await Question.aggregate([...]);
const [answerStats] = await Answer.aggregate([...]);

// This crashes with "Cannot read properties of undefined (reading 'count')"
const badges = assignBadges({
  criteria: [
    { type: "ANSWER_COUNT", count: answerStats.count },
    { type: "QUESTION_COUNT", count: questionStats.count },
    ...
  ],
});
```

### Expected Behavior
Add null checks with defaults:
```typescript
const questionStats = (await Question.aggregate([...]))[0] || { count: 0, upvotes: 0, views: 0 };
const answerStats = (await Answer.aggregate([...]))[0] || { count: 0, upvotes: 0 };
```

---

## Issue #7: Bug - getSavedQuestions Crashes When Collection is Empty

**Priority:** Medium  
**Labels:** bug, runtime-error  
**Affected Files:** `src/lib/actions/collection.action.ts`

### Description
In `getSavedQuestions`, when there are no saved questions, the `$count` pipeline stage returns an empty array. Accessing `totalCount.count` throws an error.

### Current Behavior
```typescript
const [totalCount] = await Collection.aggregate([...pipeline, { $count: "count" }]);
const isNext = totalCount.count > skip + questions.length; // Crashes if totalCount is undefined
```

### Expected Behavior
```typescript
const [totalCount] = await Collection.aggregate([...pipeline, { $count: "count" }]);
const isNext = (totalCount?.count || 0) > skip + questions.length;
```

---

## Issue #8: Code Quality - Console.log Statements Left in Production Code

**Priority:** Low  
**Labels:** code-quality, cleanup  
**Affected Files:** `src/lib/actions/general.action.ts`, `src/lib/actions/job.action.ts`, `src/components/GlobalResult.tsx`, `src/components/filters/JobFilter.tsx`, `src/components/forms/QuestionForm.tsx`

### Description
Multiple `console.log` statements are left in production code:

1. `general.action.ts`: `console.log("QUERY", params)` and `console.log(results)`
2. `job.action.ts` (jobs page): `console.log("countries", countries)`
3. `GlobalResult.tsx`: `console.log(res)` and `console.log(error)`
4. `JobFilter.tsx`: `console.log({ countriesList })`
5. `QuestionForm.tsx`: `console.log(field, e)`

### Expected Behavior
Remove all debug console.log statements or replace with proper logger calls using the existing pino logger.

---

## Issue #9: Configuration - TypeScript and ESLint Errors Ignored During Builds

**Priority:** Medium  
**Labels:** configuration, code-quality, tech-debt  
**Affected Files:** `next.config.ts`

### Description
The Next.js configuration suppresses both TypeScript and ESLint errors during builds:
```typescript
eslint: { ignoreDuringBuilds: true },
typescript: { ignoreBuildErrors: true },
```

### Current Behavior
Build succeeds even with type errors and lint violations, masking potential bugs.

### Expected Behavior
Remove these overrides and fix all underlying TypeScript and ESLint errors. These flags should only be used temporarily during migration, not permanently.

---

## Issue #10: Bug - TypeScript Suppress Comment in GlobalSearch Component

**Priority:** Low  
**Labels:** bug, typescript  
**Affected Files:** `src/components/search/GlobalSearch.tsx`

### Description
A `@ts-expect-error` comment is used to suppress a type error for `searchContainerRef.current?.contains(event.target)`. This indicates the ref is not properly typed.

### Current Behavior
```typescript
const searchContainerRef = useRef(null);
// @ts-expect-error Property 'contains' does not exist
!searchContainerRef.current?.contains(event.target)
```

### Expected Behavior
```typescript
const searchContainerRef = useRef<HTMLDivElement>(null);
// No ts-expect-error needed
!searchContainerRef.current?.contains(event.target as Node)
```



---

## Issue #11: Security - Job API Fetches Over HTTP (ip-api.com)

**Priority:** Medium  
**Labels:** security, network  
**Affected Files:** `src/lib/actions/job.action.ts`

### Description
The `fetchLocation` function makes a request over plain HTTP to `http://ip-api.com/json/`, which exposes the request to man-in-the-middle attacks.

### Current Behavior
```typescript
const response = await fetch("http://ip-api.com/json/?fields=country");
```

### Expected Behavior
Use HTTPS or a different geolocation service that supports HTTPS. The free tier of ip-api.com only supports HTTP, so consider using an alternative like `ipapi.co` which provides HTTPS.

---

## Issue #12: Bug - HomeFilter Has Typo "Recommeded" Instead of "Recommended"

**Priority:** Low  
**Labels:** bug, ui, typo  
**Affected Files:** `src/components/filters/HomeFilter.tsx`

### Description
The HomeFilter component has a hardcoded filters array with a typo: "Recommeded" instead of "Recommended".

### Current Behavior
```typescript
const filters = [
  { name: "Newest", value: "newest" },
  { name: "Popular", value: "popular" },
  { name: "Unanswered", value: "unanswered" },
  { name: "Recommeded", value: "recommended" }, // Typo in name
];
```

### Expected Behavior
```typescript
{ name: "Recommended", value: "recommended" },
```

---

## Issue #13: Code Quality - Duplicate Filter Definitions Between HomeFilter and filters.ts

**Priority:** Low  
**Labels:** code-quality, maintainability  
**Affected Files:** `src/components/filters/HomeFilter.tsx`, `src/constants/filters.ts`

### Description
The HomeFilter component defines its own local `filters` array instead of using the `HomePageFilters` constant from `src/constants/filters.ts`. This creates duplication and inconsistency.

### Expected Behavior
Import and use `HomePageFilters` from `@/constants/filters` instead of defining filters locally.

---

## Issue #14: Bug - RootLayout Has CSS Typo "realtive" Instead of "relative"

**Priority:** Low  
**Labels:** bug, ui, typo  
**Affected Files:** `src/app/(root)/layout.tsx`

### Description
The root layout has a CSS class typo that prevents the relative positioning from being applied.

### Current Behavior
```tsx
<main className="background-light850_dark100 realtive">
```

### Expected Behavior
```tsx
<main className="background-light850_dark100 relative">
```

---

## Issue #15: Missing Feature - No .env.example File for Environment Variables

**Priority:** Medium  
**Labels:** documentation, developer-experience  
**Affected Files:** Root directory

### Description
The project requires multiple environment variables (MONGODB_URI, NEXT_PUBLIC_API_BASE_URL, NEXT_PUBLIC_RAPID_API_KEY, OAuth secrets, OpenAI key) but there's no `.env.example` file documenting them.

### Expected Behavior
Create a `.env.example` file listing all required environment variables with descriptions:
```
MONGODB_URI=                    # MongoDB connection string
NEXT_PUBLIC_API_BASE_URL=       # Base URL for API (e.g., http://localhost:3000/api)
RAPID_API_KEY=                  # RapidAPI key for job search
AUTH_SECRET=                    # NextAuth secret
AUTH_GITHUB_ID=                 # GitHub OAuth App ID
AUTH_GITHUB_SECRET=             # GitHub OAuth App Secret
AUTH_GOOGLE_ID=                 # Google OAuth Client ID
AUTH_GOOGLE_SECRET=             # Google OAuth Client Secret
OPENAI_API_KEY=                 # OpenAI API key for AI answers
```

---

## Issue #16: Missing Feature - No Test Suite

**Priority:** High  
**Labels:** testing, code-quality  
**Affected Files:** Entire project

### Description
The project has zero test files. No unit tests, integration tests, or end-to-end tests exist for any component, action, utility, or API route.

### Expected Behavior
Add a testing framework (Jest/Vitest for unit tests, Playwright/Cypress for E2E) and create tests for:
- Utility functions (`lib/utils.ts`, `lib/url.ts`)
- Validation schemas (`lib/validations.ts`)
- Server actions (question, answer, vote, collection actions)
- API routes
- Critical UI components

---

## Issue #17: Documentation - README Contains Only Default Next.js Template Content

**Priority:** Medium  
**Labels:** documentation  
**Affected Files:** `README.md`

### Description
The README.md is the default create-next-app template and provides no project-specific information.

### Expected Behavior
Update README with:
- Project description and features
- Tech stack
- Prerequisites
- Setup instructions
- Environment variables documentation
- Architecture overview
- Contributing guidelines

---

## Issue #18: Performance - No Database Indexes Defined on Mongoose Models

**Priority:** High  
**Labels:** performance, database  
**Affected Files:** `src/database/question.model.ts`, `src/database/answer.model.ts`, `src/database/vote.model.ts`, `src/database/collection.model.ts`, `src/database/interaction.model.ts`, `src/database/tag-question.model.ts`

### Description
No compound indexes are defined on frequently queried field combinations. This will cause full collection scans as the database grows.

### Missing Indexes
- **Vote model**: Compound index on `{ author, actionId, actionType }` (used in `hasVoted` and `createVote`)
- **Collection model**: Compound index on `{ author, question }` (used in `hasSavedQuestion`)
- **TagQuestion model**: Compound index on `{ tag, question }` (used in queries and deletes)
- **Interaction model**: Compound index on `{ user, actionType, action }` (used in recommendations)
- **Answer model**: Index on `{ question }` (used in `getAnswers`)
- **Question model**: Index on `{ author }` (used in user profile queries)

### Suggested Solution
Add indexes to schema definitions:
```typescript
VoteSchema.index({ author: 1, actionId: 1, actionType: 1 }, { unique: true });
CollectionSchema.index({ author: 1, question: 1 }, { unique: true });
```

---

## Issue #19: Performance - No Caching Strategy for Frequently Accessed Data

**Priority:** Medium  
**Labels:** performance, caching  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/lib/actions/tag.actions.ts`

### Description
Hot questions and top tags are fetched on every page load of the RightSidebar (which renders on all pages) without any caching strategy beyond React's `cache()` for `getQuestion`.

### Expected Behavior
Implement caching for:
- `getHotQuestions()` - Use `unstable_cache` with a revalidation period
- `getTopTags()` - Use `unstable_cache` with a revalidation period
- Consider adding `revalidateTag` for invalidation on question/tag mutations

---

## Issue #20: Bug - Auth Layout Image Missing Leading Slash

**Priority:** Low  
**Labels:** bug, ui  
**Affected Files:** `src/app/(auth)/layout.tsx`

### Description
The logo image source is missing a leading slash, which may cause the image to not load correctly on nested routes.

### Current Behavior
```tsx
<Image src="images/site-logo.svg" ... />
```

### Expected Behavior
```tsx
<Image src="/images/site-logo.svg" ... />
```



---

## Issue #21: Security - RegEx Injection in Tag Creation

**Priority:** High  
**Labels:** security, injection  
**Affected Files:** `src/lib/actions/question.action.ts`

### Description
User-supplied tag names are interpolated directly into a RegExp without escaping, allowing ReDoS attacks or unexpected matching behavior.

### Current Behavior
```typescript
const existingTag = await Tag.findOneAndUpdate(
  { name: { $regex: new RegExp(`^${tag}$`, "i") } },
  ...
);
```

If a user provides a tag like `.*`, `(a+)+$`, or other regex special characters, this can cause:
- Matching unintended tags
- ReDoS (Regular Expression Denial of Service)

### Expected Behavior
Escape the tag string before using it in a regex:
```typescript
const escapedTag = tag.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
const existingTag = await Tag.findOneAndUpdate(
  { name: { $regex: new RegExp(`^${escapedTag}$`, "i") } },
  ...
);
```

---

## Issue #22: Bug - GlobalFilter Calls router.push Twice on Deselect

**Priority:** Low  
**Labels:** bug, ui  
**Affected Files:** `src/components/filters/GlobalFilter.tsx`

### Description
When deselecting an active filter type, `router.push` is called twice - once inside the `if (active === item)` block and once at the end of the function.

### Current Behavior
```typescript
const handleTypeClick = (item: string) => {
  let newUrl = "";
  if (active === item) {
    setActive("");
    newUrl = removeKeysFromUrlQuery({ ... });
    router.push(newUrl, { scroll: false }); // First push
  } else {
    setActive(item);
    newUrl = formUrlQuery({ ... });
  }
  router.push(newUrl, { scroll: false }); // Second push (always runs)
};
```

### Expected Behavior
Add `return` after the first `router.push` or move the second push inside the `else` block.

---

## Issue #23: Performance - View Counter Incremented Without Rate Limiting

**Priority:** Medium  
**Labels:** performance, security  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/app/(root)/questions/[id]/page.tsx`

### Description
The `incrementViews` function is called on every page load without any deduplication or rate limiting. A user can artificially inflate view counts by refreshing the page.

### Current Behavior
Every visit to a question page calls `incrementViews` unconditionally via `after()`.

### Expected Behavior
Implement view deduplication:
- Track views per user/session using the Interaction model
- Only increment if the same user hasn't viewed in the last X hours
- Consider using IP-based or session-based tracking for anonymous users

---

## Issue #24: Bug - hasVoted Returns success: false When No Vote Exists

**Priority:** Medium  
**Labels:** bug, logic-error  
**Affected Files:** `src/lib/actions/vote.action.ts`

### Description
When a user hasn't voted on content, the function returns `{ success: false, data: { hasUpvoted: false, hasDownvoted: false } }`. Returning `success: false` implies an error occurred, but this is a valid state.

### Current Behavior
```typescript
if (!vote)
  return {
    success: false, // Misleading - no error occurred
    data: { hasUpvoted: false, hasDownvoted: false },
  };
```

### Expected Behavior
```typescript
if (!vote)
  return {
    success: true, // No error - user simply hasn't voted
    data: { hasUpvoted: false, hasDownvoted: false },
  };
```

This causes the Votes component to not show vote status correctly since it checks `success && hasUpvoted`.

---

## Issue #25: Missing Feature - No Loading States for Most Pages

**Priority:** Medium  
**Labels:** ux, missing-feature  
**Affected Files:** `src/app/(root)/` (all page directories except community)

### Description
Only the community page has a `loading.tsx` file. All other pages (home, tags, collection, jobs, profile, questions) lack loading states, causing poor UX during server-side data fetching.

### Expected Behavior
Add `loading.tsx` skeleton components for:
- Home page (`/`)
- Tags page (`/tags`)
- Tag detail page (`/tags/[id]`)
- Collection page (`/collection`)
- Jobs page (`/jobs`)
- Profile page (`/profile/[id]`)
- Question detail page (`/questions/[id]`)

---

## Issue #26: Missing Feature - No Error Boundaries for Most Pages

**Priority:** Medium  
**Labels:** ux, error-handling  
**Affected Files:** `src/app/(root)/` (all page directories except community)

### Description
Only the community page has an `error.tsx` file. Other pages will show the default Next.js error page instead of a user-friendly error state.

### Expected Behavior
Add `error.tsx` components with proper styling for all route groups, or at minimum a root-level error boundary that matches the app's design system.

---

## Issue #27: Bug - Votes Component Uses Image onClick Instead of Button for Accessibility

**Priority:** Medium  
**Labels:** accessibility, ux  
**Affected Files:** `src/components/votes/Votes.tsx`, `src/components/questions/SaveQuestion.tsx`

### Description
Vote and save actions use `<Image onClick={...}>` which is not keyboard accessible and doesn't convey interactive semantics to screen readers.

### Current Behavior
```tsx
<Image
  src={...}
  onClick={() => !isLoading && handleVote("upvote")}
  aria-label="Upvote"
/>
```

### Expected Behavior
Wrap interactive images in `<button>` elements:
```tsx
<button onClick={() => handleVote("upvote")} disabled={isLoading} aria-label="Upvote">
  <Image src={...} />
</button>
```

---

## Issue #28: Security - AI Answer Endpoint Lacks Authentication

**Priority:** High  
**Labels:** security, api  
**Affected Files:** `src/app/api/ai/answers/route.ts`

### Description
The AI answer generation endpoint has no authentication check. Anyone can call it directly to generate AI answers, potentially abusing the OpenAI API key and incurring costs.

### Current Behavior
The POST handler validates input but never checks if the request comes from an authenticated user.

### Expected Behavior
Add authentication check:
```typescript
import { auth } from "@/auth";

export async function POST(req: Request) {
  const session = await auth();
  if (!session) {
    return NextResponse.json({ success: false, error: { message: "Unauthorized" } }, { status: 401 });
  }
  // ... rest of handler
}
```

---

## Issue #29: Performance - fetchCountries Loads ALL Countries on Every Jobs Page Visit

**Priority:** Medium  
**Labels:** performance, optimization  
**Affected Files:** `src/lib/actions/job.action.ts`, `src/app/(root)/jobs/page.tsx`

### Description
Every visit to the Jobs page fetches all ~250 countries from `restcountries.com/v3.1/all`. This data is static and should be cached or stored locally.

### Expected Behavior
Options:
1. Store country list as a static JSON file in the project
2. Use `unstable_cache` with a long revalidation period (e.g., 24 hours)
3. Fetch only country names with `?fields=name` to reduce payload size

---

## Issue #30: Bug - Jobs Page Constructs Invalid Query When No Parameters

**Priority:** Medium  
**Labels:** bug, logic-error  
**Affected Files:** `src/app/(root)/jobs/page.tsx`

### Description
The query construction has a logical error that produces invalid search queries.

### Current Behavior
```typescript
const jobs = await fetchJobs({
  query: `${query}, ${location}` || `Software Engineer in ${userLocation}`,
  page: page ?? 1,
});
```

When `query` is undefined and `location` is undefined, this produces `"undefined, undefined"` (a truthy string), so the fallback never triggers.

### Expected Behavior
```typescript
const searchQuery = query || location 
  ? `${query || 'Software Engineer'}${location ? ' in ' + location : ''}`
  : `Software Engineer in ${userLocation}`;

const jobs = await fetchJobs({ query: searchQuery, page: page ?? 1 });
```

---

## Issue #31: Architecture - proxy.ts File Has Unclear Purpose

**Priority:** Low  
**Labels:** code-quality, documentation  
**Affected Files:** `proxy.ts`

### Description
The root-level `proxy.ts` file only exports `auth` from the auth module. Its purpose is unclear - it appears to be intended as Next.js middleware but isn't named `middleware.ts`.

### Current Behavior
```typescript
import { auth } from "@/auth";
export default auth;
```

### Expected Behavior
Either:
1. Rename to `middleware.ts` if it's meant to be Next.js middleware
2. Remove if it's unused
3. Add documentation explaining its purpose

---

## Issue #32: Bug - ThemeProvider Import Path Uses Deprecated Types Location

**Priority:** Low  
**Labels:** bug, dependency  
**Affected Files:** `src/context/Theme.tsx`

### Description
The import from `next-themes/dist/types` references internal package paths that may break with package updates.

### Current Behavior
```typescript
import { ThemeProviderProps } from "next-themes/dist/types";
```

### Expected Behavior
Use the public API or infer types:
```typescript
import { ThemeProvider as NextThemesProvider } from "next-themes";
type ThemeProviderProps = React.ComponentProps<typeof NextThemesProvider>;
```



---

## Issue #33: Missing Feature - No Input Sanitization for Search Queries Used in MongoDB Regex

**Priority:** High  
**Labels:** security, injection  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/lib/actions/user.action.ts`, `src/lib/actions/general.action.ts`, `src/lib/actions/tag.action.ts`

### Description
User-provided search queries are directly used in MongoDB `$regex` operators without escaping special regex characters. This allows NoSQL regex injection.

### Current Behavior
```typescript
if (query) {
  filterQuery.$or = [
    { title: { $regex: query, $options: "i" } },
    { content: { $regex: query, $options: "i" } },
  ];
}
```

If a user searches for `.*` or `(a+)+$`, it could cause ReDoS or return unintended results.

### Expected Behavior
Escape regex special characters:
```typescript
const escapeRegex = (str: string) => str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
const safeQuery = escapeRegex(query);
filterQuery.$or = [
  { title: { $regex: safeQuery, $options: "i" } },
];
```

Or use MongoDB's `$text` search with a text index for better full-text search.

---

## Issue #34: Bug - ProfileForm Makes All Fields Required Including Optional Ones

**Priority:** Medium  
**Labels:** bug, ux, validation  
**Affected Files:** `src/lib/validations.ts`

### Description
The `ProfileSchema` requires all fields (portfolio, location, bio) to have minimum lengths, but these should be optional. A user shouldn't be forced to provide a portfolio URL or location to update their name.

### Current Behavior
```typescript
export const ProfileSchema = z.object({
  name: z.string().min(3, ...),
  username: z.string().min(3, ...),
  portfolio: z.string().url({ message: "Please provide valid URL" }), // Required
  location: z.string().min(3, ...), // Required
  bio: z.string().min(3, ...), // Required
});
```

### Expected Behavior
```typescript
export const ProfileSchema = z.object({
  name: z.string().min(3, ...),
  username: z.string().min(3, ...),
  portfolio: z.string().url(...).optional().or(z.literal("")),
  location: z.string().min(3, ...).optional().or(z.literal("")),
  bio: z.string().min(3, ...).optional().or(z.literal("")),
});
```

---

## Issue #35: Bug - EditDeleteAction Doesn't Verify Ownership Before Showing Actions

**Priority:** Medium  
**Labels:** bug, security, ui  
**Affected Files:** `src/components/user/EditDeleteAction.tsx`

### Description
The `EditDeleteAction` component performs edit/delete operations without client-side ownership verification. While server-side checks exist, the component's `handleEdit` navigates to the edit page for ANY question (regardless of whether the user owns it).

Additionally, the edit button navigates using `router.push(\`/questions/${itemId}/edit\`)` which means for Answers, it navigates to a question edit page with the answer ID — an incorrect URL.

### Current Behavior
When `type === "Answer"`, clicking edit would navigate to `/questions/${answerId}/edit` which doesn't make sense.

### Expected Behavior
- The edit button should only appear for questions (already handled by conditional rendering)
- Consider removing the edit navigation for answers or implementing answer editing

---

## Issue #36: Performance - N+1 Query Pattern in getRecommendedQuestions

**Priority:** Medium  
**Labels:** performance, database  
**Affected Files:** `src/lib/actions/question.action.ts`

### Description
The `getRecommendedQuestions` function performs multiple sequential queries that could be optimized:
1. Fetches interactions (1 query)
2. Fetches interacted questions to get tags (1 query)
3. Counts recommended questions (1 query)
4. Fetches recommended questions (1 query)

### Expected Behavior
Consider using MongoDB aggregation pipeline to combine steps 1 and 2 into a single query with `$lookup`.

---

## Issue #37: Bug - Unused Import of Navbar in Root Layout

**Priority:** Low  
**Labels:** code-quality, cleanup  
**Affected Files:** `src/app/layout.tsx`

### Description
The root layout imports `Navbar` but doesn't use it (the Navbar is rendered in the `(root)/layout.tsx` instead). There's also a commented-out import.

### Current Behavior
```typescript
import Navbar from "@/components/navigation/navbar";
// import { async } from "./../node_modules/@auth/core/jwt";
```

### Expected Behavior
Remove unused imports.

---

## Issue #38: Missing Feature - No Rate Limiting on Authentication Endpoints

**Priority:** High  
**Labels:** security, authentication  
**Affected Files:** `src/lib/actions/auth.action.ts`, `src/app/api/auth/signin-with-oauth/route.ts`

### Description
Authentication endpoints have no rate limiting, making them vulnerable to brute force attacks. An attacker could try unlimited password combinations.

### Expected Behavior
Implement rate limiting:
- Limit login attempts per IP/email to 5 per 15 minutes
- Add exponential backoff or account lockout
- Consider using a rate limiting library like `upstash/ratelimit`

---

## Issue #39: Bug - createQuestion Uses `after()` But Interaction May Fail Silently

**Priority:** Low  
**Labels:** bug, reliability  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/lib/actions/answer.action.ts`, `src/lib/actions/vote.action.ts`

### Description
The `after()` callback from `next/server` is used for logging interactions, but any errors in this callback will be silently swallowed. If the interaction creation fails (e.g., due to authorization), the error is never reported.

### Current Behavior
```typescript
after(async () => {
  await createInteraction({ ... }); // If this fails, no one knows
});
```

### Expected Behavior
Add error logging in the `after()` callback:
```typescript
after(async () => {
  try {
    await createInteraction({ ... });
  } catch (error) {
    logger.error("Failed to create interaction", error);
  }
});
```

---

## Issue #40: Missing Feature - No Pagination Information Display

**Priority:** Low  
**Labels:** ux, enhancement  
**Affected Files:** `src/components/Pagination.tsx`

### Description
The pagination component shows the current page number but doesn't display total pages or total results. Users have no idea how many pages of results exist.

### Expected Behavior
Display "Page X of Y" or "Showing X-Y of Z results" to provide context.

---

## Issue #41: Bug - EMPTY_USERS Constant Has Typo "More uses are coming soon!"

**Priority:** Low  
**Labels:** bug, typo, ui  
**Affected Files:** `src/constants/states.ts`

### Description
The EMPTY_USERS message has a typo: "More uses are coming soon!" should be "More users are coming soon!"

### Current Behavior
```typescript
export const EMPTY_USERS = {
  title: "No Users Found",
  message: "You're ALONE. The only one here. More uses are coming soon!",
};
```

### Expected Behavior
```typescript
export const EMPTY_USERS = {
  title: "No Users Found",
  message: "You're ALONE. The only one here. More users are coming soon!",
};
```

---

## Issue #42: Accessibility - Missing Keyboard Navigation for Vote and Save Interactions

**Priority:** Medium  
**Labels:** accessibility, a11y  
**Affected Files:** `src/components/votes/Votes.tsx`, `src/components/questions/SaveQuestion.tsx`

### Description
Vote buttons and save question button use `Image` with `onClick` handlers but are not focusable via keyboard (not `<button>` or `role="button"` with `tabIndex`). Users who navigate with keyboard cannot vote or save questions.

### Expected Behavior
Use proper semantic HTML buttons wrapping the images, or add `role="button"`, `tabIndex={0}`, and `onKeyDown` handlers.

---

## Issue #43: Missing Feature - No SEO Metadata for Most Pages

**Priority:** Medium  
**Labels:** seo, enhancement  
**Affected Files:** `src/app/(root)/tags/page.tsx`, `src/app/(root)/community/page.tsx`, `src/app/(root)/collection/page.tsx`, `src/app/(root)/jobs/page.tsx`, `src/app/(root)/profile/[id]/page.tsx`

### Description
Only the home page and question detail pages have proper metadata exports. All other pages lack SEO-specific titles, descriptions, and Open Graph data.

### Expected Behavior
Add `export const metadata` or `generateMetadata` to each page for proper SEO:
```typescript
export const metadata: Metadata = {
  title: "Tags | DevOverflow",
  description: "Browse all tags on DevOverflow...",
};
```

---

## Issue #44: Bug - deleteAnswer Doesn't Use Transaction

**Priority:** Medium  
**Labels:** bug, data-integrity  
**Affected Files:** `src/lib/actions/answer.action.ts`

### Description
The `deleteAnswer` function performs multiple database operations (update question count, delete votes, delete answer) without using a MongoDB transaction. If one operation fails, the data becomes inconsistent.

### Current Behavior
```typescript
await Question.findByIdAndUpdate(answer.question, { $inc: { answers: -1 } });
await Vote.deleteMany({ actionId: answerId, actionType: "answer" });
await Answer.findByIdAndDelete(answerId);
// If the last operation fails, votes are deleted but answer remains
```

### Expected Behavior
Wrap all operations in a transaction (like `deleteQuestion` already does):
```typescript
const session = await mongoose.startSession();
session.startTransaction();
try {
  // ... all operations with session
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

---

## Issue #45: Missing Feature - No Password Reset / Forgot Password Flow

**Priority:** Medium  
**Labels:** feature, authentication, ux  
**Affected Files:** `src/app/(auth)/sign-in/page.tsx`, `src/lib/actions/auth.action.ts`

### Description
There is no "Forgot Password" or password reset functionality. Users who forget their credentials have no way to recover access.

### Expected Behavior
Implement a password reset flow:
1. "Forgot Password?" link on sign-in page
2. Email-based password reset token
3. Reset password form
4. Token validation and password update



---

## Issue #46: Missing Feature - No Email Verification for Credentials Sign-Up

**Priority:** Medium  
**Labels:** feature, security, authentication  
**Affected Files:** `src/lib/actions/auth.action.ts`

### Description
Users who sign up with credentials (email/password) are immediately logged in without any email verification. This allows fake email addresses to be used.

### Expected Behavior
Implement email verification:
1. Send verification email on registration
2. Mark account as unverified until email is confirmed
3. Restrict certain actions until email is verified

---

## Issue #47: Performance - Question Content Used in Text Search Without Text Index

**Priority:** Medium  
**Labels:** performance, database  
**Affected Files:** `src/lib/actions/question.action.ts`

### Description
The `getQuestions` function uses `$regex` on `content` field for searching, which is extremely slow on large collections. MongoDB `$regex` without an anchor (`^`) cannot use indexes.

### Expected Behavior
Create a MongoDB text index on `title` and `content` fields and use `$text` search:
```typescript
QuestionSchema.index({ title: "text", content: "text" });
// Then use:
filterQuery.$text = { $search: query };
```

---

## Issue #48: Code Quality - Inconsistent Error Handling Patterns

**Priority:** Low  
**Labels:** code-quality, consistency  
**Affected Files:** Multiple action files

### Description
Error handling is inconsistent across the codebase:
- Some functions return `handleError(error) as ErrorResponse` 
- Some throw errors to be caught by error boundaries
- `job.action.ts` returns `undefined` on error
- `SocialAuthForm.tsx` catches but uses `console.log` for errors

### Expected Behavior
Standardize error handling across all actions and components. All server actions should follow the same `ActionResponse<T>` pattern consistently.

---

## Issue #49: Bug - BadgeCounts Type Referenced in Stats Component But Not Defined

**Priority:** Low  
**Labels:** bug, typescript  
**Affected Files:** `src/components/user/Stats.tsx`, `src/types/global.d.ts`

### Description
The `Stats` component uses `BadgeCounts` as a prop type, but the global type definitions only define `Badges`. These may be the same type but with different names, causing confusion.

### Current Behavior
```typescript
// Stats.tsx
interface Props {
  badges: BadgeCounts; // This type may not be properly defined
}

// global.d.ts  
interface Badges {
  GOLD: number;
  SILVER: number;
  BRONZE: number;
}
```

### Expected Behavior
Use consistent type naming. Either rename to `Badges` everywhere or add a proper `BadgeCounts` type alias.

---

## Issue #50: Missing Feature - No User Notification System

**Priority:** Low  
**Labels:** feature, enhancement  
**Affected Files:** N/A (new feature)

### Description
Users have no way to be notified when:
- Someone answers their question
- Their answer is upvoted/downvoted
- Their question receives a new comment
- They earn a badge

### Expected Behavior
Implement a basic notification system with at minimum in-app notifications for key user interactions.

---

## Issue #51: UI/UX - GlobalSearch Hidden on Mobile Devices

**Priority:** Medium  
**Labels:** ux, mobile, responsiveness  
**Affected Files:** `src/components/search/GlobalSearch.tsx`

### Description
The global search component has `max-lg:hidden` class, making it completely invisible on mobile and tablet devices. Users on smaller screens have no access to global search functionality.

### Current Behavior
```tsx
<div className="relative w-full max-w-[600px] max-lg:hidden">
```

### Expected Behavior
Provide a mobile-friendly global search alternative:
- Add a search icon in the mobile navbar that opens a full-screen search overlay
- Or integrate global search into the mobile navigation sheet

---

## Issue #52: Code Quality - Large monolithic action files need splitting

**Priority:** Low  
**Labels:** code-quality, maintainability  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/lib/actions/user.action.ts`

### Description
Some action files are very large (300+ lines) with multiple responsibilities. `question.action.ts` handles CRUD, view counting, recommendations, and hot questions. `user.action.ts` handles user CRUD, stats, questions, answers, and tags.

### Expected Behavior
Consider splitting into focused modules:
- `question.action.ts` → `question-crud.action.ts`, `question-queries.action.ts`
- Or at minimum, add clear section comments and consistent ordering

---

## Issue #53: Bug - Stale Closure in Votes Component Vote Success Message

**Priority:** Low  
**Labels:** bug, ux  
**Affected Files:** `src/components/votes/Votes.tsx`

### Description
The vote success message uses the current `hasUpvoted`/`hasDownvoted` state from the promise result (which reflects the state BEFORE the vote action). After a successful vote, the state shown to the user refers to the old state, potentially showing incorrect messaging since the component doesn't re-read the promise.

### Current Behavior
The component uses `use(hasVotedPromise)` once and never re-fetches, so after voting, the UI state may not reflect the actual server state until the page is revalidated.

### Expected Behavior
Use optimistic updates or re-fetch the vote state after a successful vote action.

---

## Issue #54: Missing Feature - No Content Moderation or Spam Prevention

**Priority:** Medium  
**Labels:** feature, security, content-moderation  
**Affected Files:** `src/lib/actions/question.action.ts`, `src/lib/actions/answer.action.ts`

### Description
There are no content moderation features:
- No spam detection
- No profanity filter
- No duplicate question detection
- No rate limiting on question/answer posting
- No flagging/reporting mechanism for inappropriate content

### Expected Behavior
At minimum, implement:
1. Rate limiting on content creation (e.g., max 5 questions per hour)
2. Basic duplicate detection before posting
3. A report/flag mechanism for users to report inappropriate content

---

## Issue #55: Bug - Question Delete Doesn't Remove Associated Interactions

**Priority:** Low  
**Labels:** bug, data-integrity  
**Affected Files:** `src/lib/actions/question.action.ts`

### Description
When a question is deleted, the `deleteQuestion` function removes answers, votes, collections, and tag-questions, but doesn't delete associated Interaction records. This leaves orphaned interaction data in the database.

### Expected Behavior
Add interaction cleanup in the delete transaction:
```typescript
await Interaction.deleteMany({
  actionId: questionId,
  actionType: "question"
}).session(session);

// Also delete interactions for the question's answers
if (answers.length > 0) {
  await Interaction.deleteMany({
    actionId: { $in: answers.map(a => a._id) },
    actionType: "answer"
  }).session(session);
}
```

---

## Issue #56: Missing Feature - No Dark Mode Scrollbar Styling

**Priority:** Low  
**Labels:** ui, dark-mode  
**Affected Files:** `src/app/globals.css`

### Description
The custom scrollbar styling only defines a white background track. In dark mode, this creates a jarring white scrollbar against the dark background.

### Current Behavior
```css
.custom-scrollbar::-webkit-scrollbar-track {
  background: #ffffff;
}
```

### Expected Behavior
Add dark mode scrollbar styles:
```css
.dark .custom-scrollbar::-webkit-scrollbar-track {
  background: #151821;
}
```

---

## Issue #57: Code Quality - Unused Dependencies in package.json

**Priority:** Low  
**Labels:** code-quality, cleanup  
**Affected Files:** `package.json`

### Description
Several dependencies appear to be unused or redundant:
- `date-fns` - imported but `dayjs` is also present and used in profile page
- `recharts` - no chart components found in the codebase
- `react-resizable-panels` - no resizable panel usage found
- `embla-carousel-react` - no carousel usage found
- `react-day-picker` - no date picker usage found
- `input-otp` - no OTP input found
- `cmdk` - no command palette found
- Many Radix UI primitives installed but not visibly used (hover-card, menubar, navigation-menu, etc.)

### Expected Behavior
Audit all dependencies and remove unused ones to reduce bundle size and maintenance burden.

---

## Issue #58: Missing Feature - No CSRF Protection on State-Changing Operations

**Priority:** Medium  
**Labels:** security  
**Affected Files:** `src/app/api/` (all POST/PUT/DELETE routes)

### Description
API routes that perform state-changing operations don't implement CSRF protection. While NextAuth provides CSRF for its own endpoints, custom API routes are unprotected.

### Expected Behavior
Since the app uses Server Actions (which Next.js protects automatically), this primarily affects the REST API routes. Consider:
1. Adding CSRF token validation to custom API routes
2. Or migrating all mutations to Server Actions exclusively

---

## Issue #59: Bug - Profile Page Crashes If getUserStats, getUserQuestions, or getUserAnswers Fails

**Priority:** Medium  
**Labels:** bug, runtime-error  
**Affected Files:** `src/app/(root)/profile/[id]/page.tsx`

### Description
The profile page destructures results from multiple async calls without checking for success:

```typescript
const { questions, isNext: hasMoreQuestions } = userQuestions!; // Crashes if null
const { answers, isNext: hasMoreAnswers } = userAnswers!; // Crashes if null
const { tags } = userTopTags!; // Crashes if null
```

If any of these calls fail, the page will crash with an unhandled error.

### Expected Behavior
Add null checks and fallback values:
```typescript
const { questions = [], isNext: hasMoreQuestions = false } = userQuestions || {};
const { answers = [], isNext: hasMoreAnswers = false } = userAnswers || {};
const { tags = [] } = userTopTags || {};
```

---

## Issue #60: Missing Feature - No Sorting for User Questions and Answers on Profile

**Priority:** Low  
**Labels:** feature, ux  
**Affected Files:** `src/app/(root)/profile/[id]/page.tsx`, `src/lib/actions/user.action.ts`

### Description
The profile page shows user questions and answers but provides no sorting options (newest, oldest, most voted). The data is always returned without explicit sort criteria in `getUserQuestions` and `getUserAnswers`.

### Expected Behavior
Add sort criteria to profile queries and provide UI filters on the profile tabs.
