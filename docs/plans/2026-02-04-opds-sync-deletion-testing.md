# OPDS Sync Deletion Feature - Testing Guide

## Test Scenarios

### 1. New Catalog with Subdirectories
1. Add a new OPDS catalog
2. Enable "Sync catalog"
3. Enable "Use subdirectory for this catalog"
4. Enable "Delete books not in catalog"
5. Perform initial sync
6. Verify: Books downloaded to `sync_dir/Catalog Name/`
7. Manually remove items from the feed (or wait for catalog to update)
8. Sync again
9. Verify: Missing books are deleted from subdirectory

### 2. Existing Catalog Migration
1. Open existing synced catalog configuration
2. Verify: Both new checkboxes are unchecked by default
3. Sync catalog
4. Verify: Behavior unchanged (no subdirectory, no deletion)

### 3. Shared Directory Mode
1. Add new catalog
2. Enable "Sync catalog"
3. Enable "Delete books not in catalog" (subdirectory OFF)
4. Perform initial sync
5. Check: `opds_sync_tracking.lua` file created in settings directory
6. Verify: File contains tracking metadata for downloads
7. Remove items from feed
8. Sync again
9. Verify: Only tracked books from this catalog are deleted

### 4. Mixed Catalog Setup
1. Catalog A: subdirectory + deletion enabled
2. Catalog B: shared directory + deletion enabled
3. Catalog C: shared directory, no deletion
4. Download books from all three
5. Remove items from feeds
6. Sync all catalogs
7. Verify: Each catalog behaves independently

### 5. Safety Fallbacks
1. Enable "Delete books not in catalog" on existing catalog (no prior tracking)
2. Sync catalog
3. Verify: No deletions occur (safe fallback)
4. Sync again
5. Verify: Tracking metadata builds up
6. On third sync with missing items, verify deletions work

## Manual Verification Checklist

- [ ] UI checkboxes appear in catalog edit dialog
- [ ] Checkboxes save and load correctly
- [ ] Subdirectory is created when enabled
- [ ] Books download to subdirectory when enabled
- [ ] Tracking file is created in shared mode
- [ ] Deletions work in subdirectory mode
- [ ] Deletions work in shared directory mode
- [ ] No deletions when delete_missing is disabled
- [ ] No deletions without tracking data (shared mode)
- [ ] User notifications show correct counts
- [ ] Existing catalogs work unchanged after update

## Logging

Enable debug logging to see detailed operation:
- Look for "Deleting missing book" messages
- Check "Tracked file" messages for metadata storage
- Verify "No tracking data" messages in fallback scenarios

## Edge Cases

- Empty subdirectories (should remain)
- Malformed tracking data (should skip safely)
- Network errors during sync (should not affect deletion)
- Books manually added to sync directory (should not be deleted in shared mode)
