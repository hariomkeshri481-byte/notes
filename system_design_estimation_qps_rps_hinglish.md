# System Design Estimation — QPS, RPS aur Capacity Planning (Hinglish Guide)

Ye estimation wala part step-by-step samjhaya gaya hai — Hinglish mein, taaki interview mein bina confuse hue calculate kar sako. Yeh ek **skill** hai jo practice se aata hai, formula yaad karne se nahi. Isliye ek **fixed process** diya gaya hai jise har question pe apply kar sakte ho.

---

## 1. Sabse pehle — kis order mein calculate karna hai

Hamesha yeh order follow karo, kyunki har cheez agli cheez pe depend karti hai:

```
Users → Daily Active Users (DAU) → Requests per day → QPS/RPS → Storage → Bandwidth → Memory/Cache
```

Interviewer jo bhi numbers de (ya na de), tumhe **reasonable assumptions** khud leni padengi. Yeh normal hai — interviewer yeh dekh raha hai ki tumhara approach sahi hai, exact number nahi.

---

## 2. QPS vs RPS — confusion clear karte hain

- **RPS (Requests Per Second)** = kitne HTTP requests aa rahe hain server pe, per second
- **QPS (Queries Per Second)** = kitne queries ja rahe hain database/backend service pe, per second

Practically interview mein dono ko **same maan sakte ho**, jab tak tumhe specifically differentiate nahi karna. Bas dhyaan rakho:
- Ek user request (1 RPS) → backend mein multiple DB queries trigger kar sakti hai (e.g. 1 search request → 3 QPS: hotel query + price query + review query)

---

## 3. DAU se QPS nikalna — yeh core formula hai

### Step A: DAU nikalo (agar diya nahi hai to assume karo)
Agar company ka scale pata hai (e.g. Booking.com jaisa bada platform), toh reasonable assumption:
- Bada platform: 50M–100M DAU
- Medium platform: 1M–10M DAU
- Startup scale: 10K–100K DAU

**Tip:** Agar interviewer kuch nahi bataye, khud bolo: *"Let's assume 50 million DAU for a platform of this scale"* — aur move on karo. Zyada time mat lagao is pe.

### Step B: DAU ko requests mein convert karo
Har user din mein kitni baar action karta hai, yeh assume karo:

```
Total requests/day = DAU × actions per user per day
```

Example: 50M DAU, average user 4 baar search karta hai = 200M searches/day

### Step C: Requests/day ko QPS (per second) mein convert karo

Yahan ek **magic number yaad rakho: 86,400** (seconds in a day)

```
Average QPS = Total requests per day / 86,400
```

Lekin calculation easy karne ke liye, interview mein log **86,400 ko round karke ~100,000** use karte hain (thoda approximate, but fast):

```
Average QPS ≈ Total requests per day / 100,000
```

**Example:**
200,000,000 requests/day ÷ 100,000 = **2000 QPS average**

(Actual precise answer 86,400 se hoga ~2314 QPS, but 2000 bolna bhi acceptable hai — interviewer ko precision nahi, approach chahiye)

---

## 4. Peak QPS — yeh bhi bolna zaroori hai

Traffic din bhar equal nahi hota — raat ko kam, evening mein zyada. Isliye:

```
Peak QPS = Average QPS × Peak Factor
```

Peak factor ka standard assumption: **2x to 3x** average (kabhi kabhi flash sale/holiday jaisa case ho toh 5x-10x bhi bol sakte ho)

**Example continued:**
Average QPS = 2000 → Peak QPS = 2000 × 3 = **6000 QPS**

Yeh number tumhare **capacity planning** (kitne servers chahiye) ke liye use hota hai.

---

## 5. Read vs Write ratio — bahut important cheez

Zyada tar systems mein **reads writes se bahut zyada hote hain**. Interview mein yeh explicitly bolna chahiye:

```
Search (read) : Booking (write) = kuch aisa hota hai jaise 100:1 ya 1000:1
```

**Example (Booking.com jaisa case):**
- 200M searches/day (reads)
- Conversion rate ~1% → 2M bookings/day (writes)

Isse dikhta hai interviewer ko ki tumhe pata hai reads aur writes ko **alag scale** karna padta hai (jaise search Elasticsearch mein, booking SQL mein).

---

## 6. Storage Estimation

Formula simple hai:

```
Total Storage = Number of records × Size per record × Retention period (agar applicable ho)
```

### Steps:
1. **Ek record ka size decide karo** — realistic assumption lo (e.g. ek booking record = 500 bytes to 1KB agar text fields hain)
2. **Kitne records per day banenge** — writes/day se pata chalega
3. **Kitne saal/din tak store karna hai** — retention period

**Example:**
- 2M bookings/day
- Har booking record ~1KB (500 bytes data + metadata + indexes overhead)
- 5 saal tak store karna hai

```
Daily storage = 2,000,000 × 1KB = 2GB/day
5 years storage = 2GB × 365 × 5 = ~3.65 TB
```

**Yaad rakhne wale units:**
- 1 KB = 10³ bytes
- 1 MB = 10⁶ bytes
- 1 GB = 10⁹ bytes
- 1 TB = 10¹² bytes

Interview mein round figures use karo — precision nahi chahiye, **order of magnitude** sahi hona chahiye (GB vs TB vs PB ka fark pata hona chahiye, exact number nahi).

---

## 7. Bandwidth Estimation

```
Bandwidth = QPS × Average size of request/response
```

**Example:**
- 6000 peak QPS (search)
- Har response ~10KB (hotel list with images metadata, JSON)

```
Bandwidth = 6000 × 10KB = 60,000 KB/s = 60 MB/s
```

Isse tumhe pata chalta hai CDN/network capacity kitni chahiye.

---

## 8. Cache/Memory sizing

Agar interviewer bole "cache design karo popular searches ke liye":

```
Cache size needed = Number of items to cache × Size per item
```

**80/20 rule use karo** (Pareto principle) — often 20% data 80% traffic serve karta hai, isliye pura data cache karne ki zaroorat nahi:

**Example:**
- 1M unique hotels
- Assume top 20% (200K hotels) 80% traffic handle karte hain
- Har hotel cache entry ~2KB

```
Cache size = 200,000 × 2KB = 400 MB
```

Yeh ek single Redis node mein easily fit ho sakta hai — isliye tum bol sakte ho "we don't even need heavy sharding for this cache."

---

## 9. Number of Servers Estimation

```
Number of servers = Peak QPS / QPS handled per server
```

Ek server realistically kitna handle kar sakta hai, yeh bhi assume karna padta hai:
- Simple stateless API server: ~1000 QPS per server (yeh generic assumption hai, context pe depend karta hai)

**Example:**
Peak QPS = 6000, per server capacity = 1000 QPS

```
Servers needed = 6000 / 1000 = 6 servers (plus redundancy, so maybe 8-10 for failover)
```

---

## 10. Poora Example — End to End (yaad karne ke liye practice karo)

**Question: "Booking.com ke liye search system estimate karo"**

1. **DAU assume karo**: 50M
2. **Actions/day**: average user 4 searches karta hai → 200M searches/day
3. **Average QPS**: 200M / 100,000 ≈ 2000 QPS
4. **Peak QPS** (3x): 6000 QPS
5. **Read:Write ratio**: 200M searches vs 2M bookings (1% conversion) → 100:1
6. **Storage** (bookings, 5 years): 2M/day × 1KB × 365 × 5 ≈ 3.65 TB
7. **Bandwidth** (search): 6000 QPS × 10KB ≈ 60 MB/s
8. **Cache size** (top hotels): 200K × 2KB = 400MB
9. **Servers**: 6000 / 1000 = ~6-10 servers with redundancy

Yeh **poora estimation 2-3 minute** mein bol dena chahiye interview mein — practice karo isko bina ruke bolna.

---

## 11. Shortcut numbers jo yaad rakhne chahiye (cheat sheet)

| Cheez | Approx value |
|---|---|
| Seconds in a day | 86,400 (round to 100,000 for quick math) |
| Seconds in a month | ~2.6 million |
| 1 million | 10⁶ |
| 1 billion | 10⁹ |
| Peak traffic multiplier | 2-3x average (10x for flash sales) |
| Read:Write ratio (typical consumer app) | 100:1 se 1000:1 |
| Bytes in ASCII char | 1 byte |
| Typical API response size | 1KB - 50KB (JSON) |
| Typical image size | 100KB - 500KB |

---

## 12. Common mistake jo log karte hain (avoid karo)

1. **Bahut zyada time lagana precise calculation mein** — approximation theek hai, interviewer ko process dikhana hai
2. **Assumptions bolna bhool jaana** — hamesha explicitly bolo "I'm assuming X because Y"
3. **Peak factor bhool jaana** — sirf average QPS bol ke ruk jaana galat hai, capacity planning ke liye peak chahiye
4. **Units mix up karna** — KB/MB/GB/TB mein confuse hona common mistake hai, dhyaan se calculate karo
5. **Round numbers use na karna** — 86,400 ki jagah 100,000 use karo, calculation fast ho jayega aur mistake kam hogi

---

## Practice karne ke liye (khud try karo)

Yeh 3 questions khud solve karo:

1. Instagram jaisa app — 500M DAU, average user din mein 20 posts dekhta hai. Average aur peak QPS nikalo.
2. WhatsApp jaisa messaging app — 1 billion users, average 40 messages/day per user. Storage estimate karo agar messages 1 saal store hote hain aur average message size 200 bytes hai.
3. E-commerce site — 10M DAU, 2% conversion rate to purchase. Peak QPS aur bandwidth nikalo agar average product page response 20KB hai.

Try karo, apna calculation likho — step by step check karke bataya ja sakta hai kahan gadbad hui agar hui toh.
