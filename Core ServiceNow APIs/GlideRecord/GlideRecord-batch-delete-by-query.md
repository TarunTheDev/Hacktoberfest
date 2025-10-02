# GlideRecord: Batch Delete by Query (with safety checks)

**Type:** Background Script (Server-side)  
**Platform:** ServiceNow  
**Summary:** Deletes records matching a query in **batches** to avoid long transactions, with a dry-run option and table allow-list.

> ⚠️ Use carefully in non-prod first. Set `DRY_RUN = true` to preview counts.

## Script

```js
// Background Script: Batch delete records that match a query.
// Safer than mass delete in one transaction; runs in chunks and logs progress.

// ======= CONFIG =======
var TABLE_NAME  = 'incident';       // Target table
var QUERY       = 'state=7^sys_created_onRELATIVELE@month@ago@1'; // Your condition
var BATCH_SIZE  = 500;              // Delete N records per loop
var DRY_RUN     = true;             // true = don't delete, just report
var ALLOW_LIST  = ['incident','sc_task','problem','change_request']; // Limit which tables are allowed
// ======================

(function runBatchDelete() {
  if (ALLOW_LIST.indexOf(TABLE_NAME) === -1) {
    gs.error('[BATCH-DELETE] Table "{0}" not in ALLOW_LIST. Aborting.', TABLE_NAME);
    return;
  }

  var total = countMatching(TABLE_NAME, QUERY);
  gs.info('[BATCH-DELETE] Table: {0} | Query: {1} | Matches: {2} | Batch: {3} | DryRun: {4}',
          TABLE_NAME, QUERY, total, BATCH_SIZE, DRY_RUN);

  if (total === 0) return;

  var deleted = 0, loops = 0;

  while (deleted < total) {
    var gr = new GlideRecord(TABLE_NAME);
    gr.addEncodedQuery(QUERY);
    gr.setLimit(BATCH_SIZE);
    gr.orderBy('sys_id'); // stable iteration
    gr.query();

    var ids = [];
    while (gr.next()) ids.push(String(gr.getUniqueValue()));

    if (ids.length === 0) break;

    if (DRY_RUN) {
      gs.info('[BATCH-DELETE] DryRun batch preview: {0} records (sample sys_id: {1})',
              ids.length, ids[0]);
    } else {
      // Delete by sys_id list to keep batches precise
      var del = new GlideRecord(TABLE_NAME);
      del.addQuery('sys_id', 'IN', ids.join(','));
      del.query();
      var batchCount = 0;
      gs.info('[BATCH-DELETE] Deleting batch of {0} records...', ids.length);
      while (del.next()) { del.deleteRecord(); batchCount++; }
      deleted += batchCount;
      gs.info('[BATCH-DELETE] Deleted so far: {0}/{1}', deleted, total);
    }

    loops++;
    if (loops > 10000) { // hard stop guard
      gs.error('[BATCH-DELETE] Too many loops; aborting as a safety measure.');
      break;
    }
  }

  gs.info('[BATCH-DELETE] Finished. Total matched: {0}. {1}: {2}.',
          total, DRY_RUN ? 'Would delete' : 'Deleted', DRY_RUN ? total : deleted);

  // ---- helpers ----
  function countMatching(table, encoded) {
    var c = new GlideAggregate(table);
    c.addEncodedQuery(encoded);
    c.addAggregate('COUNT');
    c.query();
    return c.next() ? parseInt(c.getAggregate('COUNT'), 10) : 0;
  }
})();
