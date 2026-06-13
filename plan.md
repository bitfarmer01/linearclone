Ready for review
Select text to add comments on the plan
Linear Clone — B2B Multi-Tenant SaaS — Design & Phased Build Plan
Context
We're building a Linear-style team issue tracker as a B2B multi-tenant SaaS. Each Clerk Organization = one workspace (org-required onboarding). The differentiator is an AI agent gated to the Pro plan, plus an Enterprise tier with unlimited seats. Clerk handles both authentication and billing; Convex is the realtime data/compute layer; Next.js 16 is the app shell; shadcn/ui is the component layer.

The goal of this plan is a phased build where each slice ships and is verifiable independently, with as much of the Clerk/Convex setup automated through their CLIs (acting on the user's behalf) as the tools allow.

Stack ground truth (already verified in this repo)
Next.js 16.2.9 / React 19.2 / Turbopack default / pnpm. Middleware is renamed → proxy.ts with export default clerkMiddleware() (already present). params/searchParams/cookies()/ headers() are async (must await). Config in next.config.ts.
Convex 1.41.0 installed; convex/ dir not yet created. Official Convex plugin enabled project-scoped in .claude/settings.json — its conventions are authoritative (object-form functions with args+returns validators; index name lists columns in order, never _creationTime; an index for every read path, never .filter() for WHERE; internal* for private; external IO only in actions; assertOrgMember at every function boundary).
Clerk @clerk/nextjs 7.5.2 installed. B2B billing (plans on organizations), per-seat pricing, and "unlimited members" plans all confirmed available. Gate features with has({ feature }) / <Protect> / <PricingTable for="organization" />.
shadcn/ui + Tailwind v4 (already installed). To be init-ed.
CLIs: Convex CLI present but not authenticated; Clerk CLI not installed. Both require a one-time, browser-based login (user approves) — scheduled as Phase 0 step 1.
Decisions locked
Area	Decision
Scope	Full Linear core: issues (status/priority/assignee/labels), Kanban + list, projects w/ progress, cycles/sprints, comments, per-issue activity feed, filtering + saved views, full-text search, triage inbox, Cmd-K + keyboard shortcuts
Workspace model	Org = workspace (no Teams layer). Issues grouped by Projects. IDs project-prefixed + sequential (WEB-123)
UI	shadcn/ui on Tailwind v4; @dnd-kit for Kanban DnD; cmdk for the command palette
AI agent (Pro)	Configurable LLM provider via env (Anthropic/OpenAI through the Vercel AI SDK seam). Does NL issue creation, auto-triage/enrich, summaries/standups, and a conversational copilot
Copilot	Use @convex-dev/agent for the conversational copilot (threads/tools/streaming); one-shot AI features use plain provider calls
Plans	Free (≤3 seats, no AI) · Pro (per-seat, AI on, higher limits) · Enterprise (unlimited seats; SSO/audit-log/custom-roles shown gated but stubbed)
Roles / onboarding	Admin/Member (Clerk defaults) · org-required onboarding
Plan enforcement	Defense in depth: UI via Clerk has()/<Protect>; authoritative via Clerk billing webhook → Convex organizations sync, checked in every gated function
Semantic/vector search	Out of scope (full-text via Convex search indexes covers "full-text search"). Noted as future
Architecture
Three planes: Clerk (auth, orgs, roles, seats, billing) · Convex (schema, reactive queries, mutations, actions for LLM+webhooks, crons, search) · Next 16 App Router (Server Components for shell/auth/billing; Client Components for everything calling useQuery/useMutation/useAction).

Auth/identity flow: Browser → Clerk session (cookie); proxy.ts clerkMiddleware() on every request → app wrapped in <ClerkProvider> → <ConvexProviderWithClerk client useAuth> (convex/react-clerk) → convex/auth.config.ts declares { domain: CLERK_JWT_ISSUER_DOMAIN, applicationID: "convex" } (needs a Clerk JWT template named convex incl. the org claim) → Convex functions call ctx.auth.getUserIdentity(). Functions also take orgId as an explicit arg and verify membership (keeps functions testable/unambiguous).

Realtime: no sockets to build — useQuery subscriptions re-render on any mutation. DnD = optimistic local reorder + one mutation; Convex reconciles.

Data model (convex/schema.ts)
Every shared table carries orgId + an index; search via a denormalized searchBlob (title+description) for a single search index.

organizations:  { clerkOrgId, name, slug?, plan, features[], seatLimit, imageUrl?, updatedAt }
                .index("by_clerkOrgId", ["clerkOrgId"])               // synced from Clerk webhook
users:          { clerkUserId, orgId, name, email, imageUrl?, role }  // mirror of Clerk memberships
                .index("by_clerkUserId",["clerkUserId"]).index("by_org",["orgId"]).index("by_org_clerkUser",["orgId","clerkUserId"])
projects:       { orgId, name, key, description?, leadUserId?, status, color? }
                .index("by_org",["orgId"]).index("by_org_key",["orgId","key"]).index("by_org_status",["orgId","status"])
projectCounters:{ projectId, lastNumber }  .index("by_project",["projectId"])   // atomic numbering
cycles:         { orgId, projectId?, name, number, startDate, endDate, status }
                .index("by_org",["orgId"]).index("by_org_status",["orgId","status"]).index("by_project",["projectId"])
labels:         { orgId, name, color }  .index("by_org",["orgId"]).index("by_org_name",["orgId","name"])
issues:         { orgId, projectId, number, identifier, title, description?, searchBlob, status,
                  priority, assigneeId?, creatorId, cycleId?, labelIds[], sourceType?, rank, completedAt? }
                .index("by_org",["orgId"]).index("by_org_status",["orgId","status"]).index("by_project",["projectId"])
                .index("by_project_status",["projectId","status"]).index("by_project_number",["projectId","number"])
                .index("by_org_assignee",["orgId","assigneeId"]).index("by_cycle",["cycleId"])
                .index("by_org_status_rank",["orgId","status","rank"])
                .searchIndex("search", { searchField:"searchBlob", filterFields:["orgId","projectId","status","priority","assigneeId"] })
comments:       { orgId, issueId, authorId, body }  .index("by_issue",["issueId"]).index("by_org",["orgId"])
activity:       { orgId, issueId, actorId?, type, from?, to?, meta? }  .index("by_issue",["issueId"])
savedViews:     { orgId, ownerId, name, scope, layout, groupBy?, filters{...} }
                .index("by_org",["orgId"]).index("by_org_owner",["orgId","ownerId"])
Status union includes "triage" so the triage inbox is just by_org_status where status="triage" (accept → backlog/todo + triage_accepted activity; decline → canceled + triage_declined). AI/API/email entry points create issues in status:"triage" with sourceType. The copilot's threads/messages live in the @convex-dev/agent component, not this schema.

Key mechanisms
Atomic issue numbers: single projectCounters doc per project; read-modify-write inside the issues.create mutation. Convex mutations are serializable (OCC) → concurrent creates retry, never duplicate. Counter initialized with the project (never derive from row count).
Plan-gating + seats: Clerk billing webhook → Convex httpAction in convex/http.ts (svix-verified) → internal.organizations.syncFromClerk upserts {plan, features, seatLimit} keyed by by_clerkOrgId; organizationMembership.* → internal.users.syncMembership. Helpers in convex/lib/auth.ts: assertOrgMember, assertOrgAdmin, assertPlan, assertFeature(ctx,orgId,"ai_agent"), assertSeatAvailable. Webhook URL: https://<deployment>.convex.site/clerk-webhook, secret CLERK_WEBHOOK_SECRET. Over-limit memberships are accepted + flagged (banner), not hard-rejected.
AI provider abstraction: convex/ai/provider.ts returns an AI-SDK languageModel() chosen by LLM_PROVIDER env (@ai-sdk/anthropic | @ai-sdk/openai). All LLM calls run in Convex actions; structured results persisted via ctx.runMutation(internal...). One-shot capabilities use generateObject/generateText (Zod schemas). The copilot uses @convex-dev/agent (convex.config.ts app.use(agent)) with Convex tools (searchIssues/createIssue/updateIssue/summarize), each internally assertOrgMember/assertFeature-gated so it can't touch another org.
Search: denormalized searchBlob kept in sync on issue writes → one Convex searchIndex with filterFields. Powers Cmd-K search and the filter bar.
Board ordering: rank string (LexoRank midpoint) avoids reindexing on reorder; add a rebalanceRanks mutation for occasional key collisions.
Keyboard: one window keydown hook in the app shell (guards inputs), Linear-style chords (c, g b, g l, g p, g i, /, etc.); ⌘K opens the cmdk palette.
App route structure (App Router)
app/layout.tsx (server)            ClerkProvider + ConvexProviderWithClerk wrapper, fonts, <Toaster/>
app/page.tsx (server)              marketing landing
app/(marketing)/pricing/page.tsx   <PricingTable for="organization"/>
app/(auth)/sign-in|sign-up/...     Clerk <SignIn/> <SignUp/>
app/onboarding/page.tsx (server)   no org → <CreateOrganization/>; else → /app   (org-required gate)
app/app/layout.tsx (server)        AppShell: sidebar + <OrganizationSwitcher/> + Cmd-K island; require orgId
app/app/inbox/page.tsx             triage inbox (status="triage")
app/app/issues/page.tsx            all issues (board/list toggle, filters)
app/app/issues/[identifier]/...    await params; issue detail + activity + comments
app/app/projects/[projectId]/...   await params; project board/list + progress
app/app/projects/[projectId]/cycles/[cycleId]/...  await both params; cycle board + progress
app/app/views/[viewId]/page.tsx    saved view; await params
app/app/copilot                    AI chat sidebar (Sheet/route)
app/app/settings/{general,members,billing,labels}   members=<OrganizationProfile/>, billing=<PricingTable/>
Webhook lives in convex/http.ts (not a Next route). Async params/searchParams everywhere.

CLI-driven setup (act on user's behalf where possible)
Convex CLI: convex dev (create dev deployment, device login), convex env set (issuer domain, webhook secret, LLM keys), convex deploy, convex run (seed/verify), convex data (inspect).
Clerk CLI: pnpm add -g clerk (or npx clerk) → clerk auth login → clerk init --mode agent -y (SDK scaffold) → clerk config patch (sign-in methods, redirects, enable Organizations) → clerk api to create the convex JWT template + inspect orgs → clerk env pull (keys into .env).
Dashboard-only (guided click-path): create the three billing plans (Free/Pro/Enterprise) and the ai_agent feature, seat caps, and register the webhook endpoint if not API-creatable.
Phased roadmap (each phase independently shippable + verifiable)
Phase 0 — Foundation. Executed as ordered steps:
Step 1 — Installs (do this first). One install pass, no wiring yet:
Project deps: pnpm add convex @convex-dev/agent ai @ai-sdk/anthropic @ai-sdk/openai @dnd-kit/core @dnd-kit/sortable @dnd-kit/modifiers cmdk svix zod (@clerk/nextjs already present).
Clerk CLI: pnpm add -g clerk (or use npx clerk); Convex CLI already installed.
shadcn: pnpm dlx shadcn@latest init then pnpm dlx shadcn@latest add button input textarea label dialog sheet dropdown-menu select command badge avatar tooltip tabs popover checkbox separator scroll-area skeleton sonner context-menu calendar progress card (expect a React-19 --legacy-peer-deps/--force prompt under pnpm).
Verify Step 1: pnpm ls shows the new deps; clerk --version and convex --version both work; components.json + lib/utils.ts exist.
Step 2 — Authenticate CLIs (you approve browser logins). clerk auth login; npx convex dev (device login + creates the dev deployment). clerk whoami / Convex dashboard confirm access.
Step 3 — Wire auth. clerk init --mode agent -y; clerk config patch (enable Organizations, sign-in methods, redirects); clerk api to create the convex JWT template (incl. org claim); convex/auth.config.ts w/ CLERK_JWT_ISSUER_DOMAIN; clerk env pull / convex env set for keys.
Step 4 — App shell. app/providers.tsx (ClerkProvider → ConvexProviderWithClerk); sign-in/up routes; onboarding w/ <CreateOrganization>; app/ shell w/ <OrganizationSwitcher>; org-required gate in proxy.ts; organizations/users schema + me query.
Verify Phase 0: sign up → forced create-org → /app; useQuery(api.users.me) returns identity; switching org re-scopes.
Phase 1 — Issues + Board + List. projects/issues/labels/comments/activity/projectCounters; atomic numbering; create/update/updatePosition; Kanban (dnd-kit) + list; issue detail w/ activity + comments. Verify: WEB-1/WEB-2 (no dup under concurrent create); drag persists status + updates a 2nd browser instantly; comment shows in activity feed.
Phase 2 — Projects, Cycles, Progress. cycles CRUD + date pickers; assign issues to cycle/project; project & cycle progress bars. Verify: progress bar moves as issues complete; cycle shows its set + %; reassigning updates reactively.
Phase 3 — Filtering, Saved Views, Search. savedViews CRUD; filter bar driving async searchParams
indexed queries; issues.search on the search index; grouping. Verify: filters reflected in URL + restored on reload; save→reopen restores; palette search matches title/description.
Phase 4 — Triage Inbox. triage status modeling; inbox route; accept/decline + activity; sourceType provenance; quick-add to triage. Verify: triage issue appears only in inbox; accept → board + logged; decline → canceled.
Phase 5 — Billing + Plan-Gating. Clerk plans/features; <PricingTable for="organization"/>; convex/http.ts webhook (svix) → sync mutations; assertPlan/assertFeature/assertSeatAvailable; Free cap = 3; Enterprise stubs gated. Verify: new org = Free (seatLimit 3 in Convex after webhook); 4th invite blocked/flagged; upgrade → webhook flips plan + adds ai_agent (confirm in Convex dashboard); Enterprise settings render as locked stubs.
Phase 6 — AI one-shot (Pro-gated). convex/ai/provider.ts; actions ai.createIssues, ai.enrichTriage, ai.summarize; assertFeature first; <Protect feature="ai_agent"> + upsell. Verify: Free → AI hidden and direct action call throws (server gate); Pro → NL→structured triage issue; enrich fills a triage item; summarize returns a standup; flip LLM_PROVIDER → still works.
Phase 7 — AI Copilot. @convex-dev/agent wired; Agent w/ provider model + gated Convex tools; saveMessage + scheduled internalAction agent.generateText; streaming copilot Sheet. Verify: Pro asks "urgent unassigned WEB issues" → tool lists them; "create triage issue: …" appears; messages stream; Free has no copilot.
Polish (folded across phases): shortcuts/palette finalize, optimistic DnD ranks, skeletons, empty states, dark mode.
Risks / open items (resolved as recommendations)
AI SDK seam: use AI-SDK adapters (@ai-sdk/anthropic/@ai-sdk/openai) — gives env-var provider swap and plugs into @convex-dev/agent. (Raw @anthropic-ai/sdk would be more glue, no agent compatibility.)
Embeddings: Anthropic has none; only relevant if semantic search were in scope — it isn't.
Clerk billing event names + whether webhook endpoints / JWT templates are API-creatable: confirm via clerk api ls at Phase 0/5; fall back to dashboard for anything not exposed.
JWT convex template must include the org claim, else functions rely on the orgId arg + backend membership verification.
shadcn under React 19 + pnpm: expect a --legacy-peer-deps/--force prompt on first add; possible pnpm.overrides pin for a stray Radix peer.
DnD rank rebalance must be implemented so it isn't forgotten.
Overall verification
After each phase, run pnpm lint + tsc --noEmit (Convex plugin also lints/typechecks on hooks), exercise the phase's Verify steps in two browser sessions (to prove realtime + multi-tenant isolation), and confirm cross-org data cannot be read (negative test against assertOrgMember). Billing/AI phases additionally verified by flipping plan in Clerk and confirming the Convex organizations row + gate behavior.