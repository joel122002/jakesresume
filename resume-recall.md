**Resume Bullet:**
> Reduced cache invalidation operations by 65x (30,400 → 476) in a .NET 8 microservice by implementing a debounce-based Memcached expiry strategy, eliminating inconsistent filter results caused by partial cache clears during nightly model popularity updates.

---

**Recall Guide:**

**The Problem:**
A nightly job ran during peak hours (client requirement) to update the popularity ranking of 400 car models. The existing code cleared all 76 filter cache keys after *every single model update event* — meaning 30,400 cache clear operations per job run. Since these keys were repopulated on the first user request after each clear, and the job ran during peak hours, the cache was being constantly cleared and repopulated mid-job. This meant users would see filters in a **half-updated, inconsistent state** — some models reordered, some not yet processed.

**Your Solution:**
Instead of immediately clearing cache keys on each event, you implemented a **debounce mechanism** in a .NET 8 microservice — each event would simply push the Memcached expiry time forward by 5 minutes from the current timestamp. This meant the cache stayed alive and consistent while events kept firing. The keys only truly expired once all 400 models had been processed (or if there was an unlikely >5 minute gap between events). After expiry, keys were repopulated request-driven — the first user to hit each filter page after expiry would take the DB hit and repopulate that specific key.

**The Stack:** .NET 8 microservice, Memcached.

**The Impact:**
- Cache invalidation operations: **30,400 → 476 (65x reduction)**
- Eliminated user-facing inconsistent filter ordering during nightly job runs

---

**Likely Interview Deep-Dives:**

**Q: How did you achieve the 65x reduction?**
> The old code cleared all 76 filter cache keys on every single model update event — 76 × 400 = 30,400 clears. The new approach debounces the expiry, so the keys only actually expire once at the end of the job — 400 model keys + 76 filter keys = 476 total. The number of *calls* to Memcached stays the same, but they're now lightweight expiry updates rather than cache clears.

**Q: How did you ensure cache keys were repopulated after the job?**
> Repopulation was request-driven — each of the 76 filter keys gets repopulated the first time a user hits that page after expiry. Since the job ran during peak hours, all 76 keys would get warm quickly through organic traffic. It wasn't an instant guaranteed repopulation, but the business accepted that tradeoff given peak hour traffic patterns.

**Q: Doesn't one user only repopulate 1 of the 76 keys?**
> Exactly — each filter page has its own cache key and only gets repopulated when a user visits that specific page. During peak hours, organic traffic across different filter pages repopulates all 76 keys quickly. It's a lazy repopulation strategy and the cold window per key is negligible at peak traffic.

**Q: Why not just delay the job to off-peak hours?**
> The job running at peak hours was a client requirement — it wasn't something we could change. The solution had to work within that constraint, which is exactly why controlling it at the cache layer was the right approach.

**Q: What if the 5 min debounce window isn't enough?**
> The nightly job consistently processes all 400 models well within that window. A >5 minute gap between two consecutive model update events has never been observed in production, so the debounce window is more than sufficient.

**Q: Why not a background repopulation job instead?**
> That would add infrastructure complexity — an additional scheduled job to monitor, maintain and potentially fail. Given peak hour traffic naturally repopulating the keys within seconds of expiry, the simpler request-driven approach was the right tradeoff.

---
---
---
---
---
---
---
---
---


**Resume Bullet:**
> Identified and eliminated 150GB/week of redundant CloudFront data transfer by proactively auditing CDN dashboards, discovering a duplicate apple-touch-icon that had been incorrectly imported for 2 years across all pages, saving ~$600 annually in infrastructure costs.

---

**Recall Guide:**

**The Problem:**
A previous icon change made 2 years ago had left a duplicate apple-touch-icon incorrectly imported in the `<head>` of every page. A correct import already existed and was functioning properly — this duplicate was never rendered or visible to users in any way. However, since it was referenced across all pages, every single page load was triggering a CloudFront fetch for this redundant file, silently accumulating 150GB of unnecessary data transfer every week across the platform's 80M monthly users.

**How You Found It:**
Nobody had noticed this for 2 years. You proactively went through CloudFront's dashboard, sorted files by download volume, and noticed this icon file near the top. You investigated what it did, realised it was a duplicate that served no purpose, and flagged it.

**The Fix:**
Removed the incorrect `<link>` tag from the HTML. One line change. The correct icon import was already in place and continued working as expected.

**The Stack:** CloudFront, HTML.

**The Impact:**
- **150GB/week** of redundant data transfer eliminated
- **~$600/year** saved in CloudFront costs (~$0.085/GB × 7.2TB/year)
- Unnoticed for **2 years** across a platform with **80M monthly users**

---

**Likely Interview Deep-Dives:**

**Q: How did you find this?**
> I was proactively going through CloudFront's dashboard and sorted assets by download volume. This file kept appearing near the top and when I investigated it I realised it was a duplicate icon that was never actually rendered anywhere on the platform.

**Q: Why had nobody noticed this for 2 years?**
> It was a silent cost — no user-facing bug, no errors, just unnecessary data transfer quietly accumulating. It only became visible when you specifically looked at CloudFront asset-level analytics, which isn't something people routinely audit.

**Q: $600/year isn't that much, why is this worth mentioning?**
> The dollar amount is modest, but the point is the mindset — proactively auditing infrastructure, identifying waste nobody had noticed for 2 years, and fixing it with a one line change. On a platform with 80M monthly users even small inefficiencies compound significantly over time.

**Q: Could this have impacted page load performance?**
> Marginally — every page load was triggering an unnecessary network request for a file that served no purpose. Removing it means one fewer request per page load across all 80M monthly users, which at scale is a meaningful reduction in unnecessary network overhead.

---
---
---
---
---
---
---
---
---

**Resume Bullet:**
> Reduced database calls by 93% in a .NET 8 microservice by restructuring Memcached strategy from 6.15M individual version-city keys to 4,000 version-level hashmaps containing all 1,539 city entries, reducing overall shared MySQL EC2 cluster load by ~28%.

---

**Recall Guide:**

**The Problem:**
The platform shows car variant prices to users based on their selected city. With 4,000 car variants and 1,539 cities, the existing caching strategy stored a **separate cache key for every version-city combination** — totalling 6,156,000 individual Memcached keys. Each key had a 3 hour expiry. Every time a key expired and a user requested that specific version-city combination, it triggered an individual DB call to repopulate just that one key. With 80M monthly users browsing variants across 1,539 cities, cache misses were extremely frequent, resulting in millions of unnecessary DB calls hammering a shared MySQL EC2 cluster. This microservice was responsible for approximately 30% of total calls to that cluster.

**Your Solution:**
Restructured the caching strategy to store **one cache key per variant**, containing a hashmap of all 1,539 cities and their respective prices for that variant. This reduced the total number of cache keys from 6,156,000 to just 4,000. Now when a cache miss occurs for any city under a variant, a single DB call fetches all 1,539 city prices for that variant and repopulates the entire hashmap at once. The 3 hour refresh interval remained the same. The implementation leveraged the company's internal **AEPLCore.Cache** library which handles cache stampede out of the box, ensuring only one DB call is made per key on expiry regardless of concurrent user requests.

**The Stack:** .NET 8 microservice, Memcached, MySQL on EC2, AEPLCore.Cache (internal library).

**The Impact:**
- DB calls from this microservice reduced by **93%** (measured via database-level logs and Grafana dashboards)
- Cache keys reduced from **6,156,000 → 4,000**
- This microservice accounted for ~30% of total cluster calls, so overall MySQL EC2 cluster load reduced by **~28%**

---

**Likely Interview Deep-Dives:**

**Q: How did you arrive at the 93% reduction?**
> We have database level logs and Grafana dashboards that track DB calls per microservice. The reduction was measured by comparing DB call volume before and after the change was deployed.

**Q: Why not store everything in one mega dictionary?**
> Memcached has a 1MB per key limit. With 4,000 versions and 1,539 cities, a single mega dictionary would be hundreds of MBs — far exceeding the limit. One key per version with all its cities sits comfortably within the 1MB limit at roughly 75KB per key.

**Q: Didn't you just shift the problem? Now a cache miss fetches 1,539 cities instead of 1.**
> Yes but that's a deliberate tradeoff — one slightly heavier DB call every 3 hours per version versus potentially thousands of individual DB calls per version per expiry cycle. The net DB load is dramatically lower.

**Q: How did you handle cache stampede with 4,000 keys expiring simultaneously?**
> We have an internal library called AEPLCore.Cache that our team developed which handles cache stampede out of the box. It ensures only one DB call is made per key on expiry regardless of concurrent requests hitting that key simultaneously.

**Q: Why did the 3 hour interval stay the same?**
> The refresh interval was a business requirement — city-level pricing data needed to be no more than 3 hours stale. That constraint wasn't ours to change, the optimisation was purely in how the data was structured in cache.

**Q: How did this impact the shared MySQL cluster?**
> Our microservice was responsible for around 30% of total calls to the cluster. By reducing our share by 93%, the overall cluster load dropped by roughly 28%, which meaningfully reduced pressure on a resource shared across multiple services.

**Q: Why not store everything in one mega dictionary?**
> Memcached has a 1MB per key limit. With 4,000 versions and 1,539 cities, a single mega dictionary would be hundreds of MBs — far exceeding that limit. One key per version with all 1,539 cities sits comfortably within the limit at roughly 75KB per key.

**Q: What if a variant has no demand in most cities?**
> That's actually an argument in favour of this approach — in the old system, unpopular version-city combinations would still occasionally get requested, miss cache, and hit the DB. Now all cities are loaded together regardless, so there's no such thing as a cold city entry for an already-cached version.

---
---
---
---
---
---
---
---
---

**Resume Bullet:**
> Reduced downstream microservice calls by 22% and improved p90 API latency by ~31% (84ms → 58ms) by implementing DTO-level Memcached caching for home and model pages across city-unset and top 10 cities, contributing to a reduction in microservice pod resource utilisation.

---

**Recall Guide:**

**The Problem:**
Every request to the home and model pages triggered a fan-out of downstream microservice calls — fetching prices, filtered data from Elasticsearch, and data from various other microservices — to assemble the final DTO served to the user. Although individual microservices already served their own data from cache, the assembled DTO itself was never cached, meaning every single page request triggered the full set of downstream calls regardless. With 80M monthly user sessions, this was significant unnecessary compute load across the entire microservice ecosystem.

**Your Solution:**
You proposed the idea to your manager, who got it prioritised. You then implemented DTO-level Memcached caching for the home and model pages. Rather than caching all 1,539 cities (DTOs are large and storage has limits), you strategically cached DTOs for **city-unset and the top 10 most popular cities** — which together account for ~90% of total traffic. This gave maximum impact with a controlled memory footprint. Cache expiry was set to **15 minutes**, accepted as a business tradeoff given that car data can change frequently, especially for newly released models. The implementation used the company-wide **AEPLCore.Cache** library which handles cache stampede out of the box.

**The Numbers:**
- City-unset + top 10 cities = **11 cache key variants for home page**
- 4,000 model pages × 11 variants = **~44,000 cache keys for model pages**
- Total: **~44,011 Memcached keys**

**The Stack:** .NET 8 microservice, Memcached, Elasticsearch, AEPLCore.Cache (internal library).

**The Impact:**
- Downstream microservice calls reduced by **22%** across all dependent services (measured via Grafana)
- p90 API latency reduced by **~31%** (84ms → 58ms) for home and model page APIs
- Contributed to microservice pod resource utilisation reduction alongside other optimisations
- Cache is cheaper than compute — this was a deliberate infrastructure philosophy at the company

---

**Likely Interview Deep-Dives:**

**Q: Why only cache city-unset and top 10 cities?**
> DTOs are large objects — storing all 1,539 city variants would be impractical from a memory standpoint. The top 10 cities plus city-unset account for roughly 90% of our traffic, so we got maximum impact with a controlled and justified memory footprint.

**Q: Why was this never done before?**
> Individual microservices already served their data from cache, so the assumption was that the overall response was already optimised enough. It was only after I proposed and implemented DTO-level caching that the team saw how much additional impact was possible at the aggregation layer.

**Q: Why 15 minute expiry?**
> It was a conscious business tradeoff. Cars, especially newly released models, can have frequently changing data. The business determined that anything beyond 15 minutes of staleness was unacceptable, so 15 minutes was the agreed window that balanced performance with data freshness.

**Q: How did you arrive at the 22% reduction?**
> We measured total downstream microservice call volume before and after via Grafana dashboards. The downstream microservices serve many other callers beyond just our page, so even though we eliminated ~90% of our outgoing calls for cached cities, the net impact at the downstream level across all their traffic was 22%.

**Q: Why is the p90 improvement more significant than p80?**
> Because the cached cities — city-unset and top 10 — account for ~90% of traffic. So the improvement is most visible at the 90th percentile, where the bulk of real user requests sit. The majority of requests are now served from cache without any downstream calls.

**Q: How did you handle cache stampede?**
> We used our company-wide internal library AEPLCore.Cache which handles cache stampede out of the box. It's used universally across all microservices in the company.

**Q: Did this cause any issues with stale data?**
> The 15 minute window was an accepted business tradeoff. We were aware that users could see slightly stale data within that window, but the business determined this was acceptable given the performance gains and the relatively low frequency of critical data changes within a 15 minute period.

**Q: How did this contribute to pod downsizing?**
> By eliminating the majority of downstream calls for ~90% of traffic on two of our highest traffic pages, we significantly reduced compute load across multiple microservices. Combined with other optimisations the team made, this contributed to being able to reduce pod resource utilisation across the microservice ecosystem.

---
---
---
---
---
---
---
---
---

**Resume Bullet:**
> Reduced CI pipeline build times by ~50% (25-30 mins → 12-15 mins) across the organisation's largest monorepo by migrating from Lerna to PNPM workspaces, implementing Docker BuildKit layer caching with mounted PNPM and NuGet cache volumes, saving ~5.5 hours of engineering time daily across 90+ developers.

---

**Recall Guide:**

**The Problem:**
The organisation's largest monorepo — where 80% of all engineering work happened — had CI pipeline build times of 25-30 minutes on Jenkins. This affected 90+ developers across multiple teams, with 20+ builds triggered daily. You noticed builds ran significantly faster locally and decided to investigate why on your own time, treating it as a passion project with no story points allocated.

**Root Causes You Identified:**
Two main problems:
- **Dependency installation** was the biggest bottleneck. The pipeline used Lerna bootstrap which was slow, and there was no caching — every build downloaded and installed all dependencies from scratch regardless of whether anything had changed
- **Docker layer caching was ineffective** because the Dockerfile copied all source files before running install (`COPY . .` → `RUN install`), meaning any code change — even a single line — invalidated the dependencies layer and triggered a full reinstall

**Your Solution:**

**1. Lerna → PNPM Migration:**
Migrated the entire monorepo from Lerna bootstrap to PNPM workspaces entirely on your own. This was non-trivial — Lerna had been masking a broken dependency graph through aggressive hoisting. Moving to PNPM's stricter hoisting rules surfaced broken dependency graphs and failed hoisting across the monorepo that had to be diagnosed and fixed individually. This required deep understanding of how module hoisting works and how PNPM's dependency resolution differs from Lerna's. You used Cursor as a productivity tool for mechanical fixes, but all research, diagnosis and decision-making was done independently, informed by studying how industry leaders like Vercel approach monorepo dependency management.

**2. Docker BuildKit Layer Caching:**
Restructured the Dockerfile to copy `package.json` first, then run `pnpm install`, then copy the rest of the source files. This means the dependencies layer is only invalidated when `package.json` changes, not on every code change. Further, you mounted the PNPM global cache directory as a Docker BuildKit cache mount — so even when `package.json` changes, only new/changed packages are downloaded and the rest are served from the mounted cache. The same approach was applied to NuGet cache for the .NET backend services.

**The Stack:** Jenkins, Docker BuildKit, PNPM, Lerna, NuGet, React monorepo, .NET 8.

**The Impact:**
- Build times reduced by **~50%** (25-30 mins → 12-15 mins)
- **~5.5 hours** of engineering time saved daily (20+ builds/day × ~17 mins saved)
- Impacted **90+ developers** across the organisation
- Fixed a long-standing broken dependency graph in the organisation's largest and most active repo
- Entirely self-initiated, completed in **1 week working after hours and weekends**
- Manager specifically praised the initiative for touching infrastructure the team typically never addresses

---

**Likely Interview Deep-Dives:**

**Q: How did you identify this as a problem worth solving?**
> I noticed our CI builds were taking 25-30 minutes but the same builds ran in about a minute locally. That gap told me there was something fundamentally wrong with how the CI environment was handling things, not the code itself. I started investigating on my own time.

**Q: What was the biggest challenge in the Lerna → PNPM migration?**
> Lerna's aggressive hoisting had been masking a broken dependency graph in the monorepo for a long time. When we moved to PNPM's stricter hoisting, it surfaced broken dependency graphs and failed hoisting across multiple packages that had to be individually diagnosed and fixed. Understanding which dependencies belonged where and why the graph was broken required going through the monorepo carefully.

**Q: How does the Docker layer caching work exactly?**
> Docker builds images in layers and caches each layer. The key insight is that a layer is only invalidated if something it depends on changes. Previously we were doing `COPY . .` before `RUN pnpm install` — meaning any code change invalidated the dependencies layer and triggered a full reinstall. By reordering to `COPY package.json` → `RUN pnpm install` → `COPY . .`, the dependencies layer is only rebuilt when package.json actually changes. On top of that, we mount the PNPM global cache as a BuildKit cache mount so even new package installations reuse previously downloaded modules where possible.

**Q: Why PNPM over npm or yarn?**
> PNPM uses a content-addressable store and hard links, meaning packages are never duplicated across projects. In a monorepo with many shared dependencies this is significantly faster and more efficient than npm or yarn. It also has native workspace support making it a natural Lerna replacement.

**Q: Why did you look at how Vercel handles this?**
> I knew dependency installation was the core bottleneck from profiling the pipeline. Vercel is well known for their work on monorepo performance and build optimisation, so I studied their approach to understand what industry best practice looked like and applied those learnings to our stack.

**Q: Did you do this alone?**
> Yes — entirely self-initiated and implemented alone over one week working after hours and weekends. Our team is primarily focused on feature development and microservice work, so CI infrastructure wasn't something anyone was actively maintaining. I just got frustrated with the build times and decided to fix it.

**Q: How did you use Cursor in this?**
> Cursor handled the mechanical labour of writing fixes once I knew what needed to be fixed. The research, diagnosis, and decision-making was all mine — understanding Docker layer caching, evaluating PNPM vs Lerna, identifying the broken dependency graph. I directed Cursor rather than being directed by it.

**Q: Was anything at risk when making these changes?**
> Yes — migrating the dependency management of the organisation's largest and most active monorepo is inherently risky. Broken hoisting meant some things failed unexpectedly during the migration. I worked through these systematically, tested thoroughly before rolling out, and managed it without taking any formal story points or disrupting the team's work.

---
---
---
---
---
---
---
---
---

**Resume Bullet:**
> Reduced Bhrigu microservice average CPU usage by ~62% and RAM by ~43% by implementing client-side batch tracking in React with a debounce mechanism and hard limit of 10, reducing HTTP requests from ~3,000/sec to ~300/sec across 80M monthly users firing 8B tracking events/month, with a beforeunload flush to prevent tracking loss.

---

**Recall Guide:**

**The Problem:**
Bhrigu is an internal microservice responsible for capturing all user tracking data — page views, clicks, impressions, and viewport-based component tracking — used by Product Owners to track feature performance. With 80M monthly users each firing ~100 tracking events, the system was processing approximately **8 billion tracking requests per month (~3,000 requests/second at peak)**. Every single tracking event was being sent individually via `sendBeacon` — meaning the server had to process each request independently, decoding headers, parsing payloads, and handling connections for what was largely identical header data repeated across thousands of requests per second. You identified this inefficiency after overhearing your VP mention to your manager that the system was firing an excessive number of tracking requests.

**Your Solution:**
You implemented a **client-side batching mechanism in React** with two components:

- **Debounce with hard limit:** Instead of sending each tracking immediately via `sendBeacon`, trackings are queued and sent together either when there's a 5 second gap in new tracking events, or when the queue hits the hard limit of **10 trackings** — whichever comes first. In practice, on initial page load viewport impression trackings hit the hard limit of 10 almost immediately. During content browsing, batches of 3-4 trackings are sent after the debounce window.

- **beforeunload flush:** Since the new approach introduced the concept of "pending trackings" that hadn't been sent yet — something that didn't exist in the old fire-immediately approach — you added a `beforeunload` event handler that flushes all remaining pending trackings via `sendBeacon` when the user closes or navigates away from the page, ensuring zero tracking loss.

Individual trackings already contained their own timestamps, so batching had no impact on tracking accuracy or ordering for PO analytics.

**The Stack:** React, JavaScript, sendBeacon API, Bhrigu microservice.

**The Impact (via Grafana/Prometheus):**
- HTTP requests to Bhrigu reduced from **~3,000/sec → ~300/sec (~90% reduction)**
- Bhrigu pod average CPU usage: **~65% → ~25%**
- Bhrigu pod average RAM usage: **~70% → ~40%**
- Zero tracking data loss due to beforeunload flush implementation
- Self-initiated after overhearing VP observation, no story points allocated

---

**Likely Interview Deep-Dives:**

**Q: How did you identify this as a problem?**
> I overheard our VP mention to my manager that we were firing an excessive number of tracking requests. I took that observation, looked into how our tracking worked, and realised every single event was being sent individually via sendBeacon — at 80M users firing ~100 events each, that's 8 billion individual HTTP requests a month. Batching felt like an obvious and impactful fix.

**Q: Why a 5 second debounce and hard limit of 10?**
> The 5 second window balances latency of tracking data reaching the server against batching efficiency. The hard limit of 10 ensures we never hold trackings indefinitely if the user is continuously interacting — on initial page load for example, viewport impression trackings for all visible components fire rapidly and hit the limit of 10 almost immediately, so they get sent without waiting for the full debounce window.

**Q: What about tracking loss when a user closes the tab?**
> That was a key edge case I specifically addressed. In the old system there were no pending trackings since everything was sent immediately. With batching, you introduce pending trackings that could be lost on tab close. I added a beforeunload event handler that flushes all pending trackings via sendBeacon before the page unloads, ensuring zero data loss.

**Q: Did batching affect tracking accuracy?**
> No — each individual tracking event already contained its own timestamp before being queued. So even though trackings arrive at the server in batches, the timestamps accurately reflect when each event actually occurred. PO analytics are completely unaffected.

**Q: Why client-side batching rather than server-side aggregation?**
> Client-side batching reduces the number of HTTP requests hitting the server entirely — which was the core problem. Server-side aggregation would still require processing every individual request. The goal was to reduce server load, so intercepting at the source was the right approach.

**Q: What's the risk of the debounce approach?**
> The main risk was tracking loss on page exit, which I solved with the beforeunload flush. Another theoretical risk is if the debounce window is too long — but 5 seconds is generous enough that for normal user sessions all trackings are sent well within acceptable latency for PO analytics, which don't require real-time data.

---
---
---
---
---
---
---
---
---

**Resume Bullet:**
> Eliminated a sequential microservice call at the API gateway layer by implementing an in-memory masking name dictionary with RabbitMQ-driven cache invalidation, reducing TTFB by ~23% (112ms → 86ms) and MMV microservice call volume by ~30% across 80M monthly users.

---

**Recall Guide:**

**The Problem:**
Every page on the platform used URL-based masking names (e.g. `make-masking-name/model-masking-name`) for SEO purposes, but all internal data was stored against numeric make/model IDs. This meant every page request at the API gateway (carwaleweb) required **3 sequential call phases:**
1. Call MMV microservice to resolve masking names → make ID + model ID
2. Fan-out parallel calls to multiple microservices using resolved IDs
3. Final call based on data received in phase 2

This sequential dependency meant users couldn't get their response until all 3 phases completed. Make, model and version pages together account for ~60% of total traffic, and these pages made 2 MMV calls each — one for masking name resolution and one for other MMV data.

**Your Solution:**
Conceived during an internal company hackathon (where your team won 2nd place), prototyped there and later fully implemented. You built a **masking name dictionary client** as a library in the MMV service, consumed by carwaleweb. The library works as a background worker that:
- Queries the MMV database on startup to build an in-memory dictionary of all make, model and version masking names with lightweight metadata
- Listens to existing RabbitMQ change events to detect MMV data updates and requeries the database to refresh the dictionary
- Resolves masking names → IDs entirely in-memory with no network hop

The in-memory dictionary is lightweight — each entry contains only ints and enums — making the memory footprint negligible even across thousands of makes, models and versions.

**Data Structure:**
```
MaskingNameSnapshot
├── timestamp
├── maskingNamesDictionary        → resolves masking name to full MMV details (Id, MakeId, ModelId, TrimId, MmvStatus, RootId, BodyStylesIds)
├── hyphenatedMaskingNameDictionary → resolves hyphenated variants
└── makeModelDictionary           → ConcurrentBag<ModelInfo> per make-model key
```

**Eventual Consistency Tradeoff:**
There is a brief window between a RabbitMQ change event being published and the worker consuming it where the in-memory dictionary could be stale. This was an accepted business tradeoff — new masking name URLs are never exposed to users until the worker has updated, since the sitemap and URL exposure depend on the same data pipeline. So in practice a stale dictionary never serves an invalid URL to a real user.

**The Stack:** .NET 8, carwaleweb (API gateway), MMV microservice, RabbitMQ, in-memory dictionary.

**The Impact (measured via curl and confirmed on Grafana):**
- TTFB reduced by **~23%** (112ms → 86ms)
- MMV microservice call volume reduced by **~30%** (60% of pages went from 2 MMV calls → 1)
- Eliminated one full sequential network hop from every page request
- Conceived and prototyped at internal hackathon — **won 2nd place**

---

**Likely Interview Deep-Dives:**

**Q: Why not just cache the masking name resolution in Redis/Memcached?**
> We could have, but in-memory resolution is faster than any network cache — there's literally zero network hop. Given masking names change very infrequently (~300-500 changes a year), keeping the full dictionary in memory is practical and the footprint is negligible. A network cache would still add latency compared to in-memory lookup.

**Q: What happens when masking names change?**
> We already had RabbitMQ change events in place for MMV data updates. The worker consumes these events and requeries the database to refresh the in-memory dictionary. Changes happen roughly 300-500 times a year so this is a rare operation.

**Q: What about the window between a change event and the worker updating?**
> There's a brief eventual consistency window, but it's an accepted tradeoff. New masking name URLs are never exposed anywhere — not in the sitemap, not in navigation — until the worker has already updated. So in practice no real user ever hits a stale URL during that window.

**Q: Why was this a hackathon idea?**
> We had identified that sequential microservice calls at the gateway were contributing to TTFB. The hackathon gave us the time and space to prototype a solution quickly. We demoed the prototype, won 2nd place, and it was subsequently prioritised and fully implemented.

**Q: How did you ensure thread safety with the in-memory dictionary?**
> The data structure uses ConcurrentBag for the makeModelDictionary which handles concurrent reads safely. The dictionary is replaced atomically on refresh rather than mutated in place, so readers always see a consistent snapshot.

**Q: Why does this only affect 60% of traffic?**
> Make, model and version pages are the ones that depend on masking name resolution and previously made 2 MMV calls. Other pages either don't use masking name resolution or only made 1 MMV call. So the 30% overall reduction reflects the real traffic distribution accurately.
