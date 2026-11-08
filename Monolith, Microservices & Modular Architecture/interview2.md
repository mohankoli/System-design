# System Design: Deep Pagination

## Q. Why Does Page 500 Slow Down the Database, and How Do You Fix It?

**Answer:**

OFFSET pagination becomes slow because the database must process and skip many rows before returning the requested records. It can also cause **page drift**: new inserts or deletes shift page boundaries, so users may see duplicate or missing records.

I would use **keyset (cursor-based) pagination**. The API returns an opaque `next_cursor` containing the last row's `created_at` and `id`. The ID is a unique tie-breaker when multiple orders have the same timestamp.

```sql
SELECT id, created_at, status
FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

For a page size of 20, I fetch **21 rows** to determine `has_more` without a separate COUNT query. I return at most 20 rows and create the next cursor from the last **returned** row.

```sql
CREATE INDEX idx_orders_feed ON orders (created_at DESC, id DESC);
```

For a customer's orders, use an index beginning with `customer_id`, followed by `created_at DESC, id DESC`, and filter by that customer.

**Trade-off:** Cursor pagination is efficient for next/previous navigation but does not naturally support jumping directly to page 47.

## Important Follow-up Questions

### Q1. What if two orders have the same timestamp?

**Answer:** I use `(created_at, id)` for both sorting and the cursor. The unique ID breaks ties and prevents skipping rows with equal timestamps.

### Q2. Can a row still be missed with keyset pagination?

**Answer:** Yes. Late commits or changes to sorting fields can make rows appear behind a cursor already passed. Keyset pagination is not a consistent snapshot. For strict consistency, use an appropriate snapshot or commit-order-aware strategy; a normal sequence alone does not guarantee commit order.

### Q3. How do you verify that the database uses the index?

**Answer:** Run `EXPLAIN (ANALYZE, BUFFERS)` on the actual PostgreSQL query. Check for an efficient index scan, low rows examined, and no expensive large sort. Test with realistic data.

### Q4. Why not just add an index or cache pages?

**Answer:** An index improves sorting and lookups, but a large OFFSET still requires walking skipped entries. Cache helps popular pages, not arbitrary deep pages. Keyset pagination addresses the deep-skip cost.

### Q5. What about total counts and page numbers?

**Answer:** For feeds, return `has_more` and `next_cursor` instead of total pages. If an admin grid requires page numbers, use a separate bounded OFFSET path and consider an approximate or asynchronously calculated count.

### Q6. What if someone edits the cursor?

**Answer:** Base64 is encoding, not encryption. Sign the cursor with HMAC and verify it server-side. Also validate filters, user authorization, and cursor expiry; always use parameterized SQL.

## Quick Revision

| Concept | Remember |
|---|---|
| OFFSET | Deep pages require skipping many rows |
| Page drift | Inserts/deletes shift page boundaries |
| Keyset | Continue after the last `(created_at, id)` |
| Tie-breaker | Unique ID resolves equal timestamps |
| Composite index | Supports filtering and ordered cursor lookup |
| Fetch 21 | Detect next page without COUNT |
| EXPLAIN | Verify actual query plan |
| HMAC | Detect cursor tampering |
