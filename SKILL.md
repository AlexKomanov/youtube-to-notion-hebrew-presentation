---
name: youtube-videos-to-notion-db
description: >
  Add YouTube videos to the Notion YouTube Watch List. Trigger when the user
  pastes YouTube URLs (youtube.com/watch or youtu.be), provides a JSON array of
  video metadata, or says something like "add to Notion", "save to watch list",
  "push to Notion DB", or "log these videos". Handles both raw URLs (extracts
  metadata via yt-dlp) and pre-built JSON arrays. Deduplicates against existing
  entries, assigns buckets, and bulk-creates all pages in one operation.
---

# YouTube → Notion Skill

Add YouTube videos to the Notion YouTube Watch List from either raw URLs or a JSON array of video metadata.

## Step 0 — Detect input type

**URLs:** User provided one or more YouTube URLs (`youtube.com/watch?v=` or `youtu.be/`). Go to Step 1.

**JSON array:** User provided a pre-built array of video objects with fields like `Title`, `Channel`, `Duration`, `URL`. Skip to Step 2 with that data — no yt-dlp needed.

---

## Step 1 — Extract metadata via yt-dlp (URLs only)

First verify yt-dlp is available:

```bash
which yt-dlp
```

If not found: tell the user to run `brew install yt-dlp` and stop.

For each URL run:

```bash
yt-dlp --dump-json --no-playlist "<url>"
```

Transform the JSON output:

| yt-dlp field      | Transformation                        | Output field           |
|-------------------|---------------------------------------|------------------------|
| `title`           | none                                  | `Title`                |
| `channel`         | none                                  | `Channel`              |
| `duration_string` | none (already `HH:MM:SS` or `MM:SS`) | `Duration`             |
| `upload_date`     | `"20091025"` → `"2009-10-25"`        | `date:Published:start` |
| `webpage_url`     | none                                  | `userDefined:URL`      |

Date conversion: insert `-` after characters 4 and 6 of `upload_date`.

If yt-dlp fails for a URL: warn ("⚠️ Could not fetch metadata for `<url>`, skipping") and continue with the rest.

---

## Step 2 — Find the database

Search Notion for the YouTube Watch List:

```
notion-search: query="YouTube Watch List"
```

Pick the result with `type: "database"`. Note its `id`.

Fetch the database schema:

```
notion-fetch: id=<database_id>
```

Extract from the response:
- The **data source ID** from `<data-source url="collection://...">` — used when creating pages
- The **schema** — property names and types
- The **Bucket select options** — note all valid option names
- All existing **`userDefined:URL`** values from pages already in the database — used for duplicate detection

---

## Step 3 — Deduplicate

Compare each incoming video's URL against existing URLs collected in Step 2:
- **Match** → mark as duplicate, exclude from batch
- **No match** → include in batch

If **all** videos are duplicates: tell the user all videos are already in the Watch List and stop.

Track the skipped count for the final report.

---

## Step 4 — Assign Bucket values

For each non-duplicate video, assign a `Bucket` based on title and channel signals. Use exact option names from the schema:

| Bucket | Assign when... |
|--------|----------------|
| `Claude Code` | Title/channel mentions Claude, Claude Code, Anthropic, Managed Agents, AI coding tools |
| `Lessons` | Technical tutorials, how-to, dev tools (AWS, Postman, CI/CD, MERN, Cursor, AI agents) |
| `English` | English language learning content |
| `General` | News, politics, entertainment, watches, lifestyle, short clips, unrelated to above |
| `Long Run` | Very long content (>45 min) worth saving for later |
| `Articles` / `To Sort` | Use if no other bucket fits, or if the DB has these options |

If the DB has no `Bucket` field, skip it.

---

## Step 5 — Bulk create pages

Call `notion-create-pages` **once** with all non-duplicate videos. Do not loop one-by-one.

```json
{
  "parent": { "data_source_id": "<id from Step 2>", "type": "data_source_id" },
  "pages": [
    {
      "properties": {
        "Title": "<video title>",
        "Channel": "<channel name>",
        "Duration": "<HH:MM:SS or MM:SS>",
        "date:Published:start": "<YYYY-MM-DD>",
        "date:Published:is_datetime": 0,
        "userDefined:URL": "<youtube url>",
        "Bucket": "<assigned bucket>",
        "Watched": "__NO__"
      }
    }
  ]
}
```

---

## Step 6 — Report results

Tell the user:
- Total videos added and breakdown by Bucket
- How many were skipped as duplicates (if any)

Example:
> ✅ 3 videos added to **YouTube Watch List**!
> - **Claude Code**: 1
> - **Lessons**: 2
>
> ⏭️ 2 skipped — already in the Watch List.

---

## Error Handling

| Error | Action |
|-------|--------|
| `yt-dlp` not installed | Tell user `brew install yt-dlp`, stop |
| `yt-dlp` fails for a URL | Warn and skip that URL, continue with others |
| All URLs failed yt-dlp | Tell user no metadata could be extracted, stop |
| DB not found | Ask user to share the database URL directly, use `notion-fetch` on it |
| No Bucket field in schema | Skip bucket assignment, create pages without it |
| Missing fields in JSON input | Skip optional fields — only `Title` and `URL` are required |
| All videos are duplicates | Tell user all videos are already in the Watch List, stop |
