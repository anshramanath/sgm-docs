# Session 3 Notes — Pipeline Hardening & Network Deep Dive (May 30, 2026)

---

## What Was Learned

**Network requests are the bottleneck, not sleep timers** — Each HTTP request has real latency: DNS, TCP handshake, server processing, response. Typically 200-500ms per request. Sleep is purely additive on top. Removing sleep barely matters when the network takes 0.3-0.5s anyway.
/Users/anshramanath/Desktop/bikershades-documentation/claude_migration_may_30_2.md

**Concurrent requests help — but not on all servers** — Concurrent requests overlap waiting time across multiple requests so you get N× throughput theoretically. But bikershades.com rate-limits on concurrent *connections*, not request *rate*. Even 3 concurrent caused mass 503s. Sequential was fine because only 1 connection at a time.

**The mid-write corruption problem** — Crash-safe checkpointing protects against interrupted fetches. But the write itself is vulnerable. Stopping a process mid-`json.dump()` leaves a partial/invalid JSON file. Fix: always write to a `.tmp` file first, then atomically rename it over the real file with `os.replace()`.

**`os.replace()` is atomic** — The OS renames the temp file to the destination in one indivisible operation. The old file is untouched until the new one is fully written. No intermediate state.

**The VS Code `.tmp` flash** — Every checkpoint write creates `variations.json.tmp` briefly before `os.replace()` renames it. VS Code's file explorer picks it up for a split second. Expected behavior — the faster the writes, the more frequent the flash.

**Background processes require the machine to stay on** — Closing the lid suspends all processes. Battery dying kills them. `nohup` detaches from the terminal but the machine still needs to be awake.

---

## What Was Built

### Atomic write pattern
```python
def save():
    tmp = str(OUTPUT_FILE) + ".tmp"
    with open(tmp, "w") as f:
        json.dump(results, f)
    os.replace(tmp, OUTPUT_FILE)  # atomic — old file untouched until new one is ready
```

### Retry logic with exponential backoff
```python
def fetch_with_retry(var_id, retries=3, backoff=2.0):
    for attempt in range(retries):
        if response.status_code == 503:
            wait = backoff * (2 ** attempt)  # 2s, 4s, 8s
            time.sleep(wait)
            continue
```

### Concurrent fetch (reverted)
Tried `ThreadPoolExecutor` with 10 then 3 workers. Both caused mass 503s. Server rate-limits concurrent connections. Reverted to sequential.

---

## What's Pending Tomorrow

1. Add atomic write + retry logic to the sequential `fetch_variations.py`
2. Re-run from scratch (products folder was wiped)
3. Then: reshape → download → upload → import
