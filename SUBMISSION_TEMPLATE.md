POWERPUFF GIRLS
Incident ID: RR-INC-05

1. Incident Summary

The inventory API experienced a sudden increase in failed requests and response latency. Users received HTTP 503 errors when attempting to retrieve inventory information. The P95 latency increased significantly, and the database connection pool became completely exhausted, preventing new requests from acquiring connections.

2. Root Cause

The root cause was a connection leak in app/inventory.py, specifically in the lookup_inventory() function.

The function acquired a database connection using pool.acquire(), but released it only after a successful database query.

When simulate_timeout=True, a DatabaseTimeout exception was raised before pool.release(connection) could execute.

Consequently, connections remained active even after failed requests. Repeated database timeouts gradually exhausted the connection pool, causing subsequent requests to fail with pool exhaustion errors and HTTP 503 responses.

3. Evidence

Metrics: At 14:25 UTC, the error rate was 1.2%, P95 latency was 175 ms, and 4 out of 10 connections were active. By 14:40 UTC, the error rate increased to 36.2%, P95 latency reached 5100 ms, and all 10 connections were active.

Application logs: Requests req-601, req-602, and req-603 acquired database connections and encountered database query timeouts. No corresponding pool_release events appeared after these failures.

Alerts: The DatabasePoolSaturation alert reported 10/10 active connections (100%). The high error rate alert reported 36.2%, and the high latency alert reported 5100 ms.

System behaviour: Request traffic remained relatively stable at approximately 140 requests per minute, while CPU utilization remained around 30%.

4. Corrective Action

Modified app/inventory.py to use a try...finally block inside lookup_inventory().

The database query is executed inside the try block, while pool.release(connection) is placed inside the finally block.

This ensures that the acquired connection is released whether the query succeeds or raises a DatabaseTimeout exception. The fix prevents connection leaks and allows the service to recover after repeated failures.

5. Validation

Unit test results:

Ran 6 tests in 0.000s

OK

All six tests passed.

Recovery verifier output:

RECOVERY VERIFIED

Repeated database timeouts did not leak connections.

A normal inventory request succeeded after the timeouts.

6. Final Flag

FLAG{POOL_RELEASE_RECOVERED}
