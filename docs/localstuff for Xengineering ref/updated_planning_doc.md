# Alternator Regulator Cloud Platform - System Planning Document

// To access database:
// https://supabase-nine-ashy.vercel.app/

## Technology Platform Decision

**Selected Platform: Supabase + Vercel**
- **Backend**: Supabase for PostgreSQL database, auto-generated APIs, and authentication
- **Frontend**: Vercel for dashboard hosting and deployment
- **Security**: True isolation - OTA server separate from dashboard infrastructure
- **AI-friendly development** with extensive code generation capabilities for both platforms
- Predictable costs starting free, scaling to ~$25-45/month for 500 users

### Platform Comparison & Selection Rationale

**Firebase (Rejected):**
- NoSQL database cannot handle complex leaderboard queries (no JOINs, limited sorting)
- Simple queries like "fastest 35ft sailboat" require complex workarounds
- Real-time features good but terrible for analytics
- Unpredictable costs with potential spikes

**Traditional VPS + PostgreSQL (Considered):**
- Perfect for relational data and complex queries
- Full control and predictable costs ($30/month)
- More setup work and server management required
- Less AI development assistance available

**Supabase + Vercel (Selected):**
- **Best of both worlds**: PostgreSQL + modern real-time features + global hosting
- **Security isolation**: Dashboard completely separate from OTA server
- **Auto-generated APIs** reduce backend coding
- **Extensive AI training data** for code generation on both platforms
- **Built-in authentication and security**
- **Zero server management** for dashboard
- Can evolve from simple analytics to social platform

### Hosting Architecture & Security

**Infrastructure Separation:**
- `ota.xengineering.net` → Existing VPS (firmware updates only, unchanged)
- `dashboard.xengineering.net` → Vercel hosting (dashboard frontend)
- `yourproject.supabase.co` → Supabase managed backend (data & APIs)

**Security Benefits:**
- **True isolation**: OTA server compromise cannot affect dashboard
- **Minimal attack surface**: OTA server stays exactly as-is
- **Managed security**: Supabase and Vercel handle security patches
- **Separate credentials**: No shared access between systems

### AI Development Workflow Advantages
- **ChatGPT can generate complete Supabase + Vercel projects**
- **Vercel has massive AI training data** due to popularity
- Standard PostgreSQL queries are well-known to AI
- React dashboard components easily generated
- Authentication flows follow standard patterns
- **Minimal learning curve** for non-web developers
- **4-hour planning doc speed** maintained through development

## 1. User Registration & Profile Data Fields

| Field | Notes | Public/Private |
|-------|-------|----------------|
| UserName | Display name on leaderboards/maps | Public |
| Boat Name/UserID | Must enforce uniqueness | Private |
| Boat Type | Sailboat / Catamaran / Powerboat / Trawler | Public |
| Boat Length (ft) | Rounded to nearest foot | Public |
| Boat Make/Model | | Public |
| Boat Year | | Public |
| Home Port | City/state or general region | Public |
| Usage Type | Liveaboard / Seasonal Cruiser / Weekend Warrior | Public |
| Alternator Brand/Model | | Public |
| Solar System Watts | Nominal system wattage | Public |
| Battery Bank Voltage | 12/24/48V | Public |
| Battery Capacity (Ah) | User input | Public |
| Battery Type | LiFePO4 / AGM / Lead Acid / Other | Public |
| Engine Make | | TBD |
| Engine Horsepower | | TBD |
| User Email | For auth and support only | Private |
| Device UID | ESP32 MAC address | Private |

## 2. Leaderboard Categories

| Category | Definition Notes |
|----------|------------------|
| 🔥 Alternator kWh production (1 day, 1 week, lifetime) | Top total kWh output |
| 🔥 Solar harvest (1 day, 1 week) | Max solar watts * hours |
| ⚠️ Most efficient energy usage | Ratio of solar to alternator kWh (lifetime) -- interpret carefully |
| ✅ Highest instantaneous alternator output | Top amps recorded |
| ✅ Fastest boat (sustained avg over 5 miles) | Requires GPS speed smoothing |
| ✅ Fastest boat by type/size (5ft categories) | Sailboats have "motoring" vs "sailing" categories |
| ⚠️ Fastest over predefined routes | New England to Bahamas, etc. - Needs route logic, MVP unlikely |
| ⚠️ Distance sailed upwind in 1 day | Tough calculation, placeholder |
| ⚠️ Farthest from home | Distance from home port |
| ⚠️ Most remote location | Distance to nearest landmass |
| ⚠️ Days at sea without port entry | Continuous at-sea days |
| ⚠️ Engine hours champion | Total engine runtime |
| ⚠️ Longest trip | Longest continuous GPS track between power cycles |
| ⚠️ Highest and Lowest Average SOC | Battery state efficiency |
| ⚠️ Highest power harvest in 24hrs | Solar + alternator combined |
| ⚠️ Highest power use in 24 hrs | Total consumption |
| ⚠️ Ambient temp/pressure extremes | BMP180/BMP280 data required |
| ⚠️ Highest wind gust / sustained avg wind | Needs wind sensor integration |

## 3. Personal Data Parameters

| Parameter | Aggregation Type | Public/Private |
|-----------|------------------|----------------|
| Miles traveled | Daily, weekly, monthly, yearly | Private |
| Time motoring vs sailing | Daily, weekly, monthly, yearly | Private |
| Solar kWh produced | Daily, weekly, monthly, yearly | Public summary |
| Alternator kWh produced | Daily, weekly, monthly, yearly | Public summary |
| Engine runtime | Daily, weekly, monthly, yearly | Private |
| Alternator runtime | Daily, weekly, monthly, yearly | Private |
| Fuel consumption | Estimated from runtime | Private |
| Battery SOC % | Daily avg, min, max | Private |
| Battery charge cycles | Cumulative | Private |
| Max/min voltage/current/temperature | Daily min/max | Private |
| GPS breadcrumb tracks | Time series | Private (aggregated public summaries) |
| Speed averages | Daily avg/max | Public summary |
| Sailing polar map data | continuously adds new points to average boat speed vs. apparent wind map, using averages over last 10 minutes | Private |
| Maintenance reminders | Based on engine hours (emailed) | Private |
| Messages / Interpersonal comms | Future feature | Private |
| Monthly summaries | Email with global and personal data | Private |

## 4. Global Display Parameters

### MAP DISPLAYS

| Parameter | Displayed |
|-----------|-----------|
| Live user positions (last 24h) | Map pins (green) |
| Historical locations | Heatmap overlay (blue) |
| Popular cruising areas | Aggregated heatmap |
| User breadcrumb trails | Per-user on request |
| Max wind gust locations | Map overlay (if implemented) |
| Average wind maps (lifetime) | Map overlay (if implemented) |
| Average wind maps (today) | Map overlay (if implemented) |
| Average temperature maps (today) | Map overlay (if implemented) |
| Weather integration layer | Weather overlay |
| Clickable boat details | If public profile |

### NUMERICAL/PLOTTED DATA DISPLAYS

| Parameter | Displayed |
|-----------|-----------|
| Total kWh produced (fleet) | Dashboard stat |
| Total devices sold / installed / active | Dashboard stat |
| Leaderboard rankings | Numerical tables |
| Personal performance plots | User dashboard charts |
| Fleet performance trends | Graphs on global dashboard |
| Environmental impact (CO2 saved etc) | Marketing metrics |
| Real-time device count | Dashboard stat |
| Geographic distribution | Heat map |
| Countries visited | Count stat |
| Currently online | Real-time count |

## 5. Data Architecture & Payload Structure

### JSON Payload Organization
- **Nested JSON objects** for related data grouping
- **Integer storage strategy** for decimal values to avoid floating point errors
- **Consistent scaling factors**: voltage × 100, current × 10, GPS × 100000

### Example 6-Hour Payload Structure
```
{
  "timestamp": "2024-01-15T12:00:00Z",
  "device_id": "esp32_aabbccddee",
  "energy": {
    "solar_kwh": 25,          // 2.5 kWh × 10
    "alternator_kwh": 18,     // 1.8 kWh × 10  
    "consumed_kwh": 31        // 3.1 kWh × 10
  },
  "battery": {
    "soc_min": 65, "soc_max": 89, "soc_avg": 78,
    "voltage_min": 1280,      // 12.80V × 100
    "voltage_max": 1420,      // 14.20V × 100
    "charge_cycles": 2
  },
  "alternator": {
    "runtime_minutes": 67,
    "max_amps": 850,          // 85.0A × 10
    "avg_amps": 420           // 42.0A × 10
  },
  "navigation": {
    "distance_miles": 125,    // 12.5 miles × 10
    "avg_speed": 52,          // 5.2 knots × 10  
    "start_lat": 4365900,     // 43.659° × 100000
    "start_lng": -7025700     // -70.257° × 100000
  }
}
```

### Polar Data Collection Strategy
- **Collection frequency**: Every 10 minutes while sailing
- **Data volume impact**: 6x more frequent than other metrics (36 points per 6-hour upload)
- **Storage strategy**: Include in main 6-hour payload for simplicity
- **Future optimization**: May require separate daily upload if payload becomes too large

## 6. Final Decisions Made ✅

### Data Sharing & Privacy
- **All data is shared** - requirement for having the product
- **GPS precision reduced** - remove last digits for privacy
- **Pseudonym usernames** - boat names encouraged
- **Global leaderboards** - no regional divisions
- **All features free** - no monetization strategy needed at this time

### Data Architecture
- **Data format**: JSON for easiest expansion later
- **GPS breadcrumb frequency**: 1 point per hour (6 points per 6-hour upload batch. Need to make sure we have sufficient local space for 180 days)
- **Sailing vs motoring detection**: Engine on/off status determined ESP32-side (splits certain data categories such as max speed becomes "max speed motoring" and "max speed sailing")
- **Local storage strategy**: Use full userdata partition (3.9MB/0x3D0000 bytes) until cloud upload or 180 days is reached. After 180 days, I will try to overwrite oldest data first.
- **Offline storage capacity is a non issue**: ~487 days at 4 uploads/day (~2KB per payload)

### Upload Strategy
- **6-hour upload frequency** - major architectural decision
- **Max/min/average aggregation** - not raw data streams
- **500 user target** - stay in free tier initially
- **Server-side validation** - sanity checks before public data
- **Retry logic**: Don't even try uploading if internet is not detected. Exponential backoff (every 1 min for 1 hr. if failed, then 1x per 2 hours for 6 hours, then wait 24 hours)
- **Only upload new data**
- **Keep 180 days of data locally anyway, no reason not to, unless it's significantly easier to manage if we keep deleting it after confirmed successful upload**

### Authentication & User Management
- **Device-tied authentication**: One-time pairing during initial setup
- **Auto-login from ESP32 app**: Embedded auth tokens provide invisible access
- **External access**: Standard username/password from any device/location
- **Multi-device support**: Simultaneous access from ESP32 + phone + laptop
- **No re-authentication required**: Persistent sessions with optional refresh

### Hosting & Infrastructure Strategy
- **OTA server isolation**: Keep `ota.xengineering.net` completely separate and unchanged
- **Dashboard hosting**: `dashboard.xengineering.net` on Vercel for global performance
- **Backend services**: Supabase for database, APIs, and authentication
- **Security architecture**: True separation prevents cross-system compromise
- **AI development priority**: Chose Vercel for superior AI assistance and documentation

### Server-Side Processing Requirements
- **Route-based leaderboards**: MUST be server-side (geo-fence start/finish line detection)
- **Sailing polar diagrams**: Server-side aggregation of historical data
- **Complex calculations**: "Farthest from home", "Most remote location", "Days at sea"
- **Fleet statistics aggregation**

### Cost & Scaling
- **Target users**: 500 users maximum for initial scale
- **GPS storage cost**: ~$0.18/month for GPS data (1GB over 5 years)
- **Hosting costs**: $0-45/month total (Supabase + Vercel)

### Admin & Moderation
- **Data validation**: Implement framework to discard data beyond hardcoded physical limits
- **Outlier handling**: Delete bad data entirely. IE bad data becomes NAN or just not there
- **Admin privileges**: Required for editing leaderboards and user management

### Email & Messaging (Future)
- **Monthly emails**: Triggered 1st of month, requires unsubscribe mechanism
- **Future messaging**: Can be added later without schema changes. Plan userID structure robustly now.
- **Messaging architecture**: Will require server-mediated (not direct device-to-device due to NAT)

### Technical Implementation Decisions
- **Sailing polar data frequency**: Every 10 minutes while sailing
- **Multi-user support**: ONE USER PER DEVICE
- **MVP complexity reduction**: Drop "Fastest-Over-Route" and "Sailing Polar Plots" from initial release
- **Admin dashboard**: Need separate admin interface for moderation and system management
- **Data retention**: NO AGGREGATION NECESSARY, WE ALREADY SHOWED THAT, ON EITHER ESP32 OR SERVER SIDE. DATA SIZE IS TINY.

## 7. Cost Analysis & Projections

### Service Requirements by Scale

**Getting Started (4-10 users) - $0/month:**
- **Supabase**: Free tier (500MB database, 2GB bandwidth, 500K API requests)
- **Vercel**: Free tier (100GB bandwidth)
- **Runway**: 6+ months completely free

**Scale to 500 Users - $25-45/month:**
- **Supabase Pro**: $25/month (8GB database, 250GB bandwidth, 5M API requests)
- **Vercel Pro**: $20/month (if bandwidth limits exceeded)
- **Total infrastructure cost**: Predictable and manageable

### Projected Usage (500 users)
**Database Storage:**
- User profiles: ~500 × 2KB = 1MB
- Performance data: ~500 users × 4 uploads/day × 2KB × 365 days = ~1.4GB/year
- GPS tracks: ~500 users × 24 points/day × 50 bytes × 365 days = ~220MB/year
- **Total Year 1**: ~1.6GB (fits Pro tier)

**API Requests:**
- Data uploads: 500 users × 4 uploads/day × 30 days = 60,000/month
- Dashboard access: 500 users × 10 sessions/month × 20 requests = 100,000/month
- **Total**: ~160,000/month (fits Pro tier)

**Cost Progression:**
- Months 1-6: Free tier
- Year 1-2: $25/month Pro tier
- Year 3+: $25-45/month depending on usage growth

### Alternative Cost Comparison
- **DigitalOcean VPS**: $30/month (consistent, more management)
- **AWS/Google Cloud**: $40-100/month (complex pricing, can spike)
- **Firebase**: $20-80/month (unpredictable query costs)
- **Self-hosted**: $100-300/month for proper 500-user infrastructure

## 8. Development Timeline & Phases

- User registration and authentication       (WHAT IS AUTHENTICATION?)
- Basic ESP32 data upload to Supabase (I think this is done already)
- Simple leaderboards (speed, energy production)    (not done yet)
- Admin interface for user management     (not done)
- Vercel deployment pipeline       (what does this mean?)
- Personal analytics charts      (these need definition)
- Historical data visualization (needs definition, i would say it's the same thing as "personal analytics charts")
- Advanced leaderboard categories (these can wait, but don't want to make design decisions now that make this difficult)
- Admin moderation tools     (these can wait, but don't want to make design decisions now that make this difficult)
- GPS track visualization  (this is similar to "advanced leaderboard"- something i don't need now but want to keep in back of my mind when making design decisions)
- Fleet statistics (these can wait, but don't want to make design decisions now that make this difficult)
- Email summaries    (probably never going to do this)
- Performance optimizations  (can wait until shown to be needed)
- Real-time notifications  (can wait)
- Basic messaging system (can wait)
- Collaborative features (needs definition, i don't know what this means)
- Advanced analytics  (meaningless buzzword)

## 9. System Architecture

### Overall Flow
**Proposed flow**: ✅ ESP32 assembles 6-hour payload → Supabase REST API → PostgreSQL storage → Vercel-hosted dashboard with device-tied authentication

### Infrastructure Architecture
```
ESP32 Device → Supabase (Backend) → Vercel (Frontend) → User Browser
     ↓              ↓                    ↓
Data Upload    PostgreSQL DB      dashboard.xengineering.net
Auth Tokens    Auto-generated     Global CDN
               REST APIs          
```

### Personal Analytics Dashboard
Historical analysis only, accessible via webapp tab, mobile + desktop compatible.

### Login/Authentication System
1. ESP32 generates unique device ID (MAC address)
2. First-time setup: User connects to ESP32 hotspot for registration
3. Registration webpage auto-populates device ID (invisible to user)
4. User creates username/password → Supabase links account to device
5. ESP32 stores auth token locally for embedded webpage access
6. Daily usage: ESP32 app provides automatic login via embedded token
7. External access: Username/password login from any device
8. Recovery: Factory reset allows re-pairing to new account

### User Experience Flow
**First-Time Setup (5 minutes):**
- Connect to ESP32 WiFi → Registration page opens automatically
- Enter: Username, Email, Password, Boat details
- Device ID pre-filled (user doesn't see technical details)
- Click "Create Account" → Done

**Daily Usage (Invisible):**
- Open ESP32 app → Personal dashboard loads automatically
- No login screens, no password prompts
- Full access to personal data, leaderboards, global stats

**Remote Access (Standard):**
- Visit dashboard.xengineering.net from phone/laptop anywhere
- Username/password login (one-time per device)
- Same dashboard, same data as ESP32 app

### Authentication Technical Details
**Supabase Authentication Strategy:**
- Row Level Security (RLS) ensures users only see their own data
- Device-embedded auth tokens for seamless ESP32 app access
- Standard web authentication for external access
- Permanent device pairing (factory reset only way to unpair)
- Admin override capability for account transfers/support

**Security Considerations:**
- Auth tokens stored securely in ESP32 flash memory
- Tokens can be refreshed/revoked if compromised
- User data isolated via database-level security rules
- Admin access for legitimate account transfers

## 10. Hardware Architecture

### ESP32 Partition Table
| # | Name | Type | SubType | Offset | Size |
|---|------|------|---------|--------|------|
| nvs | data | nvs | 0x9000 | 0x5000 |
| otadata | data | ota | 0xe000 | 0x2000 |
| factory | app | factory | 0x10000 | 0x400000 |
| ota_0 | app | ota_0 | 0x410000 | 0x400000 |
| factory_fs | data | spiffs | 0x810000 | 0x200000 |
| prod_fs | data | spiffs | 0xa10000 | 0x200000 |
| userdata | data | spiffs | 0xc10000 | 0x3D0000 |
| coredump | data | coredump | 0xFF0000 | 0x10000 |

## 11. Data Management

- Users need ability to delete all stored data from database
- GDPR compliance through Supabase data export/deletion features

## 12. Future Expansion Capabilities

### Real-time Social Features (Supabase Enables)
- Live position sharing between friends
- Real-time leaderboard updates
- Instant notifications for achievements
- Group voyage planning
- Weather alerts to nearby boats

### Advanced Analytics (PostgreSQL Enables)
- Complex multi-table queries for insights
- Historical trend analysis
- Predictive maintenance algorithms
- Fleet performance comparisons
- Equipment reliability statistics

### Third-party Integrations
- Weather service APIs
- Marina databases
- Parts supplier catalogs
- Insurance usage-based rates
- Navigation software integration

### Global Scaling (Vercel Enables)
- Automatic global CDN distribution
- Edge computing for regional optimizations
- A/B testing for feature rollouts
- Performance monitoring and optimization