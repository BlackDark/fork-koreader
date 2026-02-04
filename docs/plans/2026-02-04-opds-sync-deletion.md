# OPDS Sync Deletion Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add per-catalog options for subdirectories and automatic deletion of books no longer in OPDS feeds.

**Architecture:** Two-mode deletion strategy - subdirectory mode (simple file scanning) and shared directory mode (metadata-based tracking). All changes backwards compatible with safe defaults.

**Tech Stack:** Lua, KOReader plugin system, LuaFileSystem (lfs), LuaSettings

---

## Task 1: Add Data Structure Fields and Migration

**Files:**
- Modify: `plugins/opds.koplugin/main.lua:47-58`

**Step 1: Add migration code to OPDS:init()**

In `plugins/opds.koplugin/main.lua`, modify the `init()` function to add migration for new fields:

```lua
function OPDS:init()
    self.opds_settings = LuaSettings:open(self.opds_settings_file)
    if next(self.opds_settings.data) == nil then
        self.updated = true -- first run, force flush
    end
    self.servers = self.opds_settings:readSetting("servers", self.default_servers)
    self.downloads = self.opds_settings:readSetting("downloads", {})
    self.settings = self.opds_settings:readSetting("settings", {})
    self.pending_syncs = self.opds_settings:readSetting("pending_syncs", {})

    -- Migrate existing servers to add new fields with safe defaults
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

**Step 2: Verify the changes**

Read the file to confirm changes are correct:
```bash
grep -A 20 "function OPDS:init" plugins/opds.koplugin/main.lua
```

Expected: Migration code is present after reading settings

**Step 3: Commit**

```bash
git add plugins/opds.koplugin/main.lua
git commit -m "feat(opds): add migration for subdirectory and delete_missing fields

Add automatic migration to populate use_subdirectory and delete_missing
fields on existing catalogs with safe defaults (false).

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Add UI Checkboxes to Catalog Editor

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:272-313`
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:350-375`

**Step 1: Add checkbox widgets in addEditCatalog()**

In `plugins/opds.koplugin/opdsbrowser.lua`, find the `addEditCatalog` function around line 246 and modify the dialog creation section (around line 272):

```lua
local dialog, check_button_raw_names, check_button_sync_catalog, check_button_use_subdirectory, check_button_delete_missing
dialog = MultiInputDialog:new{
    title = title,
    fields = fields,
    buttons = {
        {
            {
                text = _("Cancel"),
                id = "close",
                callback = function()
                    UIManager:close(dialog)
                end,
            },
            {
                text = _("Save"),
                callback = function()
                    local new_fields = dialog:getFields()
                    new_fields[5] = check_button_raw_names.checked or nil
                    new_fields[6] = check_button_sync_catalog.checked or nil
                    new_fields[7] = check_button_use_subdirectory.checked or nil
                    new_fields[8] = check_button_delete_missing.checked or nil
                    self:editCatalogFromInput(new_fields, item)
                    UIManager:close(dialog)
                end,
            },
        },
    },
}
check_button_raw_names = CheckButton:new{
    text = _("Use server filenames"),
    checked = item and item.raw_names,
    parent = dialog,
}
check_button_sync_catalog = CheckButton:new{
    text = _("Sync catalog"),
    checked = item and item.sync,
    parent = dialog,
}
check_button_use_subdirectory = CheckButton:new{
    text = _("Use subdirectory for this catalog"),
    checked = item and item.use_subdirectory,
    parent = dialog,
}
check_button_delete_missing = CheckButton:new{
    text = _("Delete books not in catalog"),
    checked = item and item.delete_missing,
    parent = dialog,
}
dialog:addWidget(check_button_raw_names)
dialog:addWidget(check_button_sync_catalog)
dialog:addWidget(check_button_use_subdirectory)
dialog:addWidget(check_button_delete_missing)
UIManager:show(dialog)
dialog:onShowKeyboard()
```

**Step 2: Update editCatalogFromInput() to handle new fields**

Find the `editCatalogFromInput` function around line 350 and update it:

```lua
function OPDSBrowser:editCatalogFromInput(fields, item, no_refresh)
    local new_server = {
        title     = fields[1],
        url       = fields[2]:match("^%a+://") and fields[2] or "http://" .. fields[2],
        username  = fields[3] ~= "" and fields[3] or nil,
        password  = fields[4] ~= "" and fields[4] or nil,
        raw_names = fields[5],
        sync      = fields[6],
        use_subdirectory = fields[7],
        delete_missing = fields[8],
    }
    local new_item = buildRootEntry(new_server)
    local new_idx, itemnumber
    if item then
        new_idx = item.idx
        itemnumber = -1
    else
        new_idx = #self.servers + 2
        itemnumber = new_idx
    end
    self.servers[new_idx - 1] = new_server -- first item is "Downloads"
    self.item_table[new_idx] = new_item
    if not no_refresh then
        self:switchItemTable(nil, self.item_table, itemnumber)
    end
    self._manager.updated = true
end
```

**Step 3: Verify the changes**

Read the modified sections to confirm:
```bash
grep -A 35 "function OPDSBrowser:addEditCatalog" plugins/opds.koplugin/opdsbrowser.lua | head -50
grep -A 15 "function OPDSBrowser:editCatalogFromInput" plugins/opds.koplugin/opdsbrowser.lua
```

Expected: New checkboxes and fields are present

**Step 4: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): add UI checkboxes for subdirectory and deletion options

Add two new checkboxes to catalog edit dialog:
- Use subdirectory for this catalog
- Delete books not in catalog

Update editCatalogFromInput to save these settings.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Implement Subdirectory Support

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:988-995`

**Step 1: Update getCurrentDownloadDir() to support subdirectories**

Find the `getCurrentDownloadDir` function around line 988 and replace it:

```lua
function OPDSBrowser:getCurrentDownloadDir()
    if self.sync then
        local base_dir = self.settings.sync_dir
        if self.sync_server and self.sync_server.use_subdirectory then
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

**Step 2: Verify the changes**

Read the function to confirm:
```bash
grep -A 12 "function OPDSBrowser:getCurrentDownloadDir" plugins/opds.koplugin/opdsbrowser.lua
```

Expected: Function creates subdirectory when use_subdirectory is true

**Step 3: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): add subdirectory support for catalog sync

When use_subdirectory is enabled, create and use a subdirectory
named after the catalog title within the sync directory.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Add Metadata Tracking Functions

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:66-75` (init function)
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:1796+` (add new functions at end)

**Step 1: Initialize catalog_files in OPDSBrowser:init()**

Find the `OPDSBrowser:init()` function around line 66 and add catalog file tracking initialization:

```lua
function OPDSBrowser:init()
    self.item_table = self:genItemTableFromRoot()
    self.catalog_title = nil
    self.title_bar_left_icon = "appbar.menu"
    self.onLeftButtonTap = function()
        self:showOPDSMenu()
    end
    self.facet_groups = nil -- Initialize facet groups storage
    self.catalog_files = nil -- Initialize catalog file tracking (loaded lazily)
    Menu.init(self) -- call parent's init()
end
```

**Step 2: Add metadata tracking functions at end of opdsbrowser.lua**

Add these new functions after the last function in the file (after `downloadPendingSyncs`):

```lua
-- Load catalog file tracking from persistent storage
function OPDSBrowser:loadCatalogFileTracking()
    if self.catalog_files then
        return -- already loaded
    end
    local DataStorage = require("datastorage")
    local tracking_file = DataStorage:getSettingsDir() .. "/opds_sync_tracking.lua"
    local LuaSettings = require("luasettings")
    local settings = LuaSettings:open(tracking_file)
    self.catalog_files = settings:readSetting("catalog_files") or {}
    logger.dbg("Loaded catalog file tracking:", self.catalog_files)
end

-- Save catalog file tracking to persistent storage
function OPDSBrowser:saveCatalogFileTracking()
    if not self.catalog_files then
        return
    end
    local DataStorage = require("datastorage")
    local tracking_file = DataStorage:getSettingsDir() .. "/opds_sync_tracking.lua"
    local LuaSettings = require("luasettings")
    local settings = LuaSettings:open(tracking_file)
    settings:saveSetting("catalog_files", self.catalog_files)
    settings:flush()
    logger.dbg("Saved catalog file tracking")
end

-- Track a downloaded file with its catalog and book ID
function OPDSBrowser:trackDownloadedFile(catalog_url, file_path, book_id, book_url)
    if not self.catalog_files then
        self:loadCatalogFileTracking()
    end
    if not self.catalog_files[catalog_url] then
        self.catalog_files[catalog_url] = {}
    end
    self.catalog_files[catalog_url][file_path] = {
        book_id = book_id,
        download_time = os.time(),
        url = book_url,
    }
    self:saveCatalogFileTracking()
    logger.dbg("Tracked file:", file_path, "for catalog:", catalog_url)
end
```

**Step 3: Verify the changes**

Read the new functions to confirm:
```bash
tail -80 plugins/opds.koplugin/opdsbrowser.lua
```

Expected: Three new functions are present at the end of the file

**Step 4: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): add metadata tracking for downloaded files

Add functions to track which files belong to which catalog:
- loadCatalogFileTracking: load from persistent storage
- saveCatalogFileTracking: save to persistent storage
- trackDownloadedFile: record file download with metadata

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 5: Implement Deletion Logic

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:1796+` (add new function)

**Step 1: Add cleanupMissingBooks() function**

Add this function after the tracking functions added in Task 4:

```lua
-- Remove books that are no longer in the catalog feed
function OPDSBrowser:cleanupMissingBooks(catalog_url, current_feed_books)
    if not self.sync_server or not self.sync_server.delete_missing then
        return
    end

    local sync_dir = self:getCurrentDownloadDir()

    -- Strategy 1: SUBDIRECTORY MODE (simpler, more thorough)
    if self.sync_server.use_subdirectory then
        -- Build set of expected filenames from feed
        local feed_files = {}
        for _, book in ipairs(current_feed_books) do
            if book.file_path then
                feed_files[book.file_path] = true
            end
        end

        -- Check all files in the subdirectory
        local files_to_delete = {}
        for entry in lfs.dir(sync_dir) do
            if entry ~= "." and entry ~= ".." then
                local file_path = sync_dir .. "/" .. entry
                local attr = lfs.attributes(file_path)
                -- Only delete regular files, not directories
                if attr and attr.mode == "file" then
                    if not feed_files[file_path] then
                        table.insert(files_to_delete, file_path)
                    end
                end
            end
        end

        -- Delete orphaned files
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

    -- Strategy 2: SHARED DIRECTORY MODE (metadata-based, conservative)
    else
        -- SAFETY: If no tracking data exists, don't delete anything
        if not self.catalog_files or not self.catalog_files[catalog_url] then
            logger.info("No tracking data for catalog, skipping cleanup (safe fallback)")
            return
        end

        -- Build set of book IDs currently in the feed
        local feed_book_ids = {}
        for _, book in ipairs(current_feed_books) do
            if book.book_id then
                feed_book_ids[book.book_id] = true
            end
        end

        -- Check tracked files for this catalog
        local files_to_delete = {}
        for file_path, metadata in pairs(self.catalog_files[catalog_url]) do
            if metadata.book_id and not feed_book_ids[metadata.book_id] then
                if lfs.attributes(file_path) then
                    table.insert(files_to_delete, file_path)
                end
            end
        end

        -- Delete files and remove from tracking
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

**Step 2: Verify the changes**

Read the new function to confirm:
```bash
tail -100 plugins/opds.koplugin/opdsbrowser.lua | head -90
```

Expected: cleanupMissingBooks function is present with both strategies

**Step 3: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): add deletion logic for missing books

Implement cleanupMissingBooks with two strategies:
- Subdirectory mode: scan directory and delete non-feed files
- Shared directory mode: use metadata to delete tracked files only

Multiple safety checks prevent accidental deletions.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 6: Integrate Deletion into Sync Flow - Part 1 (fillPendingSyncs)

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:1542-1609`

**Step 1: Modify fillPendingSyncs to track feed books and call cleanup**

Find the `fillPendingSyncs` function around line 1542 and modify it to load tracking, collect book IDs, and call cleanup:

```lua
-- Add entries to self.pending_syncs
function OPDSBrowser:fillPendingSyncs(server)
    self.root_catalog_password  = server.password
    self.root_catalog_raw_names = server.raw_names
    self.root_catalog_username  = server.username
    self.root_catalog_title     = server.title
    self.sync_server            = server
    self.sync_server_list       = self.sync_server_list or {}
    self.sync_max_dl            = self.settings.sync_max_dl or 50

    -- Load catalog file tracking for deletion support
    self:loadCatalogFileTracking()

    local file_list
    local file_str = self.settings.filetypes
    local new_last_download = nil
    local dl_count = 1
    if file_str then
        file_list = {}
        for filetype in util.gsplit(file_str, ",") do
            file_list[util.trim(filetype)] = true
        end
    end
    local sync_list = self:getSyncDownloadList()
    if sync_list then
        local current_feed_books = {}  -- Track books in current feed for cleanup

        for i, entry in ipairs(sync_list) do
            -- for project gutenberg
            local sub_table = {}
            local item
            if entry.url then
                sub_table = self:getSyncDownloadList(entry.url)
            end
            if #sub_table > 0 then
                -- The first element seems to be most compatible. Second element has most options
                item = sub_table[2]
            else
                item = entry
            end
            for j, link in ipairs(item.acquisitions) do
                -- Only save first link in case of several file types
                if i == 1 and j == 1 then
                    new_last_download = link.href
                end
                local filetype = self.getFiletype(link)
                if filetype then
                    if not file_str or file_list and file_list[filetype] then
                        local filename = self:getFileName(entry)
                        local download_path = self:getLocalDownloadPath(filename, filetype, link.href)

                        -- Extract book ID (use entry ID from feed or hash of URL)
                        local book_id = entry.id or util.getURLHash(link.href)

                        if dl_count <= self.sync_max_dl then -- Append only max_dl entries... may still have sync backlog
                            table.insert(self.pending_syncs, {
                                file = download_path,
                                url = link.href,
                                username = self.root_catalog_username,
                                password = self.root_catalog_password,
                                catalog = server.url,
                                book_id = book_id,
                            })
                            dl_count = dl_count + 1
                        end

                        -- Track this book as part of current feed
                        table.insert(current_feed_books, {
                            book_id = book_id,
                            file_path = download_path,
                        })

                        break
                    end
                end
            end
        end

        -- Cleanup missing books after building download list
        self:cleanupMissingBooks(server.url, current_feed_books)
    end
    self.sync_server_list[server.url] = true
    if new_last_download then
        logger.dbg("Updating opds last download for server", server.title, "to", new_last_download)
        self:updateFieldInCatalog(server, "last_download", new_last_download)
    end

end
```

**Step 2: Verify the changes**

Read the function to confirm:
```bash
grep -A 80 "function OPDSBrowser:fillPendingSyncs" plugins/opds.koplugin/opdsbrowser.lua
```

Expected: Function loads tracking, collects book_ids, and calls cleanupMissingBooks

**Step 3: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): integrate deletion into fillPendingSyncs

Modify fillPendingSyncs to:
- Load catalog file tracking at start
- Extract book IDs from feed entries
- Track current feed books
- Call cleanupMissingBooks after building download list

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 7: Integrate Deletion into Sync Flow - Part 2 (Track Downloads)

**Files:**
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:1029-1072`
- Modify: `plugins/opds.koplugin/opdsbrowser.lua:1675-1796`

**Step 1: Update downloadFile to track successful downloads**

Find the `downloadFile` function around line 1029 and modify the success section:

```lua
function OPDSBrowser:downloadFile(local_path, remote_url, username, password, caller_callback)
    logger.dbg("Downloading file", local_path, "from", remote_url)
    local code, headers, status
    local parsed = url.parse(remote_url)
    if parsed.scheme == "http" or parsed.scheme == "https" then
        socketutil:set_timeout(socketutil.FILE_BLOCK_TIMEOUT, socketutil.FILE_TOTAL_TIMEOUT)
        code, headers, status = socket.skip(1, http.request {
            url      = remote_url,
            headers  = {
                ["Accept-Encoding"] = "identity",
            },
            sink     = ltn12.sink.file(io.open(local_path, "w")),
            user     = username,
            password = password,
        })
        socketutil:reset_timeout()
    else
        UIManager:show(InfoMessage:new {
            text = T(_("Invalid protocol:\n%1"), parsed.scheme),
        })
    end
    if code == 200 then
        logger.dbg("File downloaded to", local_path)

        -- Track this download if in sync mode with metadata
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
    elseif code == 302 and remote_url:match("^https") and headers.location:match("^http[^s]") then
        util.removeFile(local_path)
        UIManager:show(InfoMessage:new{
            text = T(_("Insecure HTTPS → HTTP downgrade attempted by redirect from:\n\n'%1'\n\nto\n\n'%2'.\n\nPlease inform the server administrator that many clients disallow this because it could be a downgrade attack."), BD.url(remote_url), BD.url(headers.location)),
            icon = "notice-warning",
        })
    else
        util.removeFile(local_path)
        logger.dbg("OPDSBrowser:downloadFile: Request failed:", status or code)
        logger.dbg("OPDSBrowser:downloadFile: Response headers:", headers)
        UIManager:show(InfoMessage:new {
            text = T(_("Could not save file to:\n%1\n%2"),
                BD.filepath(local_path),
                status or code or "network unreachable"),
        })
    end
end
```

**Step 2: Update downloadPendingSyncs to pass book metadata**

Find the `downloadPendingSyncs` function around line 1675 and modify the download loop section (inside the dismissable_download function):

```lua
-- Download pending syncs list
function OPDSBrowser:downloadPendingSyncs()
    local dl_list = self.pending_syncs
    local function dismissable_download()
        local info = InfoMessage:new{ text = _("Downloading… (tap to cancel)") }
        UIManager:show(info)
        UIManager:forceRePaint()
        local completed, downloaded, duplicate_list = Trapper:dismissableRunInSubprocess(function()
            local dl = {}
            local dupe_list = {}
            for _, item in ipairs(dl_list) do
                if self.sync_server_list[item.catalog] then
                    if lfs.attributes(item.file) and not self.sync_force then
                        table.insert(dupe_list, item)
                    else
                        -- Set current book metadata for tracking
                        self.current_book_metadata = {
                            catalog = item.catalog,
                            book_id = item.book_id,
                        }

                        if self:downloadFile(item.file, item.url, item.username, item.password) then
                            dl[item.file] = true
                        end

                        self.current_book_metadata = nil
                    end
                end
            end
            return dl, dupe_list
        end, info)

        if completed then
            UIManager:close(info)
        end
        local dl_count = 0
        local dl_size = #dl_list
        for i = dl_size, 1, -1 do
            local item = dl_list[i]
            if downloaded and downloaded[item.file] then
                dl_count = dl_count + 1
                table.remove(dl_list, i)
            else -- if subprocess has been interrupted, check for the downloaded file
                local attr = lfs.attributes(item.file)
                if attr then
                    if attr.size > 0 then
                        table.remove(dl_list, i)
                        if attr.modification > os.time() - 300 then -- Only count files touched in the last 5 mins
                            dl_count = dl_count + 1
                        end
                    else -- incomplete download
                        os.remove(item.file)
                    end
                end
            end
        end
        local duplicate_count = duplicate_list and #duplicate_list or 0
        dl_count = dl_count - duplicate_count
        -- Make downloaded count timeout if there's a duplicate file prompt
        local timeout = nil
        if duplicate_count > 0 then
            timeout = 3
        end
        if dl_count > 0 then
            UIManager:show(InfoMessage:new{ text = T(N_("1 book downloaded", "%1 books downloaded", dl_count), dl_count), timeout = timeout,})
        end
        self._manager.updated = true
        return duplicate_list
    end

    local duplicate_list = dismissable_download()

    if duplicate_list and #duplicate_list > 0 then
        local textviewer
        local duplicate_files = { _("These files are already on the device:") }
        for _, entry in ipairs(duplicate_list) do
            table.insert(duplicate_files, entry.file)
        end
        local text = table.concat(duplicate_files, "\n")
        textviewer = TextViewer:new{
            title = _("Duplicate files"),
            text = text,
            buttons_table = {
                {
                    {
                        text = _("Do nothing"),
                        callback = function()
                            textviewer:onClose()
                        end
                    },
                    {
                        text = _("Overwrite"),
                        callback = function()
                            self.sync_force = true
                            textviewer:onClose()
                            for _, entry in ipairs(duplicate_list) do
                                table.insert(dl_list, entry)
                            end
                            Trapper:wrap(function()
                                dismissable_download()
                            end)
                        end
                    },
                    {
                        text = _("Download copies"),
                        callback = function()
                            self.sync_force = true
                            textviewer:onClose()
                            local copy_download_dir, original_dir, copies_dir, copy_download_path
                            copies_dir = "copies"
                            original_dir = util.splitFilePathName(duplicate_list[1].file)
                            copy_download_dir = original_dir .. copies_dir .. "/"
                            util.makePath(copy_download_dir)
                            for _, entry in ipairs(duplicate_list) do
                                local _, file_name = util.splitFilePathName(entry.file)
                                copy_download_path = copy_download_dir .. file_name
                                entry.file = copy_download_path
                                table.insert(dl_list, entry)
                            end
                            Trapper:wrap(function()
                                dismissable_download()
                            end)
                        end
                    },
                },
            },
        }
        UIManager:show(textviewer)
    end
end
```

**Step 3: Verify the changes**

Read both functions to confirm:
```bash
grep -A 30 "function OPDSBrowser:downloadFile" plugins/opds.koplugin/opdsbrowser.lua | head -40
grep -A 50 "function OPDSBrowser:downloadPendingSyncs" plugins/opds.koplugin/opdsbrowser.lua | head -60
```

Expected: downloadFile tracks downloads, downloadPendingSyncs sets metadata

**Step 4: Commit**

```bash
git add plugins/opds.koplugin/opdsbrowser.lua
git commit -m "feat(opds): track downloads in sync flow

Modify downloadFile to track successful downloads with metadata.
Modify downloadPendingSyncs to set current_book_metadata before
downloading so files can be tracked properly.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 8: Add util.getURLHash() Helper Function

**Files:**
- Check: `frontend/util.lua` (to see if getURLHash exists)
- Modify: `frontend/util.lua` (add function if missing)

**Step 1: Check if util.getURLHash exists**

```bash
grep "function.*getURLHash" frontend/util.lua
```

Expected: Empty output (function doesn't exist)

**Step 2: Find a good location to add the function**

```bash
grep -n "function util\." frontend/util.lua | tail -5
```

This will show the last few util functions to find where to add the new one.

**Step 3: Add util.getURLHash() function**

Add this function near the end of the file, before the `return util` statement:

```lua
--- Create a stable hash from a URL for use as an identifier
-- @string url The URL to hash
-- @treturn string A hash string suitable for use as an identifier
function util.getURLHash(url)
    -- Simple but stable hash function
    local hash = 0
    for i = 1, #url do
        hash = (hash * 31 + string.byte(url, i)) % 2147483647
    end
    return string.format("url_%d", hash)
end
```

**Step 4: Verify the addition**

```bash
grep -A 8 "function util.getURLHash" frontend/util.lua
```

Expected: Function is present with proper implementation

**Step 5: Commit**

```bash
git add frontend/util.lua
git commit -m "feat(util): add getURLHash helper function

Add util.getURLHash() to create stable hash identifiers from URLs.
Used by OPDS sync to track books when feed doesn't provide IDs.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 9: Manual Testing and Documentation

**Files:**
- Create: `docs/plans/2026-02-04-opds-sync-deletion-testing.md`

**Step 1: Create testing documentation**

```markdown
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
```

**Step 2: Save the testing documentation**

Already done by writing the file content above.

**Step 3: Commit**

```bash
git add docs/plans/2026-02-04-opds-sync-deletion-testing.md
git commit -m "docs(opds): add testing guide for sync deletion feature

Document test scenarios, verification checklist, and edge cases
for the OPDS sync deletion feature.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 10: Final Review and Verification

**Files:**
- Review: All modified files

**Step 1: Review all changes**

```bash
git log --oneline feature/opds-sync-deletion ^master
```

Expected: 9-10 commits implementing the feature

**Step 2: Check for any syntax errors in Lua files**

```bash
lua -e "dofile('plugins/opds.koplugin/main.lua')" 2>&1 | head -20
lua -e "dofile('plugins/opds.koplugin/opdsbrowser.lua')" 2>&1 | head -20
```

Note: These will show errors about missing dependencies, but should not show syntax errors.

**Step 3: Verify all files are committed**

```bash
git status
```

Expected: "nothing to commit, working directory clean"

**Step 4: Create summary of changes**

```bash
git diff master --stat
```

Expected: Shows changes to main.lua, opdsbrowser.lua, util.lua, .gitignore, and docs

**Step 5: Final commit message preparation**

No commit needed - this is just verification.

---

## Summary

This implementation adds:
- ✅ Per-catalog subdirectory support
- ✅ Per-catalog deletion of missing books
- ✅ Two deletion strategies (subdirectory vs shared)
- ✅ Metadata tracking for safe deletion
- ✅ UI checkboxes for configuration
- ✅ Backwards compatibility with migration
- ✅ Multiple safety fallbacks
- ✅ Comprehensive testing documentation

All changes maintain backwards compatibility with safe defaults (both options disabled).

## Next Steps

After implementation:
1. Manual testing following the testing guide
2. Consider opening a pull request to KOReader upstream
3. Monitor for user feedback on deletion behavior
4. Potential future enhancement: visual indicator showing which catalogs have deletion enabled
