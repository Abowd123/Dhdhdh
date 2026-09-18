# CivilDraft — Project Setup & Configuration

> ملفات الجذر، الإعدادات، التوثيق، وخادم التطوير.

**عدد الملفات:** 7

---

## `README.md`

```markdown
# CivilDraft

مرسمة مخطّطات معمارية عربية تعمل في المتصفّح بلا اعتماديات.
وحدات ES خام و`<canvas>`، واجهة RTL، والمليمتر وحدةَ التخزين
والمتر وحدةَ الإدخال.

## التشغيل
وحدات ES لا تُحمَّل من `file://`، فشغّله بخادمٍ محلّي:

    npm run serve      # http://localhost:8080

## الاختبارات

    npm test           # كل المجموعات
    npm run check      # فحص صياغة سريع

## البنية
- `js/core/` الهندسة والحالة: جدران، فتحات، مناطق، طبقات،
  أبعاد، مقاطع، واجهات، جداول كميات.
- `js/tools/` الأدوات المسجَّلة في `registry.js`.
- `js/ui/` الشريط والأرصفة وسطر الأوامر والإدخال الحركي.
- `js/io/` تصدير DXF R12 و SVG و PDF و PNG، واستيراد DXF،
  والتخزين المحلّي.
- `js/ai/` مساعدٌ اختياري يتّصل بأي مزوّد متوافق مع OpenAI.

## العقود
- لا يتحرّك إحداثيٌّ إلّا بأمرك: لا لحم تلقائي ولا تقريب صامت.
- دمج الأركان وطرح الفتحات عرضٌ لا تعديل؛ البيانات كما رسمتها.
- المنطقة تُخبَز بأمرك فتصير كائناً مستقلّاً، وتغيّر جدارٍ
  يجعلها «قديمة» حتى تحدّثها.
- المخفيّ لا يُرسَم ولا يُصدَّر، والمقفل يُرى ولا يُلمَس.
- الفاحص يخبر ولا يصلح.

## المساعد والخصوصية
معطّل افتراضياً. الافتراضي مزوّدٌ محلّي (Ollama) فلا يخرج شيء من
جهازك. مع مزوّدٍ خارجي تُرسَل خلاصة المشروع بعد موافقةٍ صريحة تذكر
الوجهة والحجم، ونصوص المرجع المستورد والتأشير لا تُرسَل.
المفتاح لا يُكتَب على القرص إلّا بتفعيل «احفظ المفتاح».

## الرخصة
MIT
```

<a id="f-changes-md"></a>

---

## `CHANGES.md`

````markdown
# CivilDraft — التغييرات المضافة/المعدَّلة (المرحلة 1: نواة الميزات الجديدة)

هذه الحزمة تحتوي الملفات **المضافة** و**المعدَّلة** فقط، مع ملفّات فروقات (diffs)
للملفات المعدَّلة مقابل الأصل.

## الميزات (المرحلة 1 — POC للنواة، مُختبَرة 34/34 + كل اختبارات المشروع خضراء)
1. **الجدران القوسية (Arc walls)** — تمثيل القوس بحقلٍ اختياريّ `bulge`
   (بأسلوب DXF: `tan(θ/4)`). عند `bulge=0` السلوك مطابقٌ للجدار المستقيم تماماً.
2. **الإسقاط ثلاثي الأبعاد (Axonometric)** — رياضيّات خالصة بلا اعتماديات (بلا Three.js).
3. **الأبعاد التلقائية للغرف** — توليد أبعاد الغرفة (عرض×طول) + المساحة من حلقة المنطقة.

> ملاحظة: هذه المرحلة تُثبت النواة هندسياً. دمج الواجهة (زر عرض 3D، أداة رسم القوس،
> زر الأبعاد التلقائية، مكتبة الأثاث) هو المرحلة 2 التالية.

## الملفات المضافة (Added)
- `js/core/proj3d.js` — إسقاط أكسونومتري خالص: `project`, `projScreen`, `projBBox`, `shade`.
- `js/core/autodim.js` — دالّة نقيّة `roomDims(ring)` تُعيد مواصفات الأبعاد والملصق.
- `js/tests/newfeat.test.js` — اختبار Node يغطّي الأقواس + الإسقاط 3D + الأبعاد التلقائية.
- `serving/static-server.js` — (بيئة التشغيل فقط) خادم ثابت خفيف لعرض التطبيق على المنفذ 3000.

## الملفات المعدَّلة (Modified) — انظر مجلّد `diffs/`
- `js/core/walls.js`
  - إضافة: `isArc`, `arcParams`, `arcTess`, `arcLen`, `bulgeFrom3`.
  - `band(w)` صار يبني مضلّع شريطٍ قوسيّاً عند `isArc` (فيمرّ عبر الالتقاط/المناطق/الصناديق تلقائياً).
  - `addWall(..., bulge)` يقبل الانحناء اختيارياً.
- `js/core/state.js`
  - تطبيع `bulge` في `ensureShape()` (قسر ضمن حدّ آمن؛ الصفر يُحذف فيبقى الجدار مستقيماً).
- `js/core/render.js`
  - `bodyOf()` يفصل الجدران القوسية فيبنيها مضلّعاً مباشرةً (band) وتدخل الاتحاد،
    بينما تبقى المستقيمة على مسار المحاور (الدمج/طرح الفتحات) دون تغيير.
- `js/core/opens.js`
  - `addOpen()` يمنع الفتحات على الجدار القوسيّ برسالةٍ واضحة (الفتحات للمستقيم فقط).
- `package.json`
  - إضافة سكربت `test:newfeat` وإدراجه في سلسلة `test` (وبقاء 28 اختباراً + الجديدة خضراء).

## إصلاحاتٌ أُضيفت بعد المراجعة (قبل الدمج)
راجعتُ الحزمة بتطبيقها فعلياً على المشروع وتشغيل `npm test`، فظهرت
علّتان لم تكونا في CHANGES.md الأصلية، وأصلحتُهما مع اختبارَين جديدين:

1. **استيرادٌ ناقص كان يُسقِط أيّ جدارٍ قوسي** — `band()` في
   `walls.js` ينادي `cleanRing()` دون استيرادها. أُضيفت للاستيراد.
   بدونها: `npm test` يفشل فعلياً (كان يخرج بكودٍ غير صفري)، خلافاً
   لِما ذُكر من أن كل الاختبارات خضراء.
2. **`wallLen()` كانت تُرجع الوتر لا طول القوس الفعلي** — فجداول
   الكميات (`boq.js`) وفحص الحدّ الأدنى (`inspect.js`) والواجهات/
   المقاطع كانت ستحسب رقماً أقلّ من الحقيقة بصمتٍ (بلا أي تحذير)،
   لجدارٍ قوسي ربع دائرةٍ نصف قطره 1م الفرقُ نحو 11%. `wallLen`
   الآن تُرجع طول القوس الحقيقي للجدار القوسي.
3. **الواجهات/المقاطع (`elevation.js`) تستبعد الآن الجدار القوسي
   صراحةً** (`projectWall` يعيد `null`) بدل إسقاطه كوترٍ مستقيمٍ
   بشكلٍ خاطئ. الاستبعاد **مُبلَّغ لا صامت**: أُضيفت شفرة فحصٍ
   جديدة `warcnoelev` (ملاحظة) في `inspect.js` تخبر المستخدم أن
   هذا الجدار مستبعدٌ من الواجهات/المقاطع حتى تُبنى في مرحلةٍ قادمة.
4. **ثغرة صغيرة في `serving/static-server.js`**: حارس مسار التصفّح
   كان يقارن بادئةً (`p.startsWith(root)`) فيقبل مجلّداً شقيقاً
   اسمه يبدأ بنفس السلسلة (مثل `/app/mistar-secrets`). أُصلح
   بمقارنةٍ تشمل فاصل المسار.

بعد هذه الإصلاحات: **2069/2069** اختباراً ناجحاً بلا استثناء
(تشمل 42 اختباراً في `newfeat.test.js` بعد إضافة اختبارَين)،
وتغطيةٌ سلوكية 55.1%.

## كيفية التشغيل والاختبار
```bash
npm test            # كل المجموعات (يجب أن تبقى خضراء)
npm run test:newfeat
npm run serve       # عرض محلّي على http://localhost:8080
```
````

<a id="f-package-json"></a>

---

## `package.json`

```json
{
 "name": "civildraft",
 "version": "1.0.0",
 "description": "CivilDraft — مرسمة مخطّطات معمارية عربية تعمل في المتصفّح بلا اعتماديات",
 "type": "module",
 "private": true,
 "license": "MIT",
 "engines": { "node": ">=18" },
 "scripts": {
  "test": "node js/tests/geom.js && node js/tests/core.js && node js/tests/inspect.js && node js/tests/tools.js && node js/tests/ui.js && node js/tests/run.js && node js/tests/trace.js && node js/tests/store.js && node js/tests/dxfin.js && node js/tests/perf.js && node js/tests/boq.test.js && node js/tests/elevation.test.js && node js/tests/section.test.js && node js/tests/dom.js && node js/tests/cover.js && node js/tests/blocks.js && node js/tests/pricing.js && node js/tests/templates.js && node js/tests/golden.js && node js/tests/help.test.js && node js/tests/phase1.test.js && node js/tests/smartblocks.test.js && node js/tests/newfeat.test.js",
  "test:geom": "node js/tests/geom.js",
  "test:core": "node js/tests/core.js",
  "test:inspect": "node js/tests/inspect.js",
  "test:tools": "node js/tests/tools.js",
  "test:ui": "node js/tests/ui.js",
  "test:run": "node js/tests/run.js",
  "test:trace": "node js/tests/trace.js",
  "test:store": "node js/tests/store.js",
  "test:dxf": "node js/tests/dxfin.js",
  "test:perf": "node js/tests/perf.js",
  "test:dom": "node js/tests/dom.js",
  "test:boq": "node js/tests/boq.test.js",
  "test:elev": "node js/tests/elevation.test.js",
  "test:sect": "node js/tests/section.test.js",
  "test:blocks": "node js/tests/blocks.js",
  "test:pricing": "node js/tests/pricing.js",
  "test:templates": "node js/tests/templates.js",
  "test:golden": "node js/tests/golden.js",
  "golden:update": "node js/tests/golden.js --update",
  "test:all": "node js/tests/all.js",
  "test:help": "node js/tests/help.test.js",
  "test:smartblocks": "node js/tests/smartblocks.test.js",
  "test:newfeat": "node js/tests/newfeat.test.js",
  "cover": "node js/tests/cover.js",
  "cover:list": "node js/tests/cover.js --list",
  "serve": "python3 -m http.server 8080",
  "check": "node --check js/app.js && node js/tests/dom.js && node js/tests/geom.js && node js/tests/core.js"
 },
 "dependencies": {},
 "devDependencies": {},
 "keywords": ["architecture","cad","dxf","arabic","rtl","canvas"]
}
```

<a id="f-netlify-toml"></a>

---

## `netlify.toml`

```toml
[build]
  publish = "."

[[headers]]
  for = "/*"
  [headers.values]
    Content-Security-Policy = "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; connect-src 'self' https: http://localhost:* http://127.0.0.1:*; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'"
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "no-referrer"

[[headers]]
  for = "*.js"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "*.css"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

<a id="f-license"></a>

---

## `LICENSE`

```text
MIT License

Copyright (c) 2026 <اسمك>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

<a id="f-index-html"></a>

---

## `index.html`

```html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<!-- ═══ CSP ═══
     نصّ المزوّد يمرّ إلى الواجهة (prose وnotes ورسائل الخطأ التي
     تحمل ١٨٠ حرفاً من جسم ردّه)، والمفتاح مخزَّنٌ في نفس الأصل.
     سطحُ الحقن مغلقٌ في الشفرة، وهذه طبقةٌ ثانية بلا تكلفة:
     لا سكربت مضمَّن في المشروع كلّه.
     'unsafe-inline' للأنماط لازمٌ لأن style="…" يُبنى في ثلاثة
     عشر موضعاً · connect-src مضيّقٌ إلى https وlocalhost لأن عنوان
     المزوّد يختاره المستخدم لكن لا داعي لقبول أي مخطّط أو مضيفٍ
     بعيد غير مشفّر · data: blob: لأن PNG يُدرَج صورةً في PDF
     ويُنزَّل بـblob. -->
<meta http-equiv="Content-Security-Policy" content="
 default-src 'self';
 script-src 'self';
 style-src 'self' 'unsafe-inline';
 img-src 'self' data: blob:;
 connect-src 'self' https: http://localhost:* http://127.0.0.1:*;
 object-src 'none';
 base-uri 'none';
 form-action 'none';
 frame-ancestors 'none'">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>CivilDraft — مخططات معمارية</title>
<link rel="stylesheet" href="css/theme.css">
<link rel="stylesheet" href="css/base.css">
<link rel="stylesheet" href="css/tools.css">
<link rel="stylesheet" href="css/ribbon.css">
<link rel="stylesheet" href="css/dock.css">
<link rel="stylesheet" href="css/status.css">
<link rel="stylesheet" href="css/cmd.css">
 <link rel="stylesheet" href="css/palette.css">
 <link rel="stylesheet" href="css/tour.css">
 <link rel="stylesheet" href="css/helpbot.css">
</head>
<body>

<header id="top">
 <button id="appBtn" aria-haspopup="menu" aria-expanded="false"
  title="قائمة CivilDraft · Alt ثم ٠">CivilDraft
  <span class="kt" hidden>٠</span></button>
 <span id="qat" role="toolbar" aria-label="وصول سريع"></span>
 <span class="dv"></span>
 <span class="ver">مليمتر · لا يتحرّك شيء إلا بأمرك</span>
 <span class="gap"></span>
 <span class="tip">Alt دلائل · Esc يلغي · F1 مساعدة · F7 فحص</span>
</header>

<div id="appMenu" hidden></div>
 <div id="palette" hidden></div>
<div id="ribbon" hidden></div>
<nav id="tools"></nav>
<div id="optbar"></div>

<div id="main">
 <div class="strip" id="stripS" data-zone="s" hidden></div>
 <aside id="side" class="dock" data-zone="s"></aside>
 <div class="dsz" data-rsz="s" role="separator" tabindex="0"
  aria-orientation="vertical" aria-label="عرض العمود الأيمن"
  aria-valuemin="210" aria-valuemax="560" aria-valuenow="312"
  title="اسحب أو استعمل الأسهم"></div>
 <section id="work">
  <div id="stage" dir="ltr"><canvas id="cv" tabindex="0"
    role="img" aria-label="لوحة الرسم — استعمل سطر الإدخال للأوامر"
    >لوحة رسمٍ نقطية. الأوامر كلّها متاحة من سطر الإدخال
    وقوائم الشريط.</canvas>
   <div id="vpLabel"></div>
   <div id="navbar" role="toolbar" aria-label="تنقّل"></div>
   <div id="compass"></div>
   <div id="dynBox" hidden></div>
   <div id="qpCard" hidden dir="rtl">
    <div class="qpH"><span id="qpHead"></span>
     <button type="button" data-act="propsDlg"
      title="افتح لوحة الخصائص" aria-label="افتح لوحة الخصائص"
      >⤢</button>
     <button type="button" id="qpClose" title="إخفاء"
      aria-label="إخفاء الخصائص السريعة">✕</button></div>
    <div id="qpBody"></div>
   </div>
  </div>
  <div id="cmdWrap">
   <div id="cmdline">
    <span id="clPrompt">أداة:</span>
    <input id="clIn" type="text" spellcheck="false" autocomplete="off"
     aria-labelledby="clPrompt"
     placeholder="اكتب إحداثياً — 3,4 · @5,0 · @5<45 · 5 · <45 · ؟ للمساعد">
    <span id="clLive"></span>
    <div id="clSug" hidden></div>
   </div>
   <div id="log" role="log" aria-live="polite" aria-atomic="false"
    aria-label="سجل الرسائل"><div class="ln wr" id="bootMsg"
    >يُحمَّل CivilDraft… إن بقي هذا السطر فالوحدات لم تُحمَّل.
    افتح وحدة التحكّم لترى الملفّ المفقود.</div></div>
  </div>
   <div id="tour" hidden></div>
 </section>
 <div class="dsz" data-rsz="e" role="separator" tabindex="0"
  aria-orientation="vertical" aria-label="عرض العمود الأيسر"
  aria-valuemin="210" aria-valuemax="560" aria-valuenow="270"
  title="اسحب أو استعمل الأسهم" hidden></div>
 <aside id="sideE" class="dock" data-zone="e" hidden></aside>
 <div class="strip" id="stripE" data-zone="e" hidden></div>
</div>

<div id="pPark" hidden></div>
<div id="floats"></div>
<div id="pMenu" hidden role="menu"></div>
<div id="wsMenu" hidden role="menu"></div>

<footer id="status">
 <div id="stItems"></div>
 <div id="osPop" hidden></div>
</footer>

<div id="stMenu" hidden role="menu"></div>
<div id="vMenu" hidden role="menu"></div>
<div id="cmdFloat"></div>
<div id="cMenu" hidden role="menu"></div>
<div id="ctxMenu" hidden role="menu"></div>

<div id="helpBox" hidden role="dialog" aria-modal="true"
 aria-label="مساعدة CivilDraft"></div>

<noscript>
 <div dir="rtl" style="padding:16px;background:#3a1c1c;color:#ffd9d9;
  font:14px/1.7 Tahoma,Arial,sans-serif">
  <b>CivilDraft يحتاج جافاسكربت.</b> الواجهة كلّها تُبنى في المتصفّح،
  ولا خادمَ يرسمها. شغّل جافاسكربت ثم أعِد التحميل.
  <br>وإن كنت تفتح الملفّ من القرص مباشرةً (<code>file://</code>)
  فوحدات ES لا تُحمَّل منه — شغّله بخادمٍ محلّي:
  <code style="direction:ltr;display:inline-block">npm run serve</code>
 </div>
</noscript>

<script type="module" src="js/bootguard.js"></script>
<script type="module" src="js/app.js"></script>
</body>
</html>
```

---

# الخادم (`serving/`)

<a id="f-serving-static-server-js"></a>

---

## `serving/static-server.js`

```javascript
/* Lightweight zero-dependency static server for the "مِسطَر" app.
   Runs under supervisor via `yarn start`. Serves /app/mistar on PORT (3000).
   ES modules require correct MIME types — handled below. */
const http = require("http");
const fs = require("fs");
const path = require("path");

const ROOT = "/app/mistar";
const PORT = parseInt(process.env.PORT, 10) || 3000;
const HOST = process.env.HOST || "0.0.0.0";

const TYPES = {
  ".html": "text/html; charset=utf-8",
  ".js": "text/javascript; charset=utf-8",
  ".mjs": "text/javascript; charset=utf-8",
  ".css": "text/css; charset=utf-8",
  ".json": "application/json; charset=utf-8",
  ".svg": "image/svg+xml",
  ".png": "image/png",
  ".jpg": "image/jpeg",
  ".jpeg": "image/jpeg",
  ".webp": "image/webp",
  ".gif": "image/gif",
  ".ico": "image/x-icon",
  ".woff": "font/woff",
  ".woff2": "font/woff2",
  ".ttf": "font/ttf",
  ".txt": "text/plain; charset=utf-8",
  ".map": "application/json; charset=utf-8",
};

function safeJoin(root, reqPath) {
  const decoded = decodeURIComponent(reqPath.split("?")[0].split("#")[0]);
  const p = path.normalize(path.join(root, decoded));
  // path traversal guard — compare with a trailing separator so a sibling
  // directory that merely shares the root's name as a prefix (e.g.
  // "/app/mistar-secrets") is not mistaken for a path inside root.
  if (p !== root && !p.startsWith(root + path.sep)) return null;
  return p;
}

const server = http.createServer((req, res) => {
  let target = safeJoin(ROOT, req.url === "/" ? "/index.html" : req.url);
  if (!target) {
    res.writeHead(403);
    return res.end("Forbidden");
  }
  fs.stat(target, (err, st) => {
    if (!err && st.isDirectory()) target = path.join(target, "index.html");
    fs.readFile(target, (err2, data) => {
      if (err2) {
        // SPA-ish fallback to index.html for unknown non-asset routes
        if (!path.extname(target)) {
          return fs.readFile(path.join(ROOT, "index.html"), (e3, d3) => {
            if (e3) {
              res.writeHead(404);
              return res.end("Not found");
            }
            res.writeHead(200, { "Content-Type": TYPES[".html"] });
            res.end(d3);
          });
        }
        res.writeHead(404);
        return res.end("Not found");
      }
      const ext = path.extname(target).toLowerCase();
      res.writeHead(200, {
        "Content-Type": TYPES[ext] || "application/octet-stream",
        "Cache-Control": "no-cache",
      });
      res.end(data);
    });
  });
});

server.listen(PORT, HOST, () => {
  console.log(`[mistar-static] serving ${ROOT} on http://${HOST}:${PORT}`);
});
```

---

# الأنماط (`css/`)

<a id="f-css-base-css"></a>

---

