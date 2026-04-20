# AWS RDS Database Version Upgrade (EOL → New Version)

## 1. End of Life (EOL) in PostgreSQL (or DB engines)

When a database version reaches **End of Life (EOL)**:

-  No security updates
-  No bug fixes
-  No community support
-  No ecosystem/tool compatibility guarantee

### Example
PostgreSQL 10 → EOL

AWS cannot continue full support because:
- Running unsupported DB versions is a security risk
- Exposes systems to known and unknown vulnerabilities
- Can violate compliance requirements

---

## 2. What EOL actually means

### Security fixes stop
- New vulnerabilities will NOT be patched
- Systems remain permanently exposed

###  Bug fixes stop
- Crashes remain unresolved
- Performance issues persist
- Replication bugs remain

### Ecosystem breaks over time
- Extensions stop supporting old versions
- Tools may fail to install or run

---

## 3. Why AWS enforces upgrades

AWS enforces upgrades because:
- Security compliance 
- Multi-tenant infrastructure risk
- Preventing data breaches
- Maintaining platform reliability

AWS cannot patch old PostgreSQL versions themselves because:
- DB engine is community-maintained
- Fork maintenance is complex and risky
- Upstream support is required for stability

---

## 4. Why upgrading is not simple

### Stable version (old DB)
- Proven in production
- Predictable query behavior
- Known performance patterns

### New version (risk area)
- Query planner changes
- Extension compatibility issues
- Performance differences
- Behavior changes (data, locks, defaults)

 Even small changes can impact production systems

---

## 5. Upgrade risk overview
- Performance may improve or degrade
- Queries may behave differently
- Extensions may break
- Downtime may be required

 Hence, upgrades require proper planning

---

## 6. Upgrade approaches (High Level)

### 1. In-place Upgrade


### 2. Snapshot → Restore → Upgrade

### 3. Logical Replication

### 4. Blue/Green Deployment

### 5. AWS DMS (Database Migration Service)

---

# Method 1: In-place Upgrade (Modify DB Instance)

### What actually happens internally
When upgrading PostgreSQL 10 → 13 in-place:

- AWS takes a final snapshot (safety backup)
- Stops the DB instance
- Runs internal upgrade tool (pg_upgrade)
- Applies system catalog + data updates
- Restarts DB with new version

👉 Same instance becomes PostgreSQL 13

---

### Possibilities (What you can do)

### Direct major upgrade
- 10 → 13 (if AWS supports direct path)
- Or step upgrades: 10 → 11 → 12 → 13

 ### Control timing
- Immediate upgrade
- Or maintenance window-based upgrade

 ### Pre-check compatibility
- AWS runs pre-upgrade validation
- Extension compatibility checks

 ### Rollback (limited)
- Only via snapshot restore
- No instant rollback option

---

### Problems you may face

 ### 1. Downtime (major drawback)
- DB becomes unavailable during upgrade
- Duration depends on DB size & complexity
- Minutes (small DB) → hours (large DB)

 ### 2. Extension incompatibility
- Some extensions may not support new version
- Must be upgraded before DB upgrade

 ### 3. PostgreSQL behavior changes
- Query planner changes
- Deprecated features removed
- Slight SQL behavior differences

 ### 4. Replication impact
- Read replicas may need recreation or upgrade

 ### 5. Storage constraints
- Needs temporary free space
- Low storage may cause failure

---

### Risks (what can go wrong)

 ### Upgrade failure → extended downtime
- Requires restore from snapshot

 ### Application issues after upgrade
- Query failures
- ORM incompatibility

 ### Performance regression
- Slower query execution plans

 ### No downgrade option
- Only recovery = snapshot restore

---

### 🛡️ What you must take care of

 Take full snapshot before upgrade

 Test in staging environment
- Restore production snapshot
- Run full application validation

 Check extensions
```sql
SELECT * FROM pg_extension;
```
---

# Method 2: Snapshot → Restore → Upgrade

## What this method actually is
Instead of touching your live DB:

- Take a snapshot of your PostgreSQL 10 database  
- Restore it as a new DB instance  
- Upgrade that new instance to PostgreSQL 13  
- Test it  
- Switch your application to the new DB  

👉 Your original DB stays untouched

---

##  What possibilities this gives you

### 1. Safe experimentation (biggest strength)
You can:
- Test upgrade multiple times  
- Try different upgrade paths (10→12→13 vs direct)  
- Validate queries and performance  

👉 Zero risk to production

---

### 2. Parallel environments
You now have:
- Old DB (PostgreSQL 10)  
- New DB (PostgreSQL 13)  

You can compare:
- Query results  
- Performance  
- Behavior differences  

---

### 3. Easy rollback strategy
If something goes wrong:

👉 Just don’t switch traffic  

No rollback needed because:
- Production DB is untouched

---

### 4. Staging / rehearsal capability
This is how real teams use it:

- Restore snapshot in staging  
- Practice upgrade  
- Fix issues  
- Repeat until confident  

---

### 5. Flexible upgrade approach
You can:
- Do step upgrades (10→11→12→13)  
- Test extension compatibility gradually  

---

## ⚠️ Problems you will face

###  1. Data drift (major issue)
After snapshot:
- Production DB keeps receiving new data  
- Restored DB becomes outdated  

👉 This creates data mismatch

---

###  2. No live sync
Unlike Blue/Green:
- No replication between old and new  

You must manually handle:
- Final data sync  
- Downtime window  

---

###  3. Cutover complexity
Switching app to new DB involves:
- Changing endpoint  
- Restarting services  
- Ensuring no writes during switch  

---

###  4. Large database restore time
For big DBs:
- Snapshot restore can take hours  

---

###  5. Double infrastructure cost
You temporarily run:
- Old DB + New DB  

👉 Increased cost during migration

---

## 🚨 Risks (real-world)

 ### Risk 1: Data loss during cutover  
If writes are not handled properly:
- Transactions during switch may be lost  

---

 ### Risk 2: Inconsistent data  
If:
- Some data exists only in old DB  
- Some only in new DB  

👉 Leads to corruption-like issues

---

 ### Risk 3: Application misconfiguration  
- Wrong endpoint  
- Old DB still receiving traffic  

---

 ### Risk 4: Time gap too large  
If switching is delayed:
- New DB becomes outdated  

---

## 🛡️ What you must take care of

###  1. Plan cutover strategy (critical)
Controlled switch:

- Stop writes (maintenance mode)  
- Take final snapshot or sync  
- Switch app to new DB  
- Resume traffic  

---

###  2. Minimize data gap
Options:
- Take snapshot as late as possible  
- Or use logical replication (advanced)  

---

###  3. Test upgrade thoroughly
Before switching:
- Validate schema  
- Run queries  
- Check performance  

---

### 4. Validate extensions
Ensure:
- Compatibility with PostgreSQL 13  

---

### 5. Endpoint switch planning
Since new DB has new endpoint:
- Update configs carefully  
- Use DNS or config management  

---

###  6. Keep old DB for fallback
After switch:
- Do NOT delete old DB immediately  
- Keep it as rollback backup  

---

## 📊 When should you use this method?

### Good choice if:
- You want safe testing  
- You are doing upgrade rehearsal  
- You can accept downtime during cutover  

---

###  Avoid if:
- Near-zero downtime is required  
- Continuous heavy write workload exists  
- You need live sync between databases

--- 


# Method 3: Logical Replication (DIY Upgrade Path)

## 🧠 What logical replication actually is
In PostgreSQL, logical replication means:

👉 Instead of copying raw disk data (physical replication),  
it replicates **data changes (INSERT, UPDATE, DELETE)** at table level.

---

## ⚙️ How the upgrade works (step-by-step)

### 🟦 Source (Old)
- PostgreSQL 10 (Production)

### 🟩 Target (New)
- PostgreSQL 13 (New RDS instance)

---

### Flow:

- Create new PostgreSQL 13 instance  
- Enable logical replication on source  
- Create **publication** (what to send)  
- Create **subscription** on target (what to receive)  
- Initial full data copy happens  
- Continuous real-time sync starts  
- Cutover (switch application to new DB)

---

## ✅ Possibilities (why this is powerful)

### 1. Near-zero downtime
- Data sync happens continuously  
- Final switch takes only seconds  

👉 Biggest advantage of this method

---

### 2. Cross-version upgrade supported
Works across major versions:
- PostgreSQL 10 → 13  
- Even larger version jumps possible  

---

### 3. Full control over migration
You can:
- Migrate selected tables only  
- Transform schema if needed  
- Validate before final cutover  

---

### 4. No impact on source DB structure
- Source DB continues running normally  
- No restart required  
- No downtime on production DB  

---

### 5. Gradual migration possible
- Move system in parts  
- Not all-at-once migration  

---

## ⚠️ Problems you WILL face

###  1. Not everything is replicated
Logical replication does NOT include:
-  Sequences (auto-increment values)  
-  Some DDL changes (ALTER TABLE, etc.)  
-  Indexes (must be created manually on target)  
-  Roles / users  

👉 Must be handled manually

---

###  2. Schema must match
Before replication:
- Tables must already exist in target  
- Structure must match exactly  

---

###  3. Initial data load time
- Large databases take hours or even days  
- Initial sync is resource-heavy  

---

###  4. Replication lag
- High write load can delay target sync  
- Target may fall behind source  

---

###  5. Write conflicts during cutover
If both DBs are writable:
- Data inconsistency risk  
- Conflicting writes may occur  

---

###  6. Operational complexity
You must manage:
- Replication slots  
- Monitoring lag  
- Failures and retries  

👉 Not beginner-friendly

---

## 🚨 Risks (serious ones)

 ### Risk 1: Data inconsistency  
- Missing objects or manual steps can desync systems  

---

 ### Risk 2: Sequence mismatch  
Example:
- Source ID = 1000  
- Target ID still = 800  

---

 ### Risk 3: Cutover mistakes  
- Dual writes  
- Data divergence between DBs  

---

## 🛡️ What you MUST take care of

###  1. Prepare schema properly
- Create tables in target DB first  
- Ensure exact structure match  
- Apply indexes manually  

###  2. Handle sequences manually
After sync:

```sql
SELECT setval('my_seq', (SELECT MAX(id) FROM table));
```
---

# Method 4: Blue/Green Deployment

## 🧠 What Blue/Green Deployment actually is
Think of it as AWS giving you a safe parallel environment for your database.

- **Blue = Current production DB (PostgreSQL 10)**
- **Green = New environment (PostgreSQL 13)**

AWS keeps both environments in sync and allows a controlled switch.

👉 No risky in-place upgrade  
👉 No manual replication setup required  

---

## ⚙️ How it works internally

Under the hood:

- For **major version upgrades** → AWS uses logical replication  
- For **same version upgrades** → AWS uses physical replication  

---

## 🔄 Flow

- Create Green environment  
- Set up replication Blue → Green  
- Initial full data sync  
- Continuous replication (near real-time)  
- Validate Green environment  
- Perform switchover (Blue → Green)  

---

## 🔁 Lifecycle (step-by-step)

### Step 1: Create Blue/Green deployment
AWS creates:
- New DB instance (Green)
- Same or modified configuration
- Target version (PostgreSQL 13)

---

### Step 2: Data replication starts
- Initial full data copy
- Continuous sync begins
- Production (Blue) remains active

---

### Step 3: Test Green environment
You can:
- Run queries
- Test application
- Validate schema
- Benchmark performance  

👉 Without impacting production

---

### Step 4: Switchover
When ready:

- AWS pauses writes briefly  
- Ensures replication catch-up  
- Switches endpoints  

👉 Downtime: usually seconds to < 1 minute  

---

###  Step 5: Green becomes production
- Green → new production  
- Blue → kept temporarily for rollback (optional)  

---

## ✅ Key Benefits

### 1 Near-zero downtime
- Switchover is very fast  
- No long maintenance window  

---

### 2 Safe testing before cutover
- Full validation on Green before production switch  

---

### 3 Built-in replication
AWS manages:
- Replication setup  
- Sync process  
- Failover handling  

---

### 4 Easy rollback (early stage)
- Before switchover → just cancel  
- After switchover → rollback possible if Blue retained  

---

### 5 Production-grade reliability
Used in:
- Enterprise systems  
- High-traffic applications  

---

### 6 Supports major upgrades
- Works for PostgreSQL 10 → 13 upgrades  

---

## ⚠️ Problems & limitations

### 1. Not everything is perfectly replicated
- Sequences need validation  
- Some extensions may behave differently  

---

### 2. Version compatibility constraints
- Not all upgrade paths supported directly  
- May require intermediate upgrades  

---

###  3. Replication lag
- High write load can delay sync  
- Green may lag behind Blue  

---

###  4. Temporary double cost
You run:
- Blue + Green simultaneously  

👉 Increased cost during migration  

---

###  5. Feature limitations
Some configurations/extensions may not be fully supported  

---

## 🚨 Risks (real-world)

 ### Risk 1: Application incompatibility  
- App may fail after switchover even if DB is fine  

---

 ### Risk 2: Sequence mismatch  
- Auto-increment values may desync  

---

 ### Risk 3: Switchover issues  
- Replication lag may delay or block switch  

---

 ### Risk 4: Performance differences  
- Green may behave differently under real traffic  

---

## 🛡️ What you MUST take care of

### 1. Pre-check version support
- Ensure PostgreSQL 10 → 13 path is supported  
- If not, plan intermediate upgrade  

---

### 2. Test thoroughly on Green
Do not skip:
- Query validation  
- App testing  
- Performance benchmarking  

---

### 3. Monitor replication lag
Before switchover:
- Lag must be near zero  

---

### 4. Validate sequences
- Ensure auto-increment consistency  

---

### 5. Plan switchover timing
- Perform during low traffic period  

---

### 6. Keep Blue after switch
- Do not delete immediately  
- Use as rollback option  

---

### 7. Check extensions
- Ensure compatibility with PostgreSQL 13  

---

## 📊 When to use Blue/Green

### Best choice if:
- Production system  
- Near-zero downtime required  
- Medium to large databases  
- Critical applications  

---

### Overkill if:
- Very small database  
- Downtime is acceptable  
- Simple non-critical system  

---

## 🧠 Deep insight
Blue/Green = Managed replication + controlled switchover

It solves:
- Data drift issues (snapshot method)  
- Manual replication complexity  
- Downtime issues (in-place upgrade)  

---

## 🧩 Real-world architecture

Users → Application → Blue (PostgreSQL 10)  
                     ↓  
               Replication  
                     ↓  
               Green (PostgreSQL 13)  

Switchover → Users → Green  

---

## 🔑 Final takeaway
- Safest production upgrade method  
- Minimal downtime  
- Fully managed by AWS  
- Requires strong validation discipline  

---

# Method 5: AWS DMS (Database Migration Service)

AWS DMS (Database Migration Service) is a managed data movement service that enables database migration with minimal downtime.

It can:
- Move data from one database to another
- Keep both databases in sync during migration
- Support continuous replication (CDC = Change Data Capture)

👉 Think of it as an always-on data copy + sync engine

---

## 🏗️ Core Architecture

AWS DMS has 3 main components:

### 1. Source Database
Example:
- PostgreSQL 10 (current production system)

---

### 2. Replication Instance (DMS Engine)
This is the core of DMS:
- Reads data from source
- Applies transformations (if needed)
- Writes data to target

---

### 3. Target Database
Example:
- PostgreSQL 13 (new RDS instance)

---

### 🔄 Data Flow

Source DB (PostgreSQL 10)  
&nbsp;&nbsp;&nbsp;&nbsp;↓  
DMS Replication Instance  
&nbsp;&nbsp;&nbsp;&nbsp;↓  
Target DB (PostgreSQL 13)

---

## ⚙️ How AWS DMS works (step-by-step)

### Phase 1: Full Load
- Copies all existing data from source → target

---

### Phase 2: Change Data Capture (CDC)
Continuously captures changes:
- INSERT
- UPDATE
- DELETE  

👉 Keeps source and target in sync in near real-time

---

### Phase 3: Cutover
- Stop writes briefly
- Ensure replication is fully caught up
- Switch application to new DB

---

## 🚀 Key Features

### 1. Heterogeneous migration support
Supports migration between different engines:
- Oracle → PostgreSQL
- MySQL → PostgreSQL
- PostgreSQL → Aurora PostgreSQL

---

### 2. Continuous replication (CDC)
- Near real-time data sync
- Enables minimal downtime migration

---

### 3. Managed service
AWS handles:
- Replication engine
- Scaling
- Monitoring

---

### 4. Monitoring tools
Integrated with:
- CloudWatch metrics
- Task logs
- Error tracking

---

### 5. Schema conversion (AWS SCT)
Often used with:
- AWS Schema Conversion Tool (SCT)

👉 Helps convert schema before migration

---

## 1. What AWS DMS enables (possibilities)

### 🔄 1.1 Continuous migration (Full load + CDC)
- Initial full data copy
- Then continuous sync of changes (INSERT/UPDATE/DELETE)

👉 Enables near-zero downtime migrations

---

### 1.2 Cross-version upgrades
Example:
- PostgreSQL 10 → PostgreSQL 13  

👉 Works even for large version jumps

---

### 1.3 Heterogeneous migrations
You can migrate between:
- Oracle → PostgreSQL
- MySQL → PostgreSQL
- PostgreSQL → Aurora PostgreSQL

---

### 1.4 Partial table migration
- Migrate selected schemas or tables only  
- Useful for phased migrations  

---

### 1.5 Ongoing replication use cases
Beyond migration:
- Disaster Recovery (DR)
- Reporting database (analytics copy)
- Data pipelines

---

### 1.6 AWS-managed monitoring
- CloudWatch metrics
- Task logs
- Automatic retries

---

## ⚠️ 2. Problems you may face (real production issues)

### 2.1 Schema is NOT fully migrated
DMS does NOT fully handle:
- Indexes (must be recreated)
- Constraints (partial support)
- Triggers (not always migrated)
- Stored procedures (limited support)

👉 Schema must be created manually

---

### 2.2 Replication lag
If workload is high:
- DMS may lag behind source  
👉 Risk: stale data at cutover

---

### 2.3 Task failures (silent risk)
DMS tasks can fail due to:
- Network issues
- Schema mismatch
- Permission issues  

👉 If not monitored → data loss risk

---

### 2.4 Large initial load time
- Full load for large DBs may take hours or days

---

### 2.5 Data type conversion issues
Possible issues with:
- JSONB edge cases
- Custom types
- Arrays / enums

---

### 2.6 Cutover complexity
Even with DMS:
- Stop writes required
- Wait for final sync
- Switch application

👉 Not truly instant

---

### 2.7 Cost overhead
Costs include:
- Replication instance (running continuously)
- Data transfer charges

---

## 🚨 3. Risks (what can go wrong badly)

 ### Risk 1: Silent data drift  
- Replication stops unnoticed  
👉 Target DB becomes incomplete  

---

 ### Risk 2: Inconsistent data at cutover  
- Lagging transactions  
👉 Partial data migration  

---

 ### Risk 3: Duplicate key failures  
- Sequence mismatch  
👉 Application crashes after switch  

---

 ### Risk 4: Schema mismatch corruption  
- Incorrect schema alignment  
👉 Logical data corruption  

---

 ### Risk 5: Wrong cutover timing  
- Too early → missing data  
- Too late → increased lag risk  

---

 ### Risk 6: Application misrouting  
- App still writes to old DB after cutover  

---

## 🛡️ 4. What you MUST take care of

### 4.1 Schema pre-build (mandatory)
Before DMS:
- Create tables manually
- Create indexes
- Align data types

---

### 4.2 Enable correct CDC settings
Ensure source DB:
- Logical replication enabled
- Proper WAL configuration

---

### 4.3 Monitor continuously
Use:
- CloudWatch metrics
- DMS task logs

Monitor:
- Latency
- Errors
- Task status

---

### 4.4 Handle sequences manually
After migration:

```sql
SELECT setval('seq_name', (SELECT MAX(id) FROM table));
