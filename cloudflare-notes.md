# Cloudflare Developer Platform: My Notes

The big idea: a **Worker** is the code that runs my app. The other four tools add
things that code can't do by itself: remember data, coordinate users, run on a
schedule, or save work for later.

A Worker reaches each tool through a **binding**, a named connection set up in the
config file (`wrangler.jsonc` or `wrangler.toml`). Example: `env.DB` is my database
and `env.EMAIL_QUEUE` is a queue.

---

## Quick reference

| Tool | What it does | Everyday comparison | Remembers data? |
|---|---|---|---|
| **Workers** | Runs my code on Cloudflare's servers in hundreds of cities | A receptionist with a branch in every city | No, forgets after each request |
| **D1** | SQL database (built on SQLite) | Organized spreadsheets in a filing cabinet | Yes, permanently |
| **Durable Objects** | One single, named instance with its own storage | A single ticket booth for one event | Yes, its own small storage |
| **Cron Triggers** | Runs a Worker on a schedule | An alarm clock / recurring reminder | No |
| **Queues** | Saves slow work for a background Worker to do later | The order-ticket rail in a restaurant kitchen | Holds messages until they're done |

---

## 1. Workers (and workers.dev)

- **What:** Small pieces of code (JavaScript/TypeScript) that run when a request comes
  in, on the server nearest the visitor. No servers to rent, set up or patch.
  Cloudflare runs more copies automatically when traffic grows.
- **workers.dev:** A free web address for every account, like
  `my-app.yourname.workers.dev`. My Worker is live as soon as I deploy. I can add my
  own domain later.
- **Key limitation:** A Worker **forgets everything** after it sends the response.
  That's why the other tools exist.
- **Business examples:**
  - An online shop's API (`/products`, `/cart`, `/checkout`)
  - Sending visitors to the right language version, or A/B testing two page designs
  - A "gatekeeper" that checks login tokens before requests reach older servers
- **Connections:** The hub. It reads and writes D1, talks to Durable Objects, sends
  to and receives from Queues, and runs on Cron Triggers.

## 2. D1

- **What:** Cloudflare's SQL database. Tables, rows and queries (`SELECT`, `INSERT`,
  `JOIN`), with no database server to manage.
- **Problem it solves:** Workers forget everything, so permanent data (users,
  orders, products) needs a home.
- **Business examples:**
  - A bakery's menu, customers and order history
  - A salon booking app's stylists, time slots and appointments
  - A SaaS product's user accounts, plans and settings
- **Connections:** Workers query it. Cron Triggers run nightly jobs on it. Queue
  consumers write results to it. Durable Objects save final results to it.

## 3. Durable Objects (the hardest one)

- **What:** A single, named instance of code plus its own private storage.
  Cloudflare guarantees **exactly one** copy of each named object in the world, and
  it handles requests **one at a time**.
- **Problem it solves:** Normally thousands of Worker copies run separately and
  don't know about each other. When many people share **one exact thing at the
  same moment** (the last ticket, a chat room, a live counter), you need one place
  that has the official answer.
- **Bonus:** Can hold **WebSocket** connections (a live, two-way link with browsers)
  for real-time updates.
- **Business examples:**
  - One object per concert, so the last seats are never sold twice
  - One object per chat room or shared document, sending updates to everyone instantly
  - One object per API customer to limit calls ("98 of 100 this minute")
- **Connections:** Workers send requests to an object by name. Objects can send to
  Queues or save to D1. They also have **alarms**, a personal timer for one object.

## 4. Cron Triggers

- **What:** Runs a Worker automatically on a schedule, using cron syntax.
  Example: `0 2 * * *` means "2:00 AM every day."
- **Problem it solves:** Jobs that no visitor starts, like nightly cleanup or
  hourly updates.
- **In code:** The Worker uses a special `scheduled()` function instead of the
  normal request handler.
- **Business examples:**
  - A daily 8 AM sales summary email
  - A nightly check for expired free trials
  - Hourly price updates from suppliers
- **Connections:** Starts a Worker, usually works with D1, and splits big jobs into
  many small Queue messages (e.g. 5,000 reminder emails = 5,000 messages).

## 5. Queues

- **What:** "Do this later." A **producer** drops a message on the queue and moves
  on. A **consumer** Worker (using a `queue()` function) picks messages up, often in
  batches, and does the work.
- **Problem it solves:** Slow or unreliable work (emails, PDFs, calls to other
  companies' APIs) shouldn't make the user wait or break the whole request.
- **Safety net:** Failed messages are **retried automatically**. Messages that keep
  failing can go to a **dead letter queue** for inspection.
- **Business examples:**
  - Sending receipts and notifying the warehouse after checkout
  - Making thumbnails after a photo upload
  - Collecting millions of click events and saving them in big batches
- **Connections:** Workers, Durable Objects and Cron Triggers can all be producers.
  The consumer is a Worker. Consumers often write results to D1.

---

## How they work together: "Crumb & Co." bakery ordering app

The setup: 50 croissants a day, live order status for customers, and a morning
report for the owner.

```
Customer's browser ──► WORKER (the front desk / API)
                         │          │              │
                         ▼          ▼              ▼
                        D1    DURABLE OBJECT     QUEUE ──► CONSUMER WORKER
                    (menu,    (stock counter,    (slow     (emails, printer)
                    orders)   live status)       jobs)          │
                         ▲                                      │
                         └──────────── saves results ───────────┘

CRON TRIGGER (5:00 AM daily) ──► reset stock, total sales from D1,
                                 queue the owner's report email
```

**A customer orders 2 croissants:**

1. **Worker** reads the menu from **D1** and shows the page.
2. At checkout, the Worker asks the single `stock-today` **Durable Object** to
   reserve 2. Only one counter exists, so the last croissants can't be sold twice.
3. The Worker saves the order in **D1** (`status = 'received'`).
4. The Worker puts "email receipt" and "print kitchen ticket" on the **Queue**, then
   immediately shows "Thanks! Order received."
5. A **consumer Worker** sends the email and print job (retrying if needed), then
   updates the order in D1.
6. The status page keeps a WebSocket open to an `order-881` **Durable Object**. When
   the baker taps "Ready," the browser updates instantly.
7. Every morning, a **Cron Trigger** resets stock to 50, totals yesterday's sales,
   and queues the report email.

**In one sentence each:**
- **Worker:** handles every request
- **D1:** remembers everything
- **Durable Objects:** keep shared things correct and live
- **Queues:** do the slow work in the background
- **Cron Triggers:** handle the daily routine

---

## Suggested learning order

1. A "hello world" Worker on workers.dev
2. Add D1 for the menu
3. Add a Queue for receipts
4. Add a Cron Trigger
5. Add Durable Objects (most advanced)
