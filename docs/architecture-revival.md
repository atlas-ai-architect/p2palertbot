# Architecture Decision Record: P2P Alert Bot Revival

**Author:** Atlas (AI Architect)  
**Date:** 2026-02-18  
**Status:** Draft

---

## 1. Context

The p2palertbot is a Telegram bot that alerts users to P2P Bitcoin orders. It currently relies on the lnp2pbot REST API (`/orders` endpoint) which is no longer functional. We need to migrate the data source to Nostr NIP-69 events while adding a freemium model to improve user acquisition.

### Current State
- **Runtime:** Node.js + TypeScript
- **Bot Framework:** Telegraf (Telegram)
- **Database:** MySQL (Prisma ORM)
- **Payment:** LNbits (Lightning invoices)
- **Data Source:** ~~lnp2pbot REST API~~ (BROKEN)
- **Partial Nostr:** Only publishes kind:1 announcements; does not subscribe

---

## 2. Decision: NIP-69 as Primary Data Source

### 2.1 Rationale

NIP-69 (P2P Order Events, kind:38383) is the emerging standard for P2P Bitcoin order broadcasting. Multiple platforms (Mostro, RoboSats, Peach, lnp2pBot) publish to Nostr relays, providing:
- **Decentralized:** No single point of failure
- **Multi-platform:** One integration covers all NIP-69 compliant platforms
- **Real-time:** WebSocket subscriptions vs HTTP polling
- **Verifiable:** Cryptographic signatures on orders

### 2.2 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Nostr Relays                              │
│  wss://relay.mostro.network                                  │
│  wss://relay.lnp2pbot.com                                    │
│  wss://relay.damus.io                                        │
└────────────────────┬────────────────────────────────────────┘
                     │ WebSocket (kind:38383 subscribe)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              NIP69Listener (NEW COMPONENT)                   │
│  - Subscribe to multiple relays                              │
│  - Parse NIP-69 events into Order interface                  │
│  - Emit parsed orders to alert matching logic                │
└────────────────────┬────────────────────────────────────────┘
                     │ Order[]
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              OrdersUpdater (REFACTORED)                      │
│  - Remove HTTP polling                                       │
│  - Consume events from NIP69Listener                         │
│  - Same alert matching logic                                 │
│  - Add freemium rate limiting                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Database (MySQL/Prisma)                         │
│  - Order cache (deduplication)                               │
│  - User, Alert, Subscription, Payment, Delivery              │
│  - NEW: Tier field on User                                   │
│  - NEW: Daily notification tracking                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 NIP-69 Event Parsing

**Event Structure (kind:38383):**
```json
{
  "kind": 38383,
  "tags": [
    ["d", "order-id"],
    ["k", "sell"],
    ["f", "EUR"],
    ["s", "pending"],
    ["amt", "100000"],
    ["fa", "5000"],
    ["pm", "SEPA"],
    ["premium", "3.5"],
    ["source", "https://mostro.network/order/abc123"]
  ]
}
```

**Mapping to Order Interface:**
| NIP-69 Tag | Order Field | Notes |
|------------|-------------|-------|
| `d` | `_id` | Order identifier |
| `k` | `type` | "buy" or "sell" |
| `f` | `fiat_code` | Fiat currency (EUR, USD, etc.) |
| `s` | `status` | pending, active, canceled, etc. |
| `amt` | `amount` | Sats amount |
| `fa` | `fiat_amount` | Fiat amount |
| `pm` | `payment_method` | Payment method |
| `premium` | `price_margin` | Premium/discount % |
| `source` | (new) `source_url` | Link to platform order |

**Code Implementation:**
```typescript
// src/nip69-listener.ts
import { SimplePool, Event } from 'nostr-tools'

interface NIP69Order extends Order {
  sourceUrl?: string
  platform?: string
}

export function parseNIP69Event(event: Event): NIP69Order {
  const getTag = (name: string) => event.tags.find(t => t[0] === name)?.[1]
  
  return {
    _id: getTag('d') || event.id,
    type: getTag('k') || 'unknown',
    fiat_code: getTag('f') || 'unknown',
    status: getTag('s') || 'unknown',
    amount: parseInt(getTag('amt') || '0'),
    fiat_amount: parseFloat(getTag('fa') || '0'),
    payment_method: getTag('pm') || '',
    price_margin: parseFloat(getTag('premium') || '0'),
    source_url: getTag('source'),
    // ... other fields with defaults
  }
}
```

### 2.4 Relay Selection Strategy

**Primary Relays:**
1. `wss://relay.mostro.network` — Mostro (highest NIP-69 activity)
2. `wss://relay.lnp2pbot.com` — lnp2pBot (if operational)
3. `wss://relay.damus.io` — General (may have RoboSats/Peach)

**Fallback Strategy:**
- Maintain connection to all configured relays
- If primary fails, others continue providing events
- Log connection health for monitoring

---

## 3. Decision: Freemium Model

### 3.1 Rationale

Current model requires payment upfront → high friction → low conversion. Freemium allows:
- **Zero-friction onboarding:** Try before buy
- **Viral potential:** Free users can still get value and share
- **Clear upgrade path:** Hit limit → prompted to subscribe

### 3.2 Tier Design

| Feature | Free Tier | Paid Tier |
|---------|-----------|-----------|
| Max Alerts | 3 | Unlimited |
| Daily Notifications | 10 | Unlimited |
| Multi-platform | ✅ | ✅ |
| Priority Support | ❌ | ✅ |

### 3.3 Implementation

**Database Changes:**
```prisma
model User {
  id              Int           @id @default(autoincrement())
  telegramId      BigInt        @unique
  chatId          BigInt
  language        String        @default("en")
  tier            String        @default("free") // "free" | "paid"
  alert           Alert[]
  subscription    Subscription[]
  deliveries      Delivery[]
  notifications   NotificationLog[] // NEW
}

model NotificationLog {
  id          Int       @id @default(autoincrement())
  userId      Int
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  date        DateTime  @default(now())
  count       Int       @default(1)
  
  @@unique([userId, date])
}
```

**Alert Creation Limit:**
```typescript
// In handleAddAlert()
const MAX_FREE_ALERTS = 3

if (user.tier === 'free') {
  const alertCount = await db.alert.count({ where: { userId: user.id } })
  if (alertCount >= MAX_FREE_ALERTS) {
    return ctx.reply(ctx.t('free_tier_alert_limit'))
  }
}
```

**Daily Notification Limit:**
```typescript
const MAX_DAILY_NOTIFICATIONS_FREE = 10

async function canNotify(user: User): Promise<boolean> {
  if (user.tier === 'paid') return true
  
  const today = new Date().toISOString().split('T')[0]
  const log = await db.notificationLog.findUnique({
    where: { userId_date: { userId: user.id, date } }
  })
  
  if (!log) {
    await db.notificationLog.create({ data: { userId: user.id, date, count: 1 } })
    return true
  }
  
  if (log.count >= MAX_DAILY_NOTIFICATIONS_FREE) {
    return false
  }
  
  await db.notificationLog.update({
    where: { id: log.id },
    data: { count: { increment: 1 } }
  })
  return true
}
```

**Upgrade Flow:**
1. User hits limit → Bot replies with upgrade prompt
2. `/subscribe` → Shows pricing, generates LNbits invoice
3. Payment confirmed → `user.tier = "paid"`
4. Immediate access to unlimited features

---

## 4. Decision: Multi-Platform Support

### 4.1 Rationale

NIP-69 abstracts platform differences. Supporting multiple platforms increases:
- **Order volume:** More orders = more alert matches
- **User value:** One bot covers all their P2P needs
- **Resilience:** Not dependent on single platform

### 4.2 Platform Detection

**From NIP-69 `source` tag:**
```typescript
function detectPlatform(sourceUrl: string): string {
  if (sourceUrl.includes('mostro')) return 'Mostro'
  if (sourceUrl.includes('robosats')) return 'RoboSats'
  if (sourceUrl.includes('peach')) return 'Peach'
  if (sourceUrl.includes('lnp2pbot') || sourceUrl.includes('p2plightning')) return 'lnp2pBot'
  return 'Unknown'
}
```

**Notification Formatting:**
```
🟢 Mostro: Buy EUR @ -3% (SEPA)
Amount: 100,000 sats (€50)
https://mostro.network/order/abc123

🔴 RoboSats: Sell USD @ +1% (Venmo)
Amount: 500,000 sats ($300)
https://robosats.com/order/xyz789
```

### 4.3 Removing Hardcoded References

**Current:** Links to `t.me/p2plightning` in notifications  
**New:** Use `source` tag from NIP-69, fallback to generic message

---

## 5. Component Refactor Plan

### 5.1 New: `nip69-listener.ts`

**Responsibilities:**
- Manage WebSocket connections to relays
- Subscribe to kind:38383 events
- Parse events into Order interface
- Emit to OrdersUpdater

**Interface:**
```typescript
export class NIP69Listener {
  private pool: SimplePool
  private relays: string[]
  private onOrder: (order: NIP69Order) => void
  
  constructor(relays: string[], onOrder: (order: NIP69Order) => void)
  start(): void
  stop(): void
}
```

### 5.2 Refactor: `orders-updater.ts`

**Changes:**
- Remove `axios` and HTTP polling
- Remove `LNP2PBOT_BASE_URL` dependency
- Accept orders from NIP69Listener
- Add freemium notification limits

**New Interface:**
```typescript
export class OrdersUpdater {
  private db: Database
  private onNotification: OnNotificationEvent
  private listener: NIP69Listener
  
  constructor(relays: string[])
  start(onNotification: OnNotificationEvent): void
  stop(): void
  private handleOrder(order: NIP69Order): Promise<void>
  private canNotify(user: User): Promise<boolean> // Freemium check
}
```

### 5.3 Extend: `nostr.ts`

**Changes:**
- Keep existing `NostrNotifier` (publish announcements)
- Add `NIP69Listener` class (subscribe to orders)
- Export both from module

### 5.4 Extend: `db.ts`

**New Methods:**
```typescript
// Freemium tracking
async incrementNotificationCount(userId: number): Promise<void>
async getNotificationCount(userId: number, date: string): Promise<number>

// Tier management
async setUserTier(userId: number, tier: 'free' | 'paid'): Promise<void>
```

### 5.5 Extend: `index.ts` (Telegram Bot)

**Changes:**
- Add tier check in `/addalert` command
- Add upgrade prompt messages
- Update `/subscribe` to set `tier = "paid"`

---

## 6. Migration Strategy

### Phase 1: Nostr Integration (Week 1)
1. Create `nip69-listener.ts`
2. Refactor `orders-updater.ts` to use listener
3. Test with live relays (observe order flow)
4. Remove HTTP polling code

### Phase 2: Freemium Model (Week 2)
1. Database migration (add `tier`, `NotificationLog`)
2. Implement alert limits
3. Implement notification rate limiting
4. Update bot commands

### Phase 3: Multi-Platform Polish (Week 3)
1. Platform detection
2. Notification formatting
3. Remove hardcoded links

### Phase 4: Deploy (Week 4)
1. Environment setup
2. Database migration
3. Deploy and monitor

---

## 7. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| NIP-69 adoption low | Low order volume | Monitor relay activity pre-launch; fallback to direct platform APIs if needed |
| Relay connectivity issues | Missed orders | Multiple relays; connection health monitoring |
| Free tier abuse | Resource exhaustion | Rate limits by user; consider CAPTCHA for suspicious patterns |
| Nostr event format changes | Parsing breaks | Version detection; graceful degradation |
| Breaking changes in deps | Compilation errors | Pin versions; test before deploy |

---

## 8. Open Questions

1. **Relay research:** Need to validate which relays have active NIP-69 traffic
2. **Free tier limits:** 3 alerts / 10 notifications reasonable? A/B test?
3. **Pricing:** Daily vs monthly subscription model?
4. **Platform priority:** Mostro first, or all four from day one?

---

## 9. References

- [NIP-69 Specification](https://github.com/nostr-protocol/nips/blob/master/69.md)
- [Mostro Documentation](https://mostro.network/)
- [Nostr Tools](https://github.com/nbd-wtf/nostr-tools)

---

**Next Step:** Forge implements Phase 1 based on this architecture.
