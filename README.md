# STP Batch HTML Generator — Web Deployment

`example.com/post/` — generator page + generated ZIP একই folder-এ।

---

## Folder structure (upload করার পর)

```
public_html/
└── post/
    ├── index.html      ← generator page (example.com/post/)
    ├── save.php        ← ZIP save + extract endpoint
    ├── config.php      ← token + settings (edit করতেই হবে)
    ├── .htaccess       ← hardening
    ├── html-batch.zip  ← (auto তৈরি হবে)
    └── match-001.html… ← (extract on করলে auto তৈরি হবে)
```

---

## Deploy steps

### 1. Token change করো (বাধ্যতামূলক)
`config.php` খুলে `'token' => '...'` line-এ নতুন random token বসাও।

SSH থাকলে:
```bash
php -r "echo bin2hex(random_bytes(24)), PHP_EOL;"
```
না থাকলে যেকোনো ৩০+ character random string দাও। Default token রেখে দিলে
`save.php` ইচ্ছে করেই error দেবে।

### 2. Upload
পুরো `post/` folder টা `public_html/`-এ upload করো (hPanel File Manager বা FTP)।

### 3. Permission
```
folder post/   → 755
*.php, *.html  → 644
```

### 4. Test
`https://example.com/post/` খোলো → **Server token** field-এ config.php-এর token
বসাও → Generate চাপো।

সফল হলে নিচে link আসবে:
```
📦 https://example.com/post/html-batch.zip
```

---

## Output Path (নতুন — frontend থেকে destination change)

**Server Save** section-এ এখন একটা **Output Path** field আছে। এখানে
site root (public_html) থেকে relative path দাও — প্রতিবার generate
করার সময় আলাদা destination-এ পাঠাতে পারবে, tool-টা `/post/`-এই থাকুক।

```
খালি রাখলে      → config.php-এর default 'output_dir' (আগের মতো)
"video"          → example.com/video/
"hex/video"      → example.com/hex/video/
"news/2026/wc"   → example.com/news/2026/wc/
```

- Path browser-এ মনে থাকে (localStorage) — পরের বার auto-fill হয়ে থাকবে।
- `../` বা absolute path (`/etc/...`) দিলে server 403 দিয়ে reject করবে —
  base-এর বাইরে কিছু লেখা যাবে না, যতই token থাকুক না কেন।
- এই feature বন্ধ করতে চাইলে `config.php`-তে
  `'allow_client_output_dir' => false` করো — তখন field থাকলেও ignore হবে।
- Base folder ঠিক না হলে (addon domain ইত্যাদি) `config.php`-এর
  `output_base` explicit path দিয়ে override করো।

---

## Multiple Templates — Rotation (নতুন)

**HTML Template** card-এ এখন একাধিক template box রাখা যায়:

- **＋ Add Template** বাটনে চাপলে নতুন খালি template box যোগ হয়।
- প্রতিটা box-এর উপরে ✕ বাটন দিয়ে সেটা মুছে ফেলা যায় (অন্তত ১টা box থাকতেই হবে)।
- Chip (`{{Keywords}}` / `{{Keywords2}}` / `{{IMAGE_URL}}`) click করলে সবশেষ যেই
  template box-এ focus ছিল সেখানেই বসে।

**Generate** চাপলে খালি না-থাকা template গুলো ধরে **round-robin rotation** হয় —
keyword ১ → Template ১, keyword ২ → Template ২, keyword ৩ → Template ৩,
keyword ৪ → আবার Template ১ … এভাবে ঘুরতে থাকে। ১টাই template দিলে (আগের মতো)
সব file একই template থেকে বানে।

```
Template 1, Template 2, Template 3 (৩টা) + 7টা keyword দিলে:
file 1 → Template 1    file 5 → Template 2
file 2 → Template 2    file 6 → Template 3
file 3 → Template 3    file 7 → Template 1
file 4 → Template 1
```

- প্রতিটা template-এ আলাদাভাবে `{{Keywords}}` / `{{Keywords2}}` / `{{IMAGE_URL}}`
  ব্যবহার করা যায় — কোনো template-এ সবকটা থাকতেই হবে না, যেটা লাগবে সেটাই দিলেই হবে।
- অন্তত একটা template-এ কোনো একটা placeholder না থাকলে Generate আটকে যাবে ও
  কোন Template নম্বরে সমস্যা সেটা বলে দেবে।
- Status list-এ generate হওয়া প্রতিটা file-এর পাশে কোন Template ব্যবহার হয়েছে
  (`T1`, `T2` …) সেটা দেখায় — শুধু একাধিক template থাকলে।
- Stats strip-এ উপরে **Templates** সংখ্যা এবং template card-এর title-এর পাশে
  "(N — rotating)" দেখায় যখন ১টার বেশি non-empty template থাকে।

---

## Cancel — যেকোনো সময় (নতুন)

Generate চলাকালীন **✕ Cancel** button (Generate button-এর নিচে + status bar-এ) অথবা **Esc** চাপলে সব একসাথে থামে:

| কোন stage-এ cancel | কী হয় |
|---|---|
| File বানানো | সাথে সাথে থামে — কিছুই download/save হয় না, কয়টা বানানো হয়েছিল দেখায় |
| ZIP build | ZIP বানানো থামে — কিছুই download/save হয় না |
| Server upload | Upload থামে, server-এ অর্ধেক পাঠানো temp file (`.part`) সাথে সাথে মুছে যায় (`action=abort`) |
| Server processing | Browser অপেক্ষা থামায়, তবে server হয়তো ততক্ষণে save শেষ করে ফেলেছে — output folder চেক করো |

PC download একবার হয়ে গেলে সেটা ফেরত নেওয়া যায় না (browser-এর হাতে চলে যায়)।
**নতুন `save.php` লাগবে** upload-cancel cleanup-এর জন্য (পুরনোটাতেও cancel কাজ করবে, শুধু temp file ১ ঘণ্টা পরে মুছবে)।

## Random File Name (নতুন)

**🎲 Random file name** checkbox on করলে প্রতিটা file একটা random নাম পায় (serial number লাগে না):

| Type | উদাহরণ |
|---|---|
| Character only | `kqzmwtpa.html` |
| Number only | `48291736.html` |
| Character + Number | `a7k2x9qd.html` (অন্তত ১টা letter + ১টা digit থাকবেই) |

- **Length** — 3 থেকে 64 (default 8)
- **Letter case** — lowercase / UPPERCASE / MiXeD (Number only-তে প্রযোজ্য নয়)
- **Prefix** দিলে সামনে বসবে → `video-a7k2x9qd.html`; Starting Number এই mode-এ ignore হয়
- এক batch-এ কোনো duplicate হয় না (case-insensitive check, Windows/Mac-এও safe)
- শুধু `a-z A-Z 0-9` ব্যবহার হয় — URL ও server whitelist দুটোতেই safe
- **🎲 Shuffle preview** চাপলে নতুন sample নাম দেখায়; Generate চাপলে প্রতিবারই নতুন নাম
- Keyword mode আর Random mode একসাথে চলে না — একটা on করলে অন্যটা auto off

## File Naming — Prefix + Starting Number

**Starting Number** field-এ দুইভাবে দেওয়া যায়:

```
"001"      → আগের মতোই pure numeric serial: 001, 002, 003 ...
"nhgf21"   → text + trailing digit combo: nhgf21, nhgf22, nhgf23 ...
"abc007"   → nhgf-এর মতোই, কিন্তু 3-digit zero-pad: abc007, abc008 ... abc010, abc011 ...
```

কাজের নিয়ম: string-এর **শেষে যতগুলো digit** আছে সেটাই numeric অংশ ধরা হয় — বাকি
(আগের) সবটুকু static text অংশ, যেটা প্রতিটা generated file-এ **অপরিবর্তিত** থাকে।
প্রতি file-এ শুধু ওই trailing digit-টা 1 করে বাড়ে, আর যতগুলো digit দিয়েছ zero-padding
ততটাই থাকে (`007` → `008` … `010` তিন digit-ই থাকে, 4-digit-এ overflow করলে
zero-pad আর জোর করে কাটা হয় না, e.g. `099`→`100`)।

শুধু text দিলে (কোনো trailing digit ছাড়া, যেমন `nhgf`) auto একটা counter বসে
1 থেকে, zero-pad ছাড়া: `nhgf1`, `nhgf2` ...

**Prefix** field আলাদা, optional — দিলে সামনে `prefix-` বসে যায়:
Prefix=`match`, Starting Number=`nhgf21` → `match-nhgf21.html`, `match-nhgf22.html` ...
Prefix খালি রাখলে শুধু `nhgf21.html`, `nhgf22.html` ...

Illegal filename character (`/ \ : * ? " < > |`) text অংশে থাকলে auto বাদ যায়।

এই feature **Name file from Keyword** checkbox-এর সাথে প্রযোজ্য না — সেটা on
থাকলে Prefix আর Starting Number দুটোই ignore হয়ে যায় (আগের behavior অপরিবর্তিত)।

### Name file from Keyword — Max Characters

**Name file from Keyword** checkbox on করলে filename বসে সেই লাইনের Keyword থেকে।
এর পাশে একটা **Max Characters** field আছে:

```
খালি রাখলে   → পুরো keyword-ই filename হবে, কোনো limit নেই (default)
"50" দিলে    → তার চেয়ে লম্বা keyword 50 character-এ কেটে যাবে, ছোট keyword অপরিবর্তিত থাকবে
```

Space গুলো hyphen (`-`) হয়ে যায়, illegal filename character বাদ যায় (আগের মতোই)।
Max-char cut করার পর দুইটা keyword একই filename-এ চলে গেলে (duplicate keyword বা cut-এর
জায়গায় মিলে যাওয়া) auto `-2`, `-3` বসিয়ে unique রাখা হয়।

### Name file from Keyword — Convert to lowercase

**Name file from Keyword** checkbox-এর ঠিক নিচে একটা **Convert keyword to
lowercase** checkbox আছে (keyword mode-এই কাজ করে)। On করলে filename বানানোর
আগে keyword-টা lowercase হয়ে যায় — HTML content-এ `{{Keywords}}` /
`{{Keywords2}}` জায়গায় বসা টেক্সট অপরিবর্তিত (আসল case-এই) থাকে, শুধু filename-এ
প্রভাব পড়ে।

```
Keyword: "Namibia VS Comoros Live"
Checkbox OFF → Namibia-VS-Comoros-Live.html
Checkbox ON  → namibia-vs-comoros-live.html
```

Off রাখলে আগের behavior অপরিবর্তিত (keyword-এর case যেমন লেখা তেমনই filename-এ বসে)।

---

## Checkbox গুলো কী করে

| Option | কাজ |
|---|---|
| **Save ZIP to folder** | ZIP টা `/post/html-batch.zip` হিসেবে server-এ save হবে |
| **Extract .html live** | ZIP-এর ভিতরের HTML file গুলো folder-এ বের হবে → `example.com/post/match-001.html` সরাসরি live |
| **Download to PC** | আগের মতো browser download (দুটো একসাথেও চলে) |

Token খালি রাখলে server-এ কিছুই যাবে না — শুধু PC download, অর্থাৎ tool টা
আগের মতোই কাজ করবে।

---

## Useful config tweaks (`config.php`)

```php
// প্রতিবার আলাদা zip রাখতে (overwrite এড়াতে)
'timestamp_zip' => true,     // html-batch-20260812-1830.zip

// ZIP আর live HTML আলাদা folder-এ চাইলে
'output_dir' => __DIR__ . '/out',   // → example.com/post/out/

// শুধু নিজের IP থেকে লিখতে দিতে
'allowed_ips' => ['103.120.44.10'],
```

`output_dir` change করলে generator-এর **URL List** box-এও সেই base URL দাও
(`https://example.com/post/out/`), তাহলে `url-list.txt` ঠিক URL বানাবে —
stpindexer-এ সরাসরি paste করা যাবে।

---

## Security notes

`save.php` একটা file-write endpoint, তাই এগুলো built-in আছে:

- **Token auth** — `hash_equals()` দিয়ে timing-safe compare
- **ZIP magic-byte check** — `PK\x03\x04` না হলে reject
- **Filename whitelist** — শুধু `[A-Za-z0-9._-]`, path traversal (`../`) flatten করা
- **Extension whitelist** — শুধু `.html/.htm/.txt` extract হয়, `.php` কখনোই না
- **Protected names** — `index.html`, `save.php`, `config.php`, `.htaccess` overwrite হবে না
- **Limits** — 64MB upload, 2000 file, per-file 5MB
- **`config.php` deny** — `.htaccess` দিয়ে browser থেকে blocked

অতিরিক্ত সুরক্ষা চাইলে folder-এ cPanel **Directory Privacy** (HTTP Basic Auth)
বসিয়ে দাও — তাহলে page টাই public থাকবে না।

---

## Requirements

- PHP 7.4+ (8.x recommended)
- `ext-zip` enabled — Hostinger/cPanel-এ default থাকে
- Static host (GitHub Pages, Netlify, Cloudflare Pages) হলে PHP চলবে না →
  শুধু `index.html` কাজ করবে, ZIP PC-তেই নামবে

---

## Troubleshooting

| Error | কারণ / সমাধান |
|---|---|
| `Invalid token` | config.php-এর token আর page-এর token মিলছে না |
| `Default token still in config.php` | Step 1 করা হয়নি |
| `PHP ext-zip not installed` | hPanel → PHP Configuration → `zip` extension enable |
| `Output dir not writable` | folder permission 755 করো |
| `413 Request Entity Too Large` | `index.html`-এ `CHUNK_BYTES` কমিয়ে `256 * 1024` করো |
| `404 — save.php পাওয়া যায়নি` | `save.php` আর `index.html` একই folder-এ আছে কিনা দেখো |
| `File not found.` (nginx) | PHP-FPM script path পাচ্ছে না — nginx block-এ `fastcgi_param SCRIPT_FILENAME` ঠিক আছে কিনা দেখো |
| `Cannot write to temp dir` | PHP-র `sys_get_temp_dir()` writable না — `open_basedir` চেক করো |
| `403` | server ওই folder-এ PHP execute করতে দিচ্ছে না |

---

## nginx ব্যবহারকারীদের জন্য

`.htaccess` **nginx-এ কাজ করে না**। Chunked upload-এর কারণে
`client_max_body_size` না বাড়ালেও চলবে, তবে server block-এ এটা দিলে
আরও দ্রুত হবে (তখন `CHUNK_BYTES` বড় করা যাবে):

```nginx
location /post/ {
    client_max_body_size 68M;
}

# config.php browser থেকে block (বাড়তি সুরক্ষা)
location = /post/config.php { deny all; }

# ZIP download হিসেবে যাক
location ~ ^/post/.*\.zip$ { add_header Content-Disposition attachment; }
```

PHP-FPM-এর `upload_max_filesize` / `post_max_size` chunk-এর চেয়ে বড় হলেই
যথেষ্ট (default 2M/8M-ই যথেষ্ট, কারণ chunk মাত্র 512KB)।

---

Created by **Tarik Aziz** · Soft-Tech Point
