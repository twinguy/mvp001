After evaluating the functional requirements for the application—encompassing user authentication, facility/unit inventory, customer management, rental agreements, payments/billing, maintenance operations, reporting, and general system needs—the optimal platform should prioritize backend-as-a-service (BaaS) capabilities to abstract infrastructure complexities like databases, authentication, storage, and serverless functions. This aligns with the emphasis on developer friendliness for those less experienced in infra setup, enabling fast prototyping through pre-built SDKs, dashboards, and minimal configuration. The platform must also support web app building/testing/deployment/scaling, with mobile app integration (e.g., via SDKs for native or cross-platform frameworks like Flutter or React Native), even if mobile deployment occurs through app stores rather than the platform itself.

Key evaluation criteria based on your priorities:
- **Single vs. Separate Platforms**: Preference for all-in-one, but hybrid acceptable if it optimizes cost and ease.
- **Pricing**: Focus on affordable tiers with generous free limits for prototyping, low base costs for production ($25–$50/mo ideal), and usage-based scaling to avoid surprises; avoid platforms that escalate quickly at scale.
- **Fast Prototyping**: Quick setup (e.g., instant DB/auth provisioning), SDKs for rapid integration, and tools like real-time features or UI helpers.
- **Developer Friendliness**: Abstracts infra (e.g., managed DB, auth flows), supports modern stacks (JS/React/Next.js for web, Flutter for mobile), open-source flexibility, and intuitive dashboards/docs.
- **Build/Test/Deploy/Scale**: Full lifecycle support, including CI/CD, auto-scaling, and mobile SDKs/testing.
- **Mobile Capabilities**: SDKs for iOS/Android integration; bonus for testing/distribution tools.
- Other: Compliance (e.g., SOC2), real-time updates (useful for maintenance notifications), and integrations (e.g., payments like Stripe).

I assessed leading options (Firebase, AWS Amplify, Supabase, Vercel, and others like Appwrite/Nhost) using current data from official sources and comparisons as of November 2025. Platforms like Heroku or Render were deprioritized due to less comprehensive BaaS features or higher complexity for infra-novice devs. Low-code/no-code tools (e.g., Bubble) were excluded as the query implies developer-oriented platforms with coding flexibility.

### Recommended Platform: Supabase (with Vercel for Frontend Hosting if Needed)
Supabase stands out as the best overall fit, offering a near-single-platform experience for the core app while being exceptionally cost-effective and developer-friendly. It's an open-source BaaS alternative to Firebase, built on PostgreSQL for relational data—which is ideal for your app's structured needs (e.g., units, customers, agreements, ledgers) versus NoSQL's flexibility in competitors. It abstracts infra completely, allowing devs to focus on app logic via SDKs and a intuitive dashboard, enabling prototypes in hours (e.g., spin up auth/DB in minutes). For a pure single platform, it handles most needs, but pairing with Vercel for web frontend deployment creates a seamless hybrid that's still simple and preferred over fully separate setups.

#### Why Supabase?
- **Alignment with Requirements**:
  - **Build/Test**: Provides SDKs for web (JS, React, Next.js, Vue) and mobile (Flutter, Swift, Kotlin), with built-in tools like real-time subscriptions (for live updates on maintenance or occupancy) and auth hooks. Testing via audit logs, metrics, and branching (for dev/staging environments). Supports your modules: Auth for user management, Postgres for inventory/customers/agreements (with row-level security for compliance), Storage for documents/photos, Edge Functions for custom logic (e.g., payment webhooks, delinquency automation).
  - **Deploy/Scale**: Deploys backends instantly via dashboard; scales usage-based (e.g., auto-compute from Micro $10/mo to 64 cores/256 GB RAM). No vendor lock-in due to open-source—self-host for ultimate scale. Handles high traffic with dedicated instances; add-ons like point-in-time recovery ($100/mo) for reliability.
  - **Mobile Capabilities**: Full SDK support for building/deploying mobile apps (e.g., tenant portal on iOS/Android). Integrate with frameworks like Flutter for cross-platform; deploy via Apple/Google stores. No built-in mobile testing like Firebase's Test Lab, but pairs well with external tools (e.g., Appium).
  - **Fast Prototyping**: Free tier for quick starts (unlimited API requests, instant Postgres setup); devs can prototype auth/DB/integration in under an hour without infra config. Open-source encourages community extensions.
  - **Developer Friendliness**: Abstracts DB (relational queries via SQL or JS SDK), auth (MFA, OAuth, SSO), storage, and functions. Dashboard for visual management; unlimited team members with roles. Steeper for NoSQL users but friendlier for structured data apps like yours.
  - **Additional Long-Term Value**: Supports your advanced features like AI integrations (vectors for recommendations), CRM/email automations, and compliance (SOC2 in Team tier). Open-source allows custom tweaks.

- **Pricing**:
  - **Free**: $0/mo—500 MB DB, 1 GB storage, 50k MAUs, 500k function invocations. Ideal for prototyping; pauses inactive projects to save costs.
  - **Pro**: $25/mo base (+ usage, e.g., $0.125/GB extra DB, $0.00325/extra MAU)—includes 8 GB DB, 100 GB storage, 100k MAUs. Extremely affordable; reviews note it's 4-5x cheaper than Firebase/Amplify for equivalent SaaS usage (e.g., your MVP with 50 properties/units fits easily under limits).
  - **Team/Enterprise**: $599/mo+ for compliance/SLAs; custom for large-scale.
  - Overall: Usage-based avoids overpaying; spend caps prevent surprises. Best value for price-conscious scaling—far lower than Amplify's per-minute builds or Firebase's invocation fees.

- **Trade-offs**:
  - No native web hosting (recommends Vercel/Netlify for frontend deployment), but this hybrid is common and keeps things lightweight.
  - Less mobile-specific testing tools than Firebase, but sufficient via SDKs.

#### Optional Hybrid: Supabase + Vercel
If web deployment feels fragmented, add Vercel for frontend hosting—it's a natural pair (e.g., Next.js app with Supabase backend). Vercel handles build/deploy/scale for web, with zero-config CI/CD.
- **Vercel Pricing**: Hobby $0/mo (unlimited deploys); Pro $20/mo + usage (faster builds, no queues).
- **Why Add It?**: Provides the "deploy" piece Supabase lacks; total cost ~$45/mo for Pro tiers. Still developer-friendly (deploy in seconds, auto-scaling) and supports your dashboard/UI needs.
- This setup is acceptable under your "separate platforms OK" clause and optimizes for ease/cost over a rigid single platform.

#### Why Not Alternatives?
- **Firebase (Google)**: Strong runner-up for single-platform (includes hosting, excellent mobile testing/distribution). Free Spark tier for prototyping; Blaze pay-as-you-go (e.g., $0.40/million invocations). Great for real-time/mobile, but NoSQL Firestore less ideal for relational data; pricier at scale (e.g., data/egress fees add up faster than Supabase). Less open-source flexibility.
- **AWS Amplify**: Fully single-platform with AWS backing (auth via Cognito, DB via DynamoDB, hosting included). Free Tier for 12 months; then ~$0.01/build min, $0.15/GB transfer. Scalable/enterprise-ready, but steeper learning curve (more AWS config exposure) and higher costs (e.g., 2-3x Supabase for similar usage). DynamoDB (NoSQL) not as friendly for your structured queries.
- **Others**: Vercel alone lacks backend (DB/auth); Appwrite/Nhost similar to Supabase but less mature/ecosystem. Azure/Google Cloud too infra-heavy for your dev skill assumption.

Start with Supabase's Free tier for prototyping—sign up at supabase.com, import a sample schema for self-storage, and integrate Stripe for payments. For mobile, use their Flutter SDK docs. If scaling to thousands of units, upgrade to Pro and monitor usage via dashboard. This balances price, speed, and ease while meeting all day-1 and long-term requirements.