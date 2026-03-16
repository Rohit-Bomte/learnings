# Industry-Level Web Hosting Resource Allocation Guide

When allotting resources from your 102-core / 256GB pool, your goal is to balance performance, cost, and reliability. Follow this framework to provide "on-the-spot" estimates.

## 1. The Core Allocation Logic (CPU)

CPU is primarily driven by **Concurrency**. A single core can only do one thing at a time, but multithreading allows it to switch fast.

### Formula for Concurrency
`Cores Needed = (Peak Concurrent Users * Average Processing Time per Request) / 1000ms`

*Example: 50 concurrent users, each request takes 200ms of CPU time.*
`(50 * 200) / 1000 = 10 Cores.`

### "On-the-Spot" Cheat Sheet
| Site Type | Concurrent Users | Recommended Cores |
| :--- | :--- | :--- |
| **Static / Portfolio** | 100 - 500 | 1 - 2 Cores |
| **Simple WordPress / Blog** | 50 - 100 | 2 - 4 Cores |
| **E-commerce (WooCommerce)** | 20 - 50 | 4 - 8 Cores |
| **Logic-Heavy SaaS (Node/Python)** | 10 - 20 | 8 - 12 Cores |

---

## 2. The Storage Allocation Logic

Storage isn't just about the code. It's about the "Bloat Factor" over time.

### The Component Breakdown
1. **Application Files**: Usually <1 GB (unless it's a huge CMS).
2. **Media Assets**: Images/Videos (The biggest variable).
3. **Database**: Starts small (~100MB), grows by ~1MB per 1k users/orders.
4. **Logs**: Standard retention can take 5GB+ per month for busy sites.
5. **Backups**: You MUST keep local snapshots (Size * Retention Count).

### The Rule of Thumb
`Total Storage = (Site Size + Predicted Data Growth + 2x Snapshot Size) * 1.25 (Buffer)`

**Scenario**: A 5GB WooCommerce site with daily backups for 7 days.
`5GB (Site) + 7 * 5GB (Backups) = 40GB`. With buffer: **50GB**.

---

## 3. Client Interview Checklist

Ask these 4 questions to give an instant estimate:
1. **"What is your expected peak traffic?"** (The highest number of people on the site at the *same second*).
2. **"Does your site involve processing?"** (Image uploads, heavy calculations, video transcoding = 2x Cores).
3. **"How many images/videos will you upload monthly?"** (Estimates storage growth).
4. **"What is your uptime requirement?"** (Requires redundancy/clustering, doubling resources).

---

## 4. Costing & Profitability Strategy

Since you have a **fixed pool (102C / 256GB)**, think of it as "Floor Space" in a building.

### Calculation
- **Your Cost per Core**: `(Total Infrastructure Cost / 102)`
- **Your Cost per GB**: `(Total Infrastructure Cost / 256)`
- **The "Client Price"**: `(Resource Cost + Management Fee + Support Markup)`.

> [!TIP]
> **Oversubscription**: Industry standards allow for "Bursting". If you have 102 cores, you can actually sell up to 150-200 vCPUs because not every client peaks at the same time. Never oversubscribe storage.

---

## 5. Summary Recommendation
For an average dynamic website (moderate traffic), quote **4 Cores and 20GB Storage** as a "Standard" tier. Adjust baseline based on the site's profile.
