# Xiaomi / Mi Cloud Notes → `.enex` (Evernote / Apple Notes)  
*aka: “Get my notes out of there, please.”*

---

### What this does (in plain human):
- Logs into **us.i.mi.com/note** (you do that), then grabs **all your notes**.
- Saves them as **`.enex` files** (Evernote export format).  
  - **Apple Notes on macOS:** just double-click the `.enex` file and it imports.
  - **Evernote / Notebooks / others:** use their “Import Evernote” option.
- Keeps your **text, dates, emojis, real hashtags**.  
- **Removes internal tags** like `#noteId_...` or `#folderId_...`.  
- Prints how many notes/folders were saved and how many failed (404s, etc.).

You get **two flavors**:

1. **Everything in one `.enex` file** (folders will be still saved as hashtags)
2. **One `.enex` per Xiaomi folder** (nice if you want separate notebooks)

---

## Quick Start (no tech degree needed)

1. Open: **https://us.i.mi.com/note/h5** and sign in.  
2. Open the **Developer Console** (copy/paste land):  
   - Windows/Linux: `Ctrl`+`Shift`+`I` → “Console”  
   - macOS: `⌥⌘I` → “Console”  
3. Copy ONE of the scripts below, paste into the Console, hit **Enter**.  
4. Wait. Notes download. (may ~3 minutes per 100 note)
5. Celebrate.

---

👇 click to view code👇
<details>

<summary>👉 Option A: One big `.enex` file👈</summary>

```javascript
(async () => {
  /******** CONFIG ********/
  const LIMIT          = 200;
  const FETCH_DELAY_MS = 80;          // ms between note-detail calls
  const ROOT_TAG       = "Unfoldered";
  const ALWAYS_TAGS    = ["xiaomi", "Imported-Notes"];
  /************************/

  const ts   = () => Date.now();
  const pad  = (n,l=2)=>String(n).padStart(l,'0');
  const isoToEver = d => {
    const iso = (d instanceof Date ? d : new Date(d)).toISOString();
    return iso.replace(/[-:]/g,'').replace(/\.\d{3}Z$/,'Z');
  };
  const nowForName = d => `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}_${pad(d.getHours())}-${pad(d.getMinutes())}-${pad(d.getSeconds())}`;

  const xmlEscape = s => (s||'')
    .replace(/&/g,'&amp;')
    .replace(/</g,'&lt;')
    .replace(/>/g,'&gt;');

  const toENML = html =>
`<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE en-note SYSTEM "http://xml.evernote.com/pub/enml2.dtd">
<en-note>${html}</en-note>`;

  async function fetchJSON(url){
    const res = await fetch(url, {credentials:'include'});
    if(!res.ok){ const err = new Error(res.statusText); err.status=res.status; throw err; }
    return res.json();
  }

  async function getPage(syncTag){
    const u = new URL('/note/full/page', location.origin);
    u.searchParams.set('limit', LIMIT);
    u.searchParams.set('ts', ts());
    if(syncTag) u.searchParams.set('syncTag', syncTag);
    return fetchJSON(u.toString());
  }

  async function getFullNote(id){
    const url = `/note/note/${id}/?ts=${ts()}`;
    const j = await fetchJSON(url);
    return j?.data?.entry?.content || '';
  }

  console.log('Collecting pages…');
  let syncTag = null;
  const notes = [];
  const seen = new Set();
  const folderMap = {}; // id -> name
  let round = 0;

  while(true){
    const j = await getPage(syncTag);
    const d = j?.data || {};

    (d.folders || []).forEach(f=>{
      if(f.type === 'folder' && f.id) folderMap[f.id] = f.subject || '';
    });

    (d.entries || []).forEach(e=>{
      if(e.type === 'note' && !seen.has(e.id)){
        seen.add(e.id);
        notes.push(e);
      }
    });

    syncTag = d.syncTag;
    round++;
    if(d.lastPage) break;
    if(round>100){ console.warn('Too many rounds, stopping'); break; }
  }

  console.log(`Got ${notes.length} notes across ${Object.keys(folderMap).length} folders. Fetching contents…`);

  let success = 0, failed = 0, notFound = 0, idx = 0;
  const tagSet = new Set(ALWAYS_TAGS); // gather for final print

  const pieces = [];
  pieces.push(`<?xml version="1.0" encoding="UTF-8"?>`);
  pieces.push(`<en-export export-date="${isoToEver(new Date())}" application="xiaomi-to-enex" version="10">`);

  for(const e of notes){
    idx++;
    document.title = `ENEX ${idx}/${notes.length}`;

    const extra = e.extraInfo ? JSON.parse(e.extraInfo) : {};
    let title = (extra.title || e.subject || '').trim();
    if(!title){
      title = (e.snippet || 'note').trim().slice(0,60) || 'note';
    }

    // Fetch full content
    let content = '';
    try{
      content = await getFullNote(e.id);
      if(!content){
        // fallback if content empty
        if(extra.note_content_type === 'mind'){
          content = extra.mind_content_plain_text || '';
        }else{
          content = e.snippet || '';
        }
      }
      await new Promise(r=>setTimeout(r, FETCH_DELAY_MS));
      success++;
    }catch(err){
      failed++;
      if(err.status===404) notFound++;
      console.warn('Failed to fetch note', e.id, err);
      content = e.snippet || '';
    }

    // Preserve Xiaomi inline HTML like <u>, add <br/> for newlines
    const htmlBody = (content||'').replace(/\r?\n/g,'<br/>');
    const enml = toENML(htmlBody);

    // tags
    const tags = [];
    const folderName = folderMap[e.folderId] || ROOT_TAG;
    if(folderName) { tags.push(folderName); tagSet.add(folderName); }
    ALWAYS_TAGS.forEach(t=>tags.push(t));

    const created = isoToEver(e.createDate || Date.now());
    const updated = isoToEver(e.modifyDate || e.createDate || Date.now());

    const noteParts = [];
    noteParts.push(`<note>`);
    noteParts.push(`<title>${xmlEscape(title)}</title>`);
    noteParts.push(`<content><![CDATA[${enml}]]></content>`);
    noteParts.push(`<created>${created}</created>`);
    noteParts.push(`<updated>${updated}</updated>`);
    tags.forEach(t => noteParts.push(`<tag>${xmlEscape(t)}</tag>`));
    noteParts.push(`<note-attributes><source-application>xiaomi-notes</source-application></note-attributes>`);
    noteParts.push(`</note>`);

    pieces.push(noteParts.join(''));
  }

  pieces.push(`</en-export>`);

  const blob = new Blob([pieces.join('\n')], {type:'application/xml'});
  const url  = URL.createObjectURL(blob);
  const a    = document.createElement('a');
  a.href = url;
  a.download = `xiaomi_notes_${nowForName(new Date())}.enex`;
  a.click();
  setTimeout(()=>URL.revokeObjectURL(url), 60000);

  const folderCount = Object.keys(folderMap).length;
  const tagList = Array.from(tagSet).sort();

  console.log('------ SUMMARY ------');
  console.log('Success:', success);
  console.log('Failed:', failed);
  console.log('404:', notFound);
  console.log('Folders:', folderCount);
  console.log('Tags used:', tagList);

  alert(`ENEX ready!
Success: ${success}
Failed: ${failed}
404s: ${notFound}
Folders: ${folderCount}
Tags: ${tagList.join(', ')}`);
})();
```
</details>
👆 click to view code 👆

---

👇 click to view code👇
<details>

<summary>👉 Option B: One `.enex` per folder👈</summary>

```javascript
(async () => {
  /******** CONFIG ********/
  const LIMIT          = 200;
  const FETCH_DELAY_MS = 80;          // delay between note detail requests (ms)
  const ROOT_TAG       = "Unfoldered";
  const ALWAYS_TAGS    = ["xiaomi", "Imported-Notes"];
  /************************/

  const ts   = () => Date.now();
  const pad  = (n,l=2)=>String(n).padStart(l,'0');
  const isoToEver = d => {
    const iso = (d instanceof Date ? d : new Date(d)).toISOString();
    return iso.replace(/[-:]/g,'').replace(/\.\d{3}Z$/,'Z');
  };
  const nowForName = d => `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}_${pad(d.getHours())}-${pad(d.getMinutes())}-${pad(d.getSeconds())}`;
  const xmlEscape = s => (s||'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');

  const toENML = html =>
`<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE en-note SYSTEM "http://xml.evernote.com/pub/enml2.dtd">
<en-note>${html}</en-note>`;

  async function fetchJSON(url){
    const res = await fetch(url, {credentials:'include'});
    if(!res.ok){ const err = new Error(res.statusText); err.status=res.status; throw err; }
    return res.json();
  }

  async function getPage(syncTag){
    const u = new URL('/note/full/page', location.origin);
    u.searchParams.set('limit', LIMIT);
    u.searchParams.set('ts', ts());
    if(syncTag) u.searchParams.set('syncTag', syncTag);
    return fetchJSON(u.toString());
  }

  async function getFullNote(id){
    const url = `/note/note/${id}/?ts=${ts()}`;
    const j = await fetchJSON(url);
    return j?.data?.entry?.content || '';
  }

  console.log('Collecting pages…');
  let syncTag = null;
  const notes = [];
  const seen = new Set();
  const folderMap = {}; // id -> name
  let round = 0;

  while(true){
    const j = await getPage(syncTag);
    const d = j?.data || {};

    (d.folders || []).forEach(f=>{
      if(f.type === 'folder' && f.id) folderMap[f.id] = f.subject || '';
    });

    (d.entries || []).forEach(e=>{
      if(e.type === 'note' && !seen.has(e.id)){
        seen.add(e.id);
        notes.push(e);
      }
    });

    syncTag = d.syncTag;
    round++;
    if(d.lastPage) break;
    if(round>100){ console.warn('Too many rounds, stopping'); break; }
  }

  console.log(`Got ${notes.length} notes across ${Object.keys(folderMap).length} folders. Fetching contents…`);

  let success = 0, failed = 0, notFound = 0, idx = 0;
  const tagSet = new Set(ALWAYS_TAGS);
  const groups = {}; // folderName => [noteObjects]

  const sleep = ms => new Promise(r=>setTimeout(r, ms));

  for(const e of notes){
    idx++;
    document.title = `Note ${idx}/${notes.length}`;

    const extra = e.extraInfo ? JSON.parse(e.extraInfo) : {};
    let title = (extra.title || e.subject || '').trim();
    if(!title){
      title = (e.snippet || 'note').trim().slice(0,60) || 'note';
    }

    // fetch content
    let content = '';
    try{
      content = await getFullNote(e.id);
      if(!content){
        if(extra.note_content_type === 'mind'){
          content = extra.mind_content_plain_text || '';
        }else{
          content = e.snippet || '';
        }
      }
      success++;
    }catch(err){
      failed++;
      if(err.status===404) notFound++;
      console.warn('Failed to fetch note', e.id, err);
      content = e.snippet || '';
    }
    await sleep(FETCH_DELAY_MS);

    const htmlBody = (content||'').replace(/\r?\n/g,'<br/>');
    const enml = toENML(htmlBody);

    const folderName = folderMap[e.folderId] || ROOT_TAG;
    tagSet.add(folderName);

    const tags = [folderName, ...ALWAYS_TAGS];

    const created = isoToEver(e.createDate || Date.now());
    const updated = isoToEver(e.modifyDate || e.createDate || Date.now());

    const noteXML =
`<note>
  <title>${xmlEscape(title)}</title>
  <content><![CDATA[${enml}]]></content>
  <created>${created}</created>
  <updated>${updated}</updated>
  ${tags.map(t=>`<tag>${xmlEscape(t)}</tag>`).join('\n  ')}
  <note-attributes><source-application>xiaomi-notes</source-application></note-attributes>
</note>`;

    if(!groups[folderName]) groups[folderName] = [];
    groups[folderName].push(noteXML);
  }

  // build & download per folder
  const folderNames = Object.keys(groups);
  let fileIdx = 0;
  for(const fname of folderNames){
    fileIdx++;
    const header = `<?xml version="1.0" encoding="UTF-8"?>\n<en-export export-date="${isoToEver(new Date())}" application="xiaomi-to-enex" version="10">`;
    const body = groups[fname].join('\n');
    const footer = `</en-export>`;
    const xml = [header, body, footer].join('\n');
    const blob = new Blob([xml], {type:'application/xml'});
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href = url;
    a.download = `xiaomi_${fname || 'Unfoldered'}_${nowForName(new Date())}.enex`;
    a.click();
    setTimeout(()=>URL.revokeObjectURL(url), 60000);
  }

  const folderCount = folderNames.length;
  const tagList = Array.from(tagSet).sort();

  console.log('------ SUMMARY ------');
  console.log('Success:', success);
  console.log('Failed:', failed);
  console.log('404:', notFound);
  console.log('Folders:', folderCount);
  console.log('Tags used:', tagList);

  alert(`ENEX files ready!
Success: ${success}
Failed: ${failed}
404s: ${notFound}
Folders: ${folderCount}
Tags: ${tagList.join(', ')}`);
})();
```
</details>
👆 click to view code 👆

---

### Tiny Warning (because… internet)
- Use this **only for your own notes**.
- Check Mi Cloud’s Terms of Service.  
- If Xiaomi gets grumpy, that’s between you and them — not me. 😉

---

### Problems?

- **No download?** Allow pop-ups/automatic downloads for `us.i.mi.com`.  
- **Missing notes?** Scroll the console for red errors/404 counts.  
- **Still stuck?** Refresh, log in again, retry.

---

#### Trying to reach you out (ignore this, if you are not a search engine😉)  
_xiaomi notes export macos, mi cloud notes backup, download xiaomi notes, backup xiaomi notes, xiaomi notes export chrome extension, move xiaomi notes to evernote, import xiaomi notes to apple notes_

---

**MIT-ish Vibes:** Do whatever, just don’t blame me if xiaomi changed their API or your Receipt printer starts syncing notes.
