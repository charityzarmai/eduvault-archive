# Testing Guide for Three Issues Implementation

## Prerequisites

Before running tests, ensure dependencies are installed:

```bash
npm install
```

Required dependencies (already in `package.json`):
- `mongodb` - Database client
- `mongodb-memory-server` - In-memory MongoDB for tests
- `vitest` or Node.js built-in test runner

## Running Tests

### All Backend Tests
```bash
npm run test:backend
```

### Individual Test Suites

**Pagination Stability Tests**:
```bash
node --test tests/backend/pagination-stability.test.mjs
```

Expected output:
```
✔ should not duplicate records when new records are inserted mid-pagination
✔ should not skip records when records are deleted mid-pagination
✔ should maintain deterministic ordering with identical sort values
✔ should consistently exclude hidden, deleted, and suspended records
✔ should handle cursor pagination across different sort patterns
ℹ tests 5
ℹ pass 5
ℹ fail 0
```

**Data Retention Cleanup Tests**:
```bash
node --test tests/backend/data-retention-cleanup.test.mjs
```

Expected output:
```
✔ should never delete permanently protected collections
✔ should identify eligible records outside retention window
✔ should protect unread notifications from cleanup
✔ should protect dead letter events with unresolved status
✔ should protect material history while linked material exists
✔ should protect active entitlements from cleanup
✔ should protect pending outbox intents from cleanup
✔ should correctly identify eligible sync events for cleanup
✔ should handle collections that do not exist gracefully
✔ should report summary statistics correctly
ℹ tests 10
ℹ pass 10
ℹ fail 0
```

**Restore Integrity Tests**:
```bash
node --test tests/backend/restore-integrity.test.mjs
```

Expected output:
```
✔ should detect purchase referencing non-existent material
✔ should detect entitlement cache referencing non-existent material
✔ should detect refund referencing non-existent purchase
✔ should detect duplicate wallet addresses
✔ should detect duplicate UUIDs
✔ should detect purchase without buyer address
✔ should detect material without creator address
✔ should detect protected material without storage key
✔ should detect active entitlement without completed purchase
✔ should detect completed purchase without transaction hash
✔ should pass validation for complete valid dataset
ℹ tests 14
ℹ pass 14
ℹ fail 0
```

## Manual Testing

### Testing Pagination Stability

1. Start the development server:
   ```bash
   npm run dev
   ```

2. Open the marketplace and load page 1

3. In a separate terminal, add new materials:
   ```bash
   # This would require a seed script or manual database insert
   ```

4. Navigate to page 2 and verify no duplicates appear

5. Test with different sort modes:
   - Newest
   - Price ascending
   - Price descending
   - Rating
   - Popular

### Testing Data Retention Cleanup

1. Run dry-run mode:
   ```bash
   npm run clean:retention:dry
   ```

2. Review the output report:
   - Check "Total Eligible Records" count
   - Review sample records to be deleted
   - Verify protected collections are skipped

3. If satisfied, execute cleanup:
   ```bash
   npm run clean:retention
   ```

4. Verify logs show successful cleanup with no errors

### Testing Restore Validation

1. Create a test database backup:
   ```bash
   mongodump --uri="$MONGODB_URI" --gzip --archive=test-backup.gz
   ```

2. Restore to staging environment:
   ```bash
   mongorestore --uri="$STAGING_URI" --drop --gzip --archive=test-backup.gz
   ```

3. Run validation:
   ```bash
   MONGODB_URI="$STAGING_URI" npm run validate:restore
   ```

4. Expected output for healthy database:
   ```
   ✅ ALL INTEGRITY CHECKS PASSED
   Database integrity validation successful.
   ```

5. To test violation detection, manually corrupt data:
   ```javascript
   // In MongoDB shell
   db.purchases.insertOne({
     buyerAddress: "0xtest",
     materialId: "non-existent-id",
     status: "completed",
     createdAt: new Date()
   })
   ```

6. Re-run validation and verify it detects the violation

## Continuous Integration

### GitHub Actions / GitLab CI

Add to your CI pipeline:

```yaml
- name: Run Backend Tests
  run: npm run test:backend

- name: Run Pagination Tests
  run: node --test tests/backend/pagination-stability.test.mjs

- name: Run Retention Tests
  run: node --test tests/backend/data-retention-cleanup.test.mjs

- name: Run Integrity Tests
  run: node --test tests/backend/restore-integrity.test.mjs
```

### Pre-commit Hook

Add to `.git/hooks/pre-commit`:

```bash
#!/bin/bash
npm run test:backend
if [ $? -ne 0 ]; then
  echo "Backend tests failed. Commit aborted."
  exit 1
fi
```

## Troubleshooting

### Test Failures

**"Cannot find package 'mongodb'"**:
```bash
npm install
```

**"MongoDB Memory Server download failed"**:
```bash
# Set custom download URL
export MONGOMS_DOWNLOAD_URL="https://fastdl.mongodb.org/..."
npm test
```

**"Connection timeout"**:
- Increase timeout in test setup
- Check network connectivity
- Verify MongoDB Memory Server is running

### Script Execution Issues

**"Permission denied"**:
```bash
chmod +x scripts/*.mjs
```

**"Module not found"**:
- Verify Node.js version (>=18 required)
- Check `package.json` has `"type": "module"`
- Ensure imports use `.js` or `.mjs` extensions

### Database Issues

**"Collection not found"**:
- Run database setup: `npm run db:indexes`
- Seed test data: `npm run seed:local`

**"Invalid connection string"**:
- Verify `MONGODB_URI` environment variable
- Check connection string format
- Ensure database is running

## Performance Testing

### Pagination Performance Benchmark

Run existing benchmark:
```bash
npm run benchmark:marketplace
```

Expected results:
- Cursor pagination @ page 50: ~5ms
- Offset pagination @ page 50: ~180ms
- Improvement: ~36x faster

### Retention Cleanup Performance

Monitor cleanup duration:
```bash
time npm run clean:retention
```

Expected duration: < 5 minutes for normal collection sizes

### Validation Performance

Monitor validation duration:
```bash
time npm run validate:restore
```

Expected duration: < 2 minutes for normal database size

## Test Data Setup

### Minimal Test Data

```javascript
// Create minimal valid dataset
const material = {
  _id: new ObjectId(),
  title: "Test Material",
  userAddress: "0xcreator",
  visibility: "public",
  price: 10,
  storageKey: "QmTestHash",
  createdAt: new Date()
};

const purchase = {
  _id: new ObjectId(),
  buyerAddress: "0xbuyer",
  materialId: material._id.toString(),
  status: "completed",
  transactionHash: "0xtxhash",
  createdAt: new Date()
};
```

### Large Dataset for Performance Testing

Use seed script:
```bash
npm run seed:local
```

Or create custom dataset:
```javascript
// Create 1000 materials for pagination testing
const materials = Array.from({ length: 1000 }, (_, i) => ({
  _id: new ObjectId(),
  title: `Material ${i}`,
  userAddress: "0xcreator",
  visibility: "public",
  price: 10 + (i % 100),
  createdAt: new Date(Date.now() - i * 60000),
  archived: false,
  moderationStatus: "approved",
  isDeleted: false,
  creatorSuspended: false
}));

await db.collection("material_search_documents").insertMany(materials);
```

## Coverage Reports

### Generate Coverage

Using Vitest:
```bash
npm run test -- --coverage
```

Using c8 with Node test runner:
```bash
npx c8 node --test tests/backend/*.test.mjs
```

### View Coverage Report

```bash
open coverage/index.html
```

Target coverage:
- Statements: > 80%
- Branches: > 75%
- Functions: > 80%
- Lines: > 80%

## Debugging Tests

### Enable Debug Logging

```bash
DEBUG=* node --test tests/backend/pagination-stability.test.mjs
```

### Inspect Test Database

After test failure, database state is preserved in MongoDB Memory Server. Connect and inspect:

```javascript
// In test file, add before cleanup
await new Promise(resolve => setTimeout(resolve, 60000)); // Wait 60s
```

Then connect:
```bash
mongo mongodb://127.0.0.1:<port>/test
```

### VSCode Debugging

Add to `.vscode/launch.json`:
```json
{
  "type": "node",
  "request": "launch",
  "name": "Debug Backend Tests",
  "skipFiles": ["<node_internals>/**"],
  "program": "${workspaceFolder}/node_modules/.bin/node",
  "args": [
    "--test",
    "${workspaceFolder}/tests/backend/pagination-stability.test.mjs"
  ]
}
```

## Test Maintenance

### Adding New Tests

1. Create test file in `tests/backend/`
2. Follow naming convention: `feature-name.test.mjs`
3. Use MongoDB Memory Server for isolation
4. Clean up data in `beforeEach` hook
5. Add descriptive test names
6. Include both positive and negative cases

### Updating Existing Tests

1. Run tests before changes: `npm run test:backend`
2. Make test updates
3. Verify all tests still pass
4. Update documentation if needed
5. Commit with descriptive message

## Next Steps

After verifying tests pass:

1. ✅ Review test output
2. ✅ Check test coverage reports
3. ✅ Run manual testing scenarios
4. ✅ Deploy to staging
5. ✅ Run tests in staging environment
6. ✅ Monitor for 24 hours
7. ✅ Deploy to production

---

**Questions or issues?** Open a GitHub issue or contact the development team.
