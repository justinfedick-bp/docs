# Partner Engine Documentation

Community-driven documentation for the Partner Engine codebase.

## Q&A Index

- [How does the batch module work?](#how-does-the-batch-module-work)
- [Why is batch context needed? What problems does it solve?](#why-is-batch-context-needed-what-problems-does-it-solve)

---

## How does the batch module work?

The batch module in partner-engine is a sophisticated **in-memory batching system** for safely modifying application forms. Here's how it works:

### Core Architecture

The system uses an **accumulator pattern** to batch all changes in memory before persisting them in a single database transaction.

**Main flow:**
```ruby
# 1. Load context with lock
context = Batch.find_and_lock!(app_form_id)

# 2. Modify in memory (no DB hits)
context.changeset.find_answer!(question_code: :name)
  .apply(safe_value: 'John')

# 3. Save all changes at once
context.save!  # Single transaction
```

### Key Components

**`AppFormContext`** (`app/lib/batch/app_form_context.rb:1`) - Main API interface
- Entry point for all batch operations
- Holds the changeset
- Manages save lifecycle

**`Changeset`** (`app/lib/batch/changeset.rb:1`) - Core container that batches:
- Answers, answer pools, question sets (AFQS)
- Quote requests, messages, events
- Deferred actions (webhooks, broadcasts, indexing)

**Accumulators** - Track what changed:
- `SnapshotAccumulator` (`app/lib/batch/internal/snapshot_accumulator.rb:1`) - For complex items with validation
- `SimpleAccumulator` (`app/lib/batch/internal/simple_accumulator.rb:1`) - For basic sets

**Mutators** - Wrap database models:
- `Answer::Mutator` (`app/lib/batch/answer/mutator.rb:1`)
- `Afqs::Mutator` (`app/lib/batch/afqs/mutator.rb:1`)
- Track changes via before/after snapshots

### How Batching Works

1. **Lock** - Redis lock prevents concurrent updates (`app/lib/batch.rb:55`)
2. **Load** - Fetch app form + JSON document into changeset
3. **Mutate** - All changes accumulate in memory (zero DB overhead)
4. **Snapshot** - Before/after snapshots detect what changed
5. **Save** - Single transaction writes all changes (`app/lib/batch/legacy/save_changes.rb:1`)
6. **Actions** - Deferred actions execute (broadcast, index, webhooks) (`app/lib/batch/actions/`)

### Module Structure

```
batch/
├── batch.rb                          # Main module with factory methods
├── app_form_context.rb               # Primary context interface
├── changeset.rb                      # Core changeset class
│
├── changeset/                        # Changeset builders & utilities
│   ├── app_form_builder.rb           # Builds changesets from app forms
│   ├── fresh_copy_builder.rb         # Builds changesets for copies
│   ├── before_save_callback.rb       # Pre-save validation
│   ├── on_create_callback.rb         # Creation callbacks
│   ├── mutator.rb                    # State mutations
│   ├── changes.rb                    # Change tracking
│   └── memo.rb                       # Long-lived state container
│
├── internal/                         # Core batching infrastructure (37 files)
│   ├── base_mutator.rb               # Base class for all mutators
│   ├── base_snapshot.rb              # Base class for snapshots
│   ├── simple_accumulator.rb         # Basic accumulator pattern
│   ├── snapshot_accumulator.rb       # Advanced accumulator with snapshots
│   ├── copy.rb                       # Represents a single app form copy
│   └── deferred_actions.rb           # Action queuing system
│
├── answer/                           # Answer batch operations
│   ├── accumulator.rb                # Answer changes accumulation
│   ├── mutator.rb                    # Answer mutations
│   ├── snapshot.rb                   # Answer state snapshots
│   └── validate.rb                   # Answer validation
│
├── afqs/                             # Answer Form Question Set operations
├── answer_pool/                      # Answer pool operations
├── quote_request/                    # Quote request handling
├── app_form/                         # App form operations
├── app_form_process/                 # Background process handling
│
└── actions/                          # Deferred side-effect actions (29 files)
    ├── deferred_action.rb            # Base action class
    ├── broadcast_answer.rb           # Broadcast answer changes
    ├── index_app_form.rb             # Index for search
    ├── publish_to_hookshot.rb        # Webhook publishing
    └── submit_carrier_quote_requests.rb
```

### Entry Points

**`Batch.find(id, scope:, archived:)`** - Load an app form context without locking

**`Batch.find_and_lock(id, scope:, raise_on_failure:)`** - Load with non-blocking lock (~30s timeout)

**`Batch.find_and_lock!(id, scope:)`** - Load with blocking lock, raise on failure

**`Batch.find_changeset(id, scope:)`** - Fetch a read-only changeset

**`Batch.build_context(...)`** - Create a new app form context from scratch

### Architectural Patterns

#### Pattern 1: Wrapper/Mutator Pattern
Database models wrapped in mutators that track changes via snapshots. Changes queued in accumulators. Original models never directly mutated.

```ruby
# Database model stays pure
answer_model = Answer.find(1)

# Wrapped in mutator for this changeset
answer_mutator = changeset.find_answer!(id: 1)
answer_mutator.apply(value: 'new')  # In-memory only

# After save, database is updated
context.save!

# Old references are stale, need refetch
answer_mutator = changeset.find_answer!(id: 1)
```

#### Pattern 2: Snapshot-Based Change Detection
Each object generates a snapshot of its state. Before/after snapshots compared to detect changes. Only changed items included in database updates.

#### Pattern 3: Accumulator/Collector Pattern
Changes accumulate in memory during request:
- `add()` - Process single item changes
- `finalize()` - Calculate all final changes
- `post_process()` - Fire callbacks/actions after save

#### Pattern 4: Deferred Actions
Side effects queued but not executed until after database is confirmed updated. Prevents stale data from being broadcast.

#### Pattern 5: Redis Locking
Exclusive locks prevent concurrent updates. 60-second lock duration with blocking wait option. Prevents data overwriting races.

### Data Storage

**Hybrid storage approach:**

1. **JSON Document** (in Copy model) - Answers, answer pools, AFQS, messages, events
2. **Database Records** - Quote requests, application form processes, relationships
3. **Database Tables** - ApplicationForm, Copy, CopyIndex, supporting tables

### Key Benefits

- **Single round-trip**: All changes persist in one transaction
- **Race-free**: Redis locks prevent conflicts
- **Efficient**: Only changed items update via snapshot diffs
- **Safe**: Validation runs before save, actions run after
- **Flexible**: 29 deferred action types for side effects

### Important Notes

The changeset becomes **stale after save** - you must create a new one for subsequent changes. This immutability prevents subtle bugs from concurrent modifications.

**Key behaviors:**
- Only ONE changeset is active at a time
- References become stale after `.save()` - must refetch
- Prevents recursive saves (raises on nested `context.save`)
- Supports deferred actions and callbacks

---

## Why is batch context needed? What problems does it solve?

The Batch Context pattern solves several critical architectural problems that existed in the legacy ActiveRecord-based approach.

### Problems Solved

#### 1. ActiveRecord Callback Hell

From `app/lib/batch/README.md:5`:
> "The context and changeset evolved organically out of an approach that used ActiveRecord callback hooks to evaluate rules and initiate side effects."

The old approach triggered callbacks on every save, leading to:
- Deeply nested, recursive callback chains
- Hard-to-reason-about execution flows
- Unpredictable side effects

**Evidence:** `app/models/application_form.rb:329-335` shows a deprecation warning actively discouraging direct ActiveRecord saves:
```ruby
def after_save_deprecation_message
  return if in_batch_update  # Skip when batch system is saving
  ActiveSupport.deprecator.warn(
    'use context.changeset.application_form.apply(...) instead'
  )
end
```

#### 2. Cascading Rules Engine Actions

When answers change, the rules engine (`app/lib/batch/internal/check_flow.rb`) fires business logic that can:
- Trigger additional features
- Add/remove carrier products
- Opt into/out of carriers
- Create messages and events
- Initiate quote requests

**Without batching:** Each rule action would trigger a separate save → more rules → more saves (infinite recursion risk)

**With batching:** All rule actions accumulate in memory, rules fire once, then everything saves together

#### 3. Race Conditions & Concurrent Modifications

From `app/lib/batch.rb:73-77`:
```ruby
# Previously we could execute the code even if the lock wasn't obtained
# after a while, but with copies we have to abort in order to avoid
# overwriting data from another request that is in flight that holds
# the lock.
```

**Problem:** Multiple requests modifying the same form simultaneously could overwrite each other's changes

**Solution:** Redis locks (`app/lib/batch.rb:46-62`) ensure exclusive access during the entire modification process:
- Lock acquired when context is created
- Held for entire business logic execution (~60 seconds max)
- Released after transaction commits

#### 4. N+1 Database Writes

Creating an application form touches **9 different tables** across 2 databases:
- `application_forms`
- `copies` (tenant-specific database)
- `copy_indices`
- `payment_methods`
- `messages`
- `events`
- `quote_requests`
- `application_form_dispositions`
- `application_form_processes`

**Without batching:** 9+ separate INSERT/UPDATE statements, each potentially in its own transaction

**With batching:** Single transaction, bulk upserts where possible (`app/lib/batch/legacy/save_changes.rb:79-104`)

#### 5. Transaction Atomicity

All changes must succeed or fail together. If the rules engine determines a carrier should be excluded, but the save fails halfway through, you'd have inconsistent state.

**Solution:** `app/lib/batch/legacy/save_changes.rb:106-111`:
```ruby
TenantDatabase.as_tenant(changeset) do
  ApplicationRecord.transaction { PartnerRecord.transaction(&) }
end
```

Nested transactions across multiple databases ensure all-or-nothing persistence.

#### 6. Stale Object Prevention

From the README:
> "Unlike ActiveRecord, where you can hold on to an answer even after saving, when `context.save` is called, any references to answers, AFQS's, quote requests, etc., will become stale and will need to be refetched from the latest changeset."

**This is intentional.** After a save:
- Rules may have modified data
- Holding onto old references would give you outdated state
- Forces explicit refetch, preventing bugs from stale data

#### 7. Performance - Single Database Roundtrip

Instead of:
```
Read form → Save answer 1 → Fire rule → Read form → Save answer 2 →
Fire rule → Read form → Save message → ...
```

You get:
```
Read form once → Accumulate all changes in memory → Fire rules once →
Save everything in one transaction
```

### Key Design Principles

From `app/lib/batch/README.md`:
- "There will ideally only be one call to `context.save` per HTTP request"
- "Modifications happen in memory... changes are written to the database in one go"
- "Recursive saves should be avoided... hard to reason about"

### Usage Pattern

**Controller:** `app/controllers/application_forms_controller.rb:62-86`
```ruby
Batch.build_context(...) do |context|
  CreateAppForm.call(...)  # All business logic
  # ... potentially hundreds of changes accumulate ...
  context.finalize.save  # ONE save, all changes persisted
end
```

The entire pattern exists to centralize control, prevent race conditions, optimize database writes, and make the complex rules engine execution predictable and testable.

---

*Last updated: 2026-01-29*
