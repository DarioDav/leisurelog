# Leisurelog

A personal media tracker for books, shows, anime, movies, games, podcasts, activities, music, and more. Inspired by Goodreads but for everything you consume.

Single HTML file — no build tools, no server, no dependencies to install. Open it in any browser and it works.

## Features

- **9 media types** — Books, Shows, Anime, Movies, Games, Podcasts, Activities, Music, Other
- **Status tracking** — Want to, In Progress, Completed, On Hold, Dropped
- **Per-entry fields** — creator, star rating (1–5), progress, start/finish dates, tags, notes
- **Auto start date** — new entries default to today, override anytime
- **Dashboard** — stats by type, recently added, top rated, status breakdown
- **Library** — filter by type or status, full-text search, sortable
- **Google Sheets sync** — optional cloud backend so data works across devices
- **Token-protected API** — requests without your secret token are rejected
- **Offline-first** — always cached locally in `localStorage`

## Usage

Just open `index.html` in a browser. No installation required.

**Keyboard shortcuts**

| Shortcut | Action |
|----------|--------|
| `⌘K` / `Ctrl+K` | Focus search |
| `⌘N` / `Ctrl+N` | Add new entry |
| `Esc` | Close modal |

## Google Sheets Sync (optional)

Lets you use the app across multiple devices. Your data stays in a Google Sheet you own.

### Setup

1. Go to [sheets.google.com](https://sheets.google.com) and create a new spreadsheet.
2. Click **Extensions → Apps Script**. Delete the placeholder and paste the script below.
3. On line 1, replace `your-secret-token-here` with a strong passphrase of your choosing.
4. Click **Deploy → New deployment**. Type: **Web app**. Execute as: **Me**. Who has access: **Anyone**. Deploy and authorise.
5. Copy the Web App URL. Open **Settings** (⚙️) in the app, paste the URL and your token, click **Connect**.

### Apps Script

```javascript
const SECRET     = 'your-secret-token-here'; // ← change this!
const SHEET_NAME = 'Entries';
const COLS = ['id','title','type','status','creator','rating',
              'progress','tags','notes','dateAdded','dateStarted','dateFinished'];

function doGet(e) {
  var params = e.parameter || {};
  if (params.token !== SECRET) return forbidden();
  if (params.action === 'load') return respond(loadEntries());
  return respond({ error: 'Unknown action' });
}

function doPost(e) {
  var body = JSON.parse(e.postData.contents);
  if (body.token !== SECRET) return forbidden();
  if (body.action === 'save') {
    saveEntries(body.entries || []);
    return respond({ ok: true, saved: body.entries.length });
  }
  return respond({ error: 'Unknown action' });
}

function forbidden() {
  return ContentService
    .createTextOutput(JSON.stringify({ error: 'Unauthorized' }))
    .setMimeType(ContentService.MimeType.JSON);
}

function loadEntries() {
  var sheet = getSheet();
  var rows  = sheet.getDataRange().getValues();
  if (rows.length <= 1) return { entries: [] };
  var entries = rows.slice(1).map(function(row) {
    var entry = {};
    COLS.forEach(function(c, i) {
      entry[c] = row[i] === '' ? null : row[i];
    });
    entry.tags   = entry.tags ? String(entry.tags).split(',') : [];
    entry.rating = entry.rating !== null ? Number(entry.rating) : null;
    return entry;
  });
  return { entries: entries };
}

function saveEntries(entries) {
  var sheet = getSheet();
  sheet.clearContents();
  sheet.appendRow(COLS);
  entries.forEach(function(entry) {
    sheet.appendRow(COLS.map(function(c) {
      if (c === 'tags') return (entry[c] || []).join(',');
      return entry[c] == null ? '' : entry[c];
    }));
  });
}

function getSheet() {
  var ss    = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName(SHEET_NAME);
  if (!sheet) {
    sheet = ss.insertSheet(SHEET_NAME);
    sheet.appendRow(COLS);
  }
  return sheet;
}

function respond(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### Security

The Apps Script URL and secret token are **never stored in this repo** — they live only in each device's `localStorage`. Publishing this code to GitHub exposes nothing sensitive. Anyone who finds the URL without the token gets a 401 and nothing else.

On each new device, open Settings (⚙️) and enter the URL + token once.

> **Note:** avoid making changes on two devices simultaneously. The sync model is last-write-wins — open the app, let it sync, then make changes.

## Data

All data is stored locally in `localStorage` under the key `leisurelog_v1`. If Sheets sync is configured, a full copy also lives in the connected spreadsheet. Clearing browser data removes the local cache but leaves Sheets untouched.
