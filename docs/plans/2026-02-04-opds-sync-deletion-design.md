# OPDS Sync with Optional Deletion Design

## Overview

Add per-catalog options to the OPDS plugin for automatic cleanup of books that are no longer in the catalog feed during sync operations. This feature is opt-in and backwards compatible with existing sync behavior.

## Problem Statement

Currently, OPDS sync only downloads new books from catalogs. Books that are removed from a catalog's feed remain on the device indefinitely. Users who want to maintain an exact mirror of their OPDS catalog have no automated way to remove books that are no longer available.

## Goals

1. Add per-catalog option to enable subdirectories for better organization
2. Add per-catalog option to delete books not in the catalog feed
3. Maintain backwards compatibility - no behavior changes for existing users
4. Ensure safe deletion with proper fallbacks and tracking
5. Support both shared sync directory and per-catalog subdirectory modes

## Non-Goals

- Automatic deletion without explicit user opt-in
- Deletion of manually added books or books from other sources
- Sync conflict resolution (last-write-wins, etc.)

## Design

### 1. Data Structures

#### Per-Catalog Settings

Add two new optional boolean fields to each server object in `opds.lua` settings:

```lua
{
    title = "Catalog Name",
    url = "https://...",
    username = "...",
    password = "...",
    sync = true/false,                    -- existing
    use_subdirectory = true/false,        -- NEW: create catalog subfolder
    delete_missing = true/false,          -- NEW: delete books not in feed
    raw_names = true/false,               -- existing
    last_download = "...",                -- existing
}
```

**Default values**: Both new fields default to `false` for safety and backwards compatibility.

#### Download Tracking Enhancement

Extend existing download/pending_syncs structure to include stable book identifiers:

```lua
{
    file = "/path/to/book.epub",
    url = "http://book/download/url",
    catalog = "http://catalog.url",
    username = "...",
    password = "...",
    book_id = "unique_book_identifier",   -- NEW: from feed entry.id or URL hash
}
```

#### Persistent Catalog File Tracking

New settings file: `opds_sync_tracking.lua`

```lua
{
    catalog_files = {
        ["http://catalog.url"] = {
            ["/path/to/book1.epub"] = {
                book_id = "id_from_feed",
                download_time = 1234567890,
                url = "http://book/download/url"
            },
            ["/path/to/book2.epub"] = {
                book_id = "another_id",
                download_time = 1234567900,
                url = "http://another/book/url"
            },
        },
        ["http://another-catalog.url"] = { ... },
    }
}
```

This file is created and maintained automatically during sync operations.

### 2. User Interface Changes

#### Catalog Edit Dialog

Modify `OPDSBrowser:addEditCatalog()` in `opdsbrowser.lua` to add two new checkboxes below the existing "Sync catalog" checkbox:

```lua
check_button_sync_catalog = CheckButton:new{
    text = _("Sync catalog"),
    checked = item and item.sync,
    parent = dialog,
}

check_button_use_subdirectory = CheckButton:new{
    text = _("Use subdirectory for this catalog"),
    checked = item and item.use_subdirectory,
    parent = dialog,
    enabled = check_button_sync_catalog.checked,
}

check_button_delete_missing = CheckButton:new{
    text = _("Delete books not in catalog"),
    checked = item and item.delete_missing,
    parent = dialog,
    enabled = check_button_sync_catalog.checked,
}
```

**Visual Layout**:
```
☑ Sync catalog
  ☐ Use subdirectory for this catalog
  ☐ Delete books not in catalog
```

**Behavior**:
- Subdirectory and delete options are only enabled when "Sync catalog" is checked
- Both options can be toggled independently
- Settings are saved per-catalog in the server configuration

### 3. Sync Logic Implementation

#### Download Path Determination

Modify `OPDSBrowser:getCurrentDownloadDir()` to support subdirectories:

```lua
function OPDSBrowser:getCurrentDownloadDir()
    if self.sync then
        local base_dir = self.settings.sync_dir
        if self.sync_server.use_subdirectory then
            -- Create safe subdirectory name from catalog title
            local subdir = util.replaceAllInvalidChars(self.root_catalog_title)
            local full_path = base_dir .. "/" .. subdir
            util.makePath(full_path)  -- ensure directory exists
            return full_path
        end
        return base_dir
    else
        return G_reader_settings:readSetting("download_dir") or G_reader_settings:readSetting("lastdir")
    end
end
```

#### Metadata Tracking Functions

Add new functions to `OPDSBrowser`:

```lua
-- Load catalog file tracking from persistent storage
function OPDSBrowser:loadCatalogFileTracking()
    local tracking_file = DataStorage:getSettingsDir() .. "/opds_sync_tracking.lua"
    local settings = LuaSettings:open(tracking_file)
    self.catalog_files = settings:readSetting("catalog_files") or {}
end

-- Save catalog file tracking to persistent storage
function OPDSBrowser:saveCatalogFileTracking()
    local tracking_file = DataStorage:getSettingsDir() .. "/opds_sync_tracking.lua"
    local settings = LuaSettings:open(tracking_file)
    settings:saveSetting("catalog_files", self.catalog_files)
    settings:flush()
end

-- Track a downloaded file with its catalog and book ID
function OPDSBrowser:trackDownloadedFile(catalog_url, file_path, book_id, book_url)
    if not self.catalog_files[catalog_url] then
        self.catalog_files[catalog_url] = {}
    end
    self.catalog_files[catalog_url][file_path] = {
        book_id = book_id,
        download_time = os.time(),
        url = book_url,
    }
    self:saveCatalogFileTracking()
end
```

#### Deletion Logic with Two Strategies

Add new function `OPDSBrowser:cleanupMissingBooks()` with dual strategy support:

**Strategy 1: Subdirectory Mode** (when `use_subdirectory = true`)
- More straightforward and thorough
- Scan all files in the catalog's subdirectory
- Delete any file not present in the current feed
- No metadata requirements since subdirectory boundary is clear

**Strategy 2: Shared Directory Mode** (when `use_subdirectory = false`)
- Conservative and metadata-dependent
- Only delete files we have explicit tracking data for
- Verify book_id from tracking matches feed absence
- Never delete if tracking data is missing (safe fallback)

```lua
function OPDSBrowser:cleanupMissingBooks(catalog_url, current_feed_books)
    if not self.sync_server.delete_missing then
        return
    end

    local sync_dir = self:getCurrentDownloadDir()

    if self.sync_server.use_subdirectory then
        -- SUBDIRECTORY MODE: Delete files in subdirectory not in feed
        local feed_files = {}
        for _, book in ipairs(current_feed_books) do
            if book.file_path then
                feed_files[book.file_path] = true
            end
        end

        local files_to_delete = {}
        for entry in lfs.dir(sync_dir) do
            if entry ~= "." and entry ~= ".." then
                local file_path = sync_dir .. "/" .. entry
                local attr = lfs.attributes(file_path)
                if attr and attr.mode == "file" and not feed_files[file_path] then
                    table.insert(files_to_delete, file_path)
                end
            end
        end

        for _, file_path in ipairs(files_to_delete) do
            logger.info("Deleting missing book from subdirectory:", file_path)
            os.remove(file_path)
        end

        if #files_to_delete > 0 then
            UIManager:show(InfoMessage:new{
                text = T(N_("1 book removed (no longer in catalog)",
                           "%1 books removed (no longer in catalog)",
                           #files_to_delete), #files_to_delete),
                timeout = 3,
            })
        end

    else
        -- SHARED DIRECTORY MODE: Only delete tracked files not in feed
        if not self.catalog_files or not self.catalog_files[catalog_url] then
            logger.info("No tracking data for catalog, skipping cleanup (safe fallback)")
            return
        end

        local feed_book_ids = {}
        for _, book in ipairs(current_feed_books) do
            if book.book_id then
                feed_book_ids[book.book_id] = true
            end
        end

        local files_to_delete = {}
        for file_path, metadata in pairs(self.catalog_files[catalog_url]) do
            if metadata.book_id and not feed_book_ids[metadata.book_id] then
                if lfs.attributes(file_path) then
                    table.insert(files_to_delete, file_path)
                end
            end
        end

        for _, file_path in ipairs(files_to_delete) do
            logger.info("Deleting missing book:", file_path)
            os.remove(file_path)
            self.catalog_files[catalog_url][file_path] = nil
        end

        if #files_to_delete > 0 then
            self:saveCatalogFileTracking()
            UIManager:show(InfoMessage:new{
                text = T(N_("1 book removed (no longer in catalog)",
                           "%1 books removed (no longer in catalog)",
                           #files_to_delete), #files_to_delete),
                timeout = 3,
            })
        end
    end
end
```

### 4. Integration into Sync Flow

#### Modify fillPendingSyncs

Update `OPDSBrowser:fillPendingSyncs()` to:
1. Load catalog file tracking at the start
2. Extract book IDs from feed entries
3. Track current feed books for comparison
4. Call cleanup after building download list

```lua
function OPDSBrowser:fillPendingSyncs(server)
    -- ... existing setup code ...

    self:loadCatalogFileTracking()  -- NEW

    local sync_list = self:getSyncDownloadList()
    if sync_list then
        local current_feed_books = {}  -- NEW

        for i, entry in ipairs(sync_list) do
            -- ... existing entry processing ...

            for j, link in ipairs(item.acquisitions) do
                -- ... existing filetype checking ...

                if filetype then
                    if not file_str or file_list and file_list[filetype] then
                        local filename = self:getFileName(entry)
                        local download_path = self:getLocalDownloadPath(filename, filetype, link.href)

                        -- NEW: Extract book ID
                        local book_id = entry.id or util.getURLHash(link.href)

                        if dl_count <= self.sync_max_dl then
                            table.insert(self.pending_syncs, {
                                file = download_path,
                                url = link.href,
                                username = self.root_catalog_username,
                                password = self.root_catalog_password,
                                catalog = server.url,
                                book_id = book_id,  -- NEW
                            })
                            dl_count = dl_count + 1
                        end

                        -- NEW: Track as part of current feed
                        table.insert(current_feed_books, {
                            book_id = book_id,
                            file_path = download_path,
                        })

                        break
                    end
                end
            end
        end

        -- NEW: Cleanup missing books after building download list
        self:cleanupMissingBooks(server.url, current_feed_books)
    end

    -- ... existing code ...
end
```

#### Modify downloadFile

Update `OPDSBrowser:downloadFile()` to track successful downloads:

```lua
function OPDSBrowser:downloadFile(local_path, remote_url, username, password, caller_callback)
    -- ... existing download code ...

    if code == 200 then
        logger.dbg("File downloaded to", local_path)

        -- NEW: Track this download if in sync mode with metadata
        if self.sync and self.current_book_metadata then
            self:trackDownloadedFile(
                self.current_book_metadata.catalog,
                local_path,
                self.current_book_metadata.book_id,
                remote_url
            )
        end

        if caller_callback then
            caller_callback(local_path)
        end
        return true
    end

    -- ... existing error handling ...
end
```

#### Modify downloadPendingSyncs

Update `OPDSBrowser:downloadPendingSyncs()` to pass book metadata to download function:

```lua
-- Inside the download loop, before calling downloadFile:
self.current_book_metadata = {
    catalog = item.catalog,
    book_id = item.book_id,
}
self:downloadFile(item.file, item.url, item.username, item.password)
self.current_book_metadata = nil
```

### 5. Initialization & Migration

#### Plugin Initialization

Modify `OPDS:init()` in `main.lua` to migrate existing catalogs:

```lua
function OPDS:init()
    self.opds_settings = LuaSettings:open(self.opds_settings_file)
    if next(self.opds_settings.data) == nil then
        self.updated = true
    end
    self.servers = self.opds_settings:readSetting("servers", self.default_servers)
    self.downloads = self.opds_settings:readSetting("downloads", {})
    self.settings = self.opds_settings:readSetting("settings", {})
    self.pending_syncs = self.opds_settings:readSetting("pending_syncs", {})

    -- NEW: Migrate existing servers with safe defaults
    for _, server in ipairs(self.servers) do
        if server.use_subdirectory == nil then
            server.use_subdirectory = false
        end
        if server.delete_missing == nil then
            server.delete_missing = false
        end
    end

    self:onDispatcherRegisterActions()
    self.ui.menu:registerToMainMenu(self)
end
```

#### Browser Initialization

Update `OPDSBrowser:init()` to initialize tracking:

```lua
function OPDSBrowser:init()
    -- ... existing code ...

    -- NEW: Initialize catalog file tracking (loaded lazily)
    self.catalog_files = nil
end
```

### Migration Strategy for Existing Users

1. **Safe Defaults**: Existing catalogs automatically get `use_subdirectory = false` and `delete_missing = false`
2. **No Behavior Changes**: Until user explicitly opts in, sync works exactly as before
3. **Gradual Metadata Building**: New downloads start building tracking metadata automatically
4. **Safe Fallback**: If user enables `delete_missing` in shared directory mode before metadata exists, system safely does nothing
5. **No Data Loss**: No files are ever deleted without explicit user opt-in and proper tracking

## Safety Features

### Multiple Layers of Protection

1. **Opt-in by Default**: Both features disabled by default
2. **Per-Catalog Control**: Each catalog can be configured independently
3. **Metadata Fallback**: Shared directory mode requires tracking data before deletion
4. **File Existence Checks**: Only delete files that actually exist
5. **Logging**: All deletions are logged with full file paths
6. **User Notifications**: Users are informed when books are removed
7. **Subdirectory Isolation**: Subdirectory mode provides clear boundaries
8. **Regular File Check**: Only delete regular files, not directories or special files

### Edge Cases Handled

- **No tracking data**: Skip deletion (safe fallback)
- **Missing book_id**: Skip that file in deletion check
- **File already deleted**: Skip gracefully
- **Incomplete downloads**: Already handled by existing code
- **Sync interrupted**: Metadata saved incrementally
- **Catalog URL changed**: Treated as new catalog (no deletions from old URL)

## Testing Recommendations

### Manual Testing Scenarios

1. **New catalog with subdirectories**
   - Enable sync + use_subdirectory + delete_missing
   - Verify files go to subdirectory
   - Remove items from feed, verify deletion on next sync

2. **Existing catalog migration**
   - Open existing synced catalog config
   - Verify both checkboxes default to unchecked
   - Verify sync behavior unchanged

3. **Shared directory mode**
   - Enable sync + delete_missing (without subdirectory)
   - Download some books
   - Verify tracking file created
   - Remove items from feed, verify deletion on next sync

4. **Mixed catalog setup**
   - Catalog A: subdirectory + deletion
   - Catalog B: shared directory + deletion
   - Catalog C: shared directory, no deletion
   - Verify each behaves independently

5. **Safety fallbacks**
   - Enable delete_missing without prior tracking data
   - Verify no deletions occur (safe fallback)
   - Sync once to build tracking
   - Verify deletions work on second sync

### Unit Testing Considerations

- `getCurrentDownloadDir()` with and without subdirectories
- `trackDownloadedFile()` creates proper data structures
- `cleanupMissingBooks()` in both modes (subdirectory and shared)
- Book ID extraction from feed entries
- Migration code for existing servers

## Files to Modify

- `plugins/opds.koplugin/main.lua` - Initialization and migration
- `plugins/opds.koplugin/opdsbrowser.lua` - Main implementation
  - UI changes in `addEditCatalog()`
  - New functions for tracking and cleanup
  - Modifications to sync flow functions
  - Download path logic updates

## Files to Create

- `docs/plans/2026-02-04-opds-sync-deletion-design.md` - This document
- (Runtime) `opds_sync_tracking.lua` - Created automatically by plugin

## Implementation Order

1. Add data structure fields and migration code
2. Implement UI changes (checkboxes)
3. Implement subdirectory support
4. Implement metadata tracking functions
5. Implement deletion logic (both strategies)
6. Integrate into sync flow
7. Test thoroughly with various configurations

## Future Enhancements (Out of Scope)

- Sync conflict resolution
- Bandwidth optimization (delta sync)
- Scheduled automatic sync
- Sync status dashboard
- Export/import catalog configurations
- Sync multiple file formats per book
