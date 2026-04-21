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