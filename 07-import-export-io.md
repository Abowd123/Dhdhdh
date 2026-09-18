# CivilDraft — Import/Export (IO)

> استيراد وتصدير الملفات: DXF ،PDF ،SVG ،PNG ،CSV، وحفظ المشروع.

**عدد الملفات:** 15

---

## `js/io/boq.js`

```javascript
/* ═══ تصدير جدول الكميات — CSV ═══
   نصٌّ صريح يفتحه أي جدولٍ حسابيّ، وأصدقُ ما يُصدَّر إليه رقمٌ
   يُجمَع لا نصٌّ يُقرَأ. فالأرقام تخرج بالنقطة العشرية لا بالفاصلة
   العربية: Excel وLibreOffice يقرآن «12.34» رقماً و«١٢٫٣٤» نصّاً
   لا يُجمَع — والعناوين عربيةٌ لأنها تُقرَأ، والقيَم إنجليزيةٌ
   لأنها تُحسَب.

   والفاصلُ فاصلةٌ لا منقوطة: RFC 4180 هو العقد، والحقلُ الذي
   يحوي فاصلةً يُقتبَس. ولا تُبدَّل بمنقوطةٍ إرضاءً لإعدادٍ محلّيّ
   في نسخةٍ واحدة من برنامجٍ واحد — فالملفّ يُقرَأ في مكانٍ لا
   نعرفه.

   وBOM في أوّل الملفّ: بدونه يقرأ Excel على ويندوز العربيةَ
   بترميزٍ محلّيّ فتخرج رموزاً. محرفٌ واحد يمنع عطباً لا يُشخَّص.

   والوحدات في العنوان لا في الخليّة: «المساحة (م²)» ثم رقمٌ
   مجرَّد — لا «12.34 م²» فيصير الحقلُ نصّاً في كل صفٍّ من ألف. */
import {sqm,m2,m3} from "../core/units.js";

const BOM="\ufeff";
/* حقلُ CSV: يُقتبَس إن حوى فاصلةً أو اقتباساً أو سطراً — والاقتباس
   يُضاعَف داخل الاقتباس، وهو نصُّ المعيار لا اجتهاداً */
const Q=v=>{
 const s=String(v==null?"":v);
 return /[",\n\r]/.test(s) ? `"${s.replace(/"/g,'""')}"` : s;
};
const row=a=>a.map(Q).join(",");
/* الأرقام: المليمتر يخرج بوحدته المقروءة، بالنقطة لا بالفاصلة.
   وsqm وm2 وm3 يعيدون نصّاً بالنقطة سلفاً (toFixed) — فلا
   تحويلَ ثانٍ يفترق عنها. */
const A2=v=>sqm(v);          /* مم² → م² بمنزلتين */
const L2=v=>m2(v);           /* مم  → م  بمنزلتين */
const L3=v=>m3(v);           /* مم  → م  بثلاث    */
const V3=v=>((+v||0)/1e9).toFixed(3);   /* مم³ → م³ */

export function toCSV(B,opt){
 const O=Object.assign({sep:"\n"},opt||{});
 const L=[];
 const put=a=>L.push(row(a));
 const gap=()=>L.push("");

 /* ═══ الترويسة ═══ */
 put(["جدول الكميات",B.name]);
 put(["المقياس",`1:${B.scale}`]);
 if(B.date)put(["التاريخ",B.date]);
 put(["ارتفاع الجدار (م)",L2(B.wallH)]);
 gap();

 /* ═══ المناطق ═══
    الملاحظة عمودٌ صريح: القديمةُ تُعَدّ ولا تُخفى، والمجموع
    يشملها — فهو صادقٌ عن الحالة كما هي لا كما يُرجى. */
 put(["المناطق"]);
 put(["المعرّف","الاسم","المساحة (م²)","المحيط (م)","ملاحظة"]);
 B.areas.rows.forEach(r=>{
  put([r.id,r.name,A2(r.area),L2(r.perim),r.stale?"قديمة":""]);
 });
 put(["","المجموع",A2(B.areas.total),"",
  B.areas.stale?`${B.areas.stale} قديمة`:""]);
 gap();

 /* ═══ الفتحات ═══
    المدى يُذكَر بعمودَين لا بنصٍّ «900–1200»: عمودان يُفرَزان
    ويُجمَعان، والنصُّ يُقرَأ ولا يُحسَب. */
 put(["الفتحات"]);
 put(["النوع","العدد","أصغر عرض (م)","أكبر عرض (م)",
  "أصغر ارتفاع (م)","أكبر ارتفاع (م)","المساحة (م²)","المصاريع"]);
 B.opens.rows.forEach(r=>{
  put([r.name,r.n,L2(r.wMin),L2(r.wMax),
   L2(r.hMin),L2(r.hMax),A2(r.ar),r.pan||""]);
 });
 put(["المجموع",B.opens.total,"","","","",A2(B.opens.ar),""]);
 gap();

 /* ═══ الجدران ═══
    «سماكات» عمودٌ يقول إن كان «الأشيع» يخفي غيرَه: قيمةٌ أكبر
    من واحدٍ تعني أن الحجم تقديرٌ لا قياس. تُقال ولا تُصلَح. */
 put(["الجدران"]);
 put(["النوع","العدد","الطول (م)","السماكة الأشيع (م)","سماكات",
  "الارتفاع (م)","مساحة الوجه (م²)","الحجم (م³)","فتحات"]);
 B.walls.rows.forEach(r=>{
  put([r.name,r.n,L2(r.len),L3(r.t),r.tn,
   L2(r.h),A2(r.face),V3(r.vol),r.opens||""]);
 });
 put(["المجموع",B.walls.n,L2(B.walls.len),"","","",
  A2(B.walls.face),V3(B.walls.vol),""]);

 return {txt:BOM+L.join(O.sep)+O.sep,
  lines:L.length,
  notes:noteOf(B)};
}
/* ═══ الملاحظات ═══
   ما لا يقوله الجدولُ صراحةً يُقال هنا، فيبلغ لوحةَ الحالة —
   كما تفعل notes في io/svg.js. */
export function noteOf(B){
 const n=[];
 if(B.areas.stale)
  n.push(`${B.areas.stale} منطقةً قديمة دخلت المجموع — `
   +`حدّثها بأمر arearef قبل الاعتماد عليه`);
 B.walls.rows.forEach(r=>{
  if(r.tn>1)
   n.push(`${r.name}: ${r.tn} سماكات مختلفة — `
    +`الحجم محسوبٌ بالأشيع (${m3(r.t)} م)`);
 });
 if(B.opens.total)
  n.push("أطوال الجدران ومساحاتها لا تُطرَح منها الفتحات");
 if(!B.areas.n&&!B.walls.n&&!B.opens.total)
  n.push("المشروع فارغ");
 return n;
}
/* اسمُ الملفّ من اسم المشروع — بلا محارف تُعطِب مساراً */
export const csvName=B=>String(B.name||"PLAN")
 .replace(/[\\/:*?"<>|]+/g,"_").slice(0,40).trim()
 .replace(/\s+/g,"_")+"-BOQ.csv";
```

<a id="f-js-io-boqcsv-js"></a>

---

## `js/io/boqcsv.js`

```javascript
/* ═══ تصدير حصر الكميات المسعّر إلى CSV متوافق مع Excel العربي ═══ */
const q = v => {
  const s = String(v == null ? "" : v);
  return /[",\n\r]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
};
export function toCSV(rows, columns) {
  const head = columns.map(c => q(c.title)).join(",");
  const body = (rows || []).map(r => columns.map(c => q(r[c.key])).join(",")).join("\r\n");
  return "\uFEFF" + head + (body ? "\r\n" + body : "");
}
export function boqToCSV(priced) {
  const cols = [
    { key: "label", title: "البند" }, { key: "unit", title: "الوحدة" },
    { key: "qty", title: "الكمية" }, { key: "rate", title: "سعر الوحدة" },
    { key: "amount", title: "الإجمالي" }
  ];
  const rows = priced.rows.slice();
  rows.push({});
  rows.push({ label: "المجموع الفرعي", amount: priced.subtotal });
  rows.push({ label: `ضريبة (${Math.round(priced.taxRate * 100)}%)`, amount: priced.tax });
  rows.push({ label: "الإجمالي النهائي", amount: priced.total });
  return toCSV(rows, cols);
}
export function download(filename, text, mime = "text/csv;charset=utf-8") {
  const blob = new Blob([text], { type: mime }), url = URL.createObjectURL(blob);
  const a = document.createElement("a"); a.href = url; a.download = filename;
  document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 1000);
}
```

<a id="f-js-io-cp1256-js"></a>

---

## `js/io/cp1256.js`

```javascript
/* ═══ ترميز CP1256 ═══
   DXF R12 لا يحمل وسم ترميزٍ لكل نصّ: الملفّ كلّه بصفحة رمزٍ
   واحدة يُعلنها $DWGCODEPAGE. وكان المُصدِّر يُعلن ANSI_1256
   ويكتب بايتات UTF-8 (‏Blob يُرمِّز السلاسل بها دائماً) — فالإعلان
   والبايتات متناقضان، والنصّ العربي يصل أوتوكاد خربشةً.
   والدورة الداخلية كانت سليمةً ومغلقةً على نفسها لأن decodeDXF
   يجرّب UTF-8 أوّلاً، فالعلّة لا تظهر إلّا عند من يفتح ملفّك.

   الجدول للنطاق 0x80–0xFF وحده؛ ما دون 0x80 مطابقٌ لـASCII.
   وما لا يُرمَّز يصير «؟» ويُعَدّ، فيُقال للمستخدم لا يُكتَم. */

/* موضع البايت 0x80+i ⇒ نقطة يونيكود */
const HI=[
 0x20AC,0x067E,0x201A,0x0192,0x201E,0x2026,0x2020,0x2021,
 0x02C6,0x2030,0x0679,0x2039,0x0152,0x0686,0x0698,0x0688,
 0x06AF,0x2018,0x2019,0x201C,0x201D,0x2022,0x2013,0x2014,
 0x06A9,0x2122,0x0691,0x203A,0x0153,0x200C,0x200D,0x06BA,
 0x00A0,0x060C,0x00A2,0x00A3,0x00A4,0x00A5,0x00A6,0x00A7,
 0x00A8,0x00A9,0x06BE,0x00AB,0x00AC,0x00AD,0x00AE,0x00AF,
 0x00B0,0x00B1,0x00B2,0x00B3,0x00B4,0x00B5,0x00B6,0x00B7,
 0x00B8,0x00B9,0x061B,0x00BB,0x00BC,0x00BD,0x00BE,0x061F,
 0x06C1,0x0621,0x0622,0x0623,0x0624,0x0625,0x0626,0x0627,
 0x0628,0x0629,0x062A,0x062B,0x062C,0x062D,0x062E,0x062F,
 0x0630,0x0631,0x0632,0x0633,0x0634,0x0635,0x0636,0x00D7,
 0x0637,0x0638,0x0639,0x063A,0x0640,0x0641,0x0642,0x0643,
 0x00E0,0x0644,0x00E2,0x0645,0x0646,0x0647,0x0648,0x00E7,
 0x00E8,0x00E9,0x00EA,0x00EB,0x0649,0x064A,0x00EE,0x00EF,
 0x064B,0x064C,0x064D,0x064E,0x00F4,0x064F,0x0650,0x00F7,
 0x0651,0x00F9,0x0652,0x00FB,0x00FC,0x200E,0x200F,0x06D2];

let MAP=null;
const map=()=>{
 if(MAP)return MAP;
 MAP=new Map();
 for(let i=0;i<HI.length;i++)MAP.set(HI[i],0x80+i);
 /* أشباهٌ مقبولة: محارف عزل الاتجاه (الدفعة ٢) لا وجود لها في
    الصفحة، وهي غير مرئية — فتُطرَح لا تُبدَّل بعلامة استفهام.
    وعلاماتُ الترقيم العربية موجودةٌ في الجدول أصلاً. */
 [0x2066,0x2067,0x2068,0x2069,0xFEFF].forEach(c=>MAP.set(c,-1));
 return MAP;
};
/* نصٌّ ⇒ بايتات · تُعيد {bytes, bad} — bad عددُ ما لم يُرمَّز.
   الطول ×2 لأن ما فوق BMP زوجُ وحداتٍ في JS ويصير بايتاً واحداً
   («؟») هنا، فلا يفيض. */
export function encode(str){
 const M=map(), s=String(str==null?"":str);
 const out=new Uint8Array(s.length*2+8);
 let n=0, bad=0;
 for(const ch of s){
  const c=ch.codePointAt(0);
  if(c<0x80){out[n++]=c; continue}
  const b=M.get(c);
  if(b===-1)continue;              /* يُطرَح بلا أثر */
  if(b!==undefined){out[n++]=b; continue}
  out[n++]=0x3F;                   /* ؟ */
  bad++;
 }
 return {bytes:out.subarray(0,n), bad};
}
export const canEncode=str=>encode(str).bad===0;

/* الفكّ — لبيئةٍ بلا TextDecoder (Node العاري في المِعمَل) وللدورة
   الكاملة في الاختبار. والمتصفّح يستعمل TextDecoder المدمج. */
export function decode(bytes){
 const b=(bytes instanceof ArrayBuffer)?new Uint8Array(bytes):bytes;
 let s="";
 for(let i=0;i<b.length;i++){
  const v=b[i];
  s+=(v<0x80)?String.fromCharCode(v)
   :String.fromCodePoint(HI[v-0x80]);
 }
 return s;
}
export const CP=1256;
```

<a id="f-js-io-dxf-js"></a>

---

## `js/io/dxf.js`

```javascript
/* ═══ كاتب DXF يدوياً ═══
   يقرأ أوّليات المشهد نفسها التي تُرسَم على الشاشة، فلا يفترق
   المُصدَّر عن المعروض. والهيئة من io/style.js — مصدرٌ واحد يقرأه
   الأربعة.

   ثلاثة قرارات مصرَّح بها:
   · الشرطة المتقطّعة تُكسَر إلى قطعٍ حقيقية بأطوالها المرسومة، بدل
     تعريف LTYPE بأنماط — أثقل ملفّاً وأصدق تمثيلاً. وهي تُطبَّق
     لشرطة الطبقة أيضاً بعد اليوم: كانت تُهمَل هنا وتُطبَّق في SVG
     وPNG، فالمخرَجان يختلفان في الشكل نفسه.
   · الهاشور يُولَّد خطوطاً مقصوصة على الحلقات، لأن HATCH ليست في
     R12 ولا نكتبها في R2000. النتيجة هندسة صريحة يقرأها كل برنامج.
   · الإصدار AC1015 لا AC1009: كنّا نكتب وزن الخطّ (370) و$INSUNITS
     وهما بعد R12، فالإعلان كان أقدم من المحتوى. */
/* الجدولُ الحيُّ لا المصنع: لونٌ يضبطه المستخدم كان يظهر
   على الشاشة ولا يصل الملفّ. */
import {S} from "../core/state.js";
import {resolve,plots} from "../core/layers.js";
import {styleOf,fillOf,hatchOf,hatchLines,ctxOf} from "./style.js";
import {encode} from "./cp1256.js";
export {hatchLines};          /* من كان يستوردها من هنا يبقى عاملاً */

const F=(v,n)=>{
 const x=(+v||0).toFixed(n==null?4:n);
 return (/^-0\.0+$/.test(x))?x.slice(1):x;
};
const L2=(c,v)=>`${c}\n${v}\n`;
/* لونٌ حقيقيّ لـ420: ACI ٢٥٦ لوناً لا تحمل ما يختاره المستخدم */
const rgb24=css=>{
 const s=String(css||"#000000").replace("#","");
 const n=(s.length===3)
  ? s.split("").map(c=>parseInt(c+c,16))
  : [parseInt(s.slice(0,2),16),parseInt(s.slice(2,4),16),
     parseInt(s.slice(4,6),16)];
 const v=n.map(x=>isFinite(x)?Math.max(0,Math.min(255,x)):0);
 return ((v[0]<<16)|(v[1]<<8)|v[2]);
};

/* ═══ الشرطة → قطع حقيقية ═══ خاصّةٌ بـDXF فتبقى هنا ═══ */
export function dashSegs(a,b,pat){
 const P=(pat||[]).map(v=>Math.max(1,+v||0)).filter(v=>v>0);
 if(!P.length)return [[a,b]];
 const dx=b[0]-a[0], dy=b[1]-a[1], L=Math.hypot(dx,dy);
 if(L<1)return [[a,b]];
 const ux=dx/L, uy=dy/L, out=[];
 let s=0, i=0, on=true, guard=0;
 while(s<L&&guard++<20000){
  const d=Math.min(P[i%P.length],L-s);
  if(on&&d>0.4)out.push([
   [a[0]+ux*s, a[1]+uy*s],
   [a[0]+ux*(s+d), a[1]+uy*(s+d)]]);
  s+=d; i++; on=!on;
 }
 return out.length?out:[[a,b]];
}
/* ═══ كيانات DXF ═══ */
const eLine=(lay,a,b)=>L2(0,"LINE")+L2(8,lay)
 +L2(10,F(a[0]))+L2(20,F(a[1]))+L2(30,"0.0")
 +L2(11,F(b[0]))+L2(21,F(b[1]))+L2(31,"0.0");

const ePoly=(lay,pts,closed)=>{
 let s=L2(0,"POLYLINE")+L2(8,lay)+L2(66,"1")
  +L2(10,"0.0")+L2(20,"0.0")+L2(30,"0.0")
  +L2(70,closed?"1":"0");
 pts.forEach(p=>{s+=L2(0,"VERTEX")+L2(8,lay)
  +L2(10,F(p[0]))+L2(20,F(p[1]))+L2(30,"0.0")});
 return s+L2(0,"SEQEND")+L2(8,lay);
};
const eArc=(lay,cx,cy,r,a0,a1)=>{
 const sw=((a1-a0)%360+360)%360;
 if(sw<0.05||sw>359.95)
  return L2(0,"CIRCLE")+L2(8,lay)
   +L2(10,F(cx))+L2(20,F(cy))+L2(30,"0.0")
   +L2(40,F(Math.max(0.01,r)));
 return L2(0,"ARC")+L2(8,lay)
  +L2(10,F(cx))+L2(20,F(cy))+L2(30,"0.0")
  +L2(40,F(Math.max(0.01,r)))+L2(50,F(a0,3))+L2(51,F(a1,3));
};
/* 72: 0 يسار · 1 وسط · 2 يمين   |   73: 0 قاعدة · 2 وسط */
const JU={bl:[0,0],bc:[1,0],br:[2,0],ml:[0,2],mc:[1,2],mr:[2,2]};
const eText=(lay,g)=>{
 const j=JU[g.al]||JU.bc;
 let s=L2(0,"TEXT")+L2(8,lay)
  +L2(10,F(g.x))+L2(20,F(g.y))+L2(30,"0.0")
  +L2(40,F(Math.max(1,g.h)))
  +L2(1,String(g.s).replace(/[\r\n]+/g," "))
  +L2(7,"STANDARD");
 if(g.rot)s+=L2(50,F(g.rot,3));
 if(j[0]||j[1]){
  s+=L2(72,String(j[0]));
  s+=L2(11,F(g.x))+L2(21,F(g.y))+L2(31,"0.0");
  if(j[1])s+=L2(73,String(j[1]));
 }
 return s;
};
/* ═══ التحويل من أوّلية إلى كيانات ═══
   الهيئة من المُحلّ: st.cut يعني «طبِّق الشرطة بالتقطيع»، فالقرار
   في style.js لا هنا — وكان النصّ يقرأ resolve بنفسه فيختلف عن
   الإعلان الذي يُبلَّغ للمستخدم. */
function emit(g,SO){
 const lay=g.L||"0";
 if(g.t==="hatch"){
  const hs=hatchOf(g,"dxf",SO);
  if(hs.skip)return "";
  const parts=[];
  hs.sets.forEach(([ang,d])=>{
   const H2=hatchLines(g.loops,ang,d);
   SO.hatchCut=(SO.hatchCut||0)+H2.cut;
   H2.lines.forEach(q=>{parts.push(eLine(lay,q[0],q[1]))});
  });
  return parts.join("");
 }
 if(g.t==="fill"){
  const f=fillOf(g,"dxf",SO);
  if(f.skip)return "";
  const r=g.ring||[];
  if(r.length<3)return "";
  if(f.hatch){
   /* المنطقة المهشَّرة: خطوطٌ مولَّدة كالثلاثة الآخرين — وكان
      يُصدَّر حدُّها وحده فتختفي تعبئتها من DXF دون غيره */
   const H2=hatchLines([r],45,f.sp);
   SO.hatchCut=(SO.hatchCut||0)+H2.cut;
   return H2.lines.map(q=>eLine(lay,q[0],q[1])).join("");
  }
  /* الصبغة الشفافة: حدُّها يُصدَّر خطاً — والعجز مُعلَنٌ في notes */
  return ePoly(lay,r,1);
 }
 const st=styleOf(g,"dxf",SO);
 if(st.skip)return "";
 const dash=(st.dash&&st.dash.length&&st.cut)?st.dash:null;
 if(g.t==="line"){
  if(dash)return dashSegs(g.a,g.b,dash)
   .map(s=>eLine(lay,s[0],s[1])).join("");
  return eLine(lay,g.a,g.b);
 }
 if(g.t==="poly"){
  const pts=g.pts||[];
  if(pts.length<2)return "";
  if(!dash)return ePoly(lay,pts,g.cl!==0);
  const parts=[];
  const n=(g.cl!==0)?pts.length:pts.length-1;
  for(let i=0;i<n;i++)
   dashSegs(pts[i],pts[(i+1)%pts.length],dash)
    .forEach(q=>{parts.push(eLine(lay,q[0],q[1]))});
  return parts.join("");
 }
 if(g.t==="arc")return eArc(lay,g.cx,g.cy,g.r,g.a0,g.a1);
 if(g.t==="text")return eText(lay,g);
 return "";
}
/* ═══ الملفّ الكامل ═══ */
export function toDXF(prims,bbox,opt){
 const O=opt||{};
 const k=Math.max(1,S.meta.scale);
 /* px=1: DXF يعمل في وحدات النموذج · showWarn=0: التسليم للعميل
    لا يحمل ألوان تشخيص، وCAPS.dxf.warn=0 على أي حال */
 const SO=ctxOf("dxf",{dark:0,k,px:1,minLw:0,
  notes:O.notes||[], showWarn:false, hatchCut:0,
  hs:Math.max(8,(S.meta.txtMM*k)*2.5)});
 /* طبقةٌ لا يُطبَع منها شيءٌ لا تُعلَن: emit يُسقِط أوّلياتها
    فيخرج إعلانٌ لطبقةٍ فارغة. */
 const used=new Set();
 (prims||[]).forEach(g=>{
  const n=g.L||"0";
  if(plots(n))used.add(n);
 });
 const B=bbox||{x0:0,y0:0,x1:1000,y1:1000};
 let s="";
 /* الترويسة */
 s+=L2(0,"SECTION")+L2(2,"HEADER")
  /* AC1015 (R2000) لا AC1009 (R12): وزن الخطّ (370) و$INSUNITS
     كلاهما بعد R12، فالإعلان كان أقدم من المحتوى. والبنية نفسها
     مقبولةٌ في R15 حرفاً بحرف — ولا HATCH فيها على أي حال،
     فالقرار المُعلَن (هاشورٌ خطوطاً) قائم. */
  +L2(9,"$ACADVER")+L2(1,"AC1015")
  /* البايتات بهذه الصفحة فعلاً — انظر toDXFBytes.
     تمريرُ نصٍّ إلى Blob كان يجعل الإعلان كاذباً والعربية
     خربشةً في أوتوكاد. */
  +L2(9,"$DWGCODEPAGE")+L2(3,"ANSI_1256")
  +L2(9,"$INSUNITS")+L2(70,"4")
  +L2(9,"$INSBASE")+L2(10,"0.0")+L2(20,"0.0")+L2(30,"0.0")
  +L2(9,"$EXTMIN")+L2(10,F(B.x0))+L2(20,F(B.y0))+L2(30,"0.0")
  +L2(9,"$EXTMAX")+L2(10,F(B.x1))+L2(20,F(B.y1))+L2(30,"0.0")
  +L2(9,"$LUNITS")+L2(70,"2")
  +L2(9,"$LTSCALE")+L2(40,"1.0")
  +L2(9,"$TEXTSTYLE")+L2(7,"STANDARD")
  +L2(0,"ENDSEC");
 /* الجداول */
 s+=L2(0,"SECTION")+L2(2,"TABLES");
 s+=L2(0,"TABLE")+L2(2,"LTYPE")+L2(70,"1")
  +L2(0,"LTYPE")+L2(2,"CONTINUOUS")+L2(70,"0")
  +L2(3,"Solid line")+L2(72,"65")+L2(73,"0")+L2(40,"0.0")
  +L2(0,"ENDTAB");
 s+=L2(0,"TABLE")+L2(2,"LAYER")+L2(70,String(used.size+1))
  +L2(0,"LAYER")+L2(2,"0")+L2(70,"0")+L2(62,"7")
  +L2(6,"CONTINUOUS");
 used.forEach(n=>{
  if(n==="0")return;
  /* الجدولُ الحيُّ لا المصنع — وكان styleOf يقرأ الحيَّ وهذا
     الجدولُ يقرأ المصنع، فينجرفان لحظةَ يُعدَّل لونٌ أو وزن. */
  const r=resolve(n,"plot");
  /* 70 بت ٤ = مقفلة: تُرى ولا تُلمَس عند من يفتح رسمك.
     والشرطةُ تبقى CONTINUOUS هنا لأننا نقطّعها قطعاً حقيقية
     (CAPS.dxf.cut) — فلو أعلنّا نمطاً لطُبِّق مرّتين.
     و370 سالبةٌ ثلاثاً تعني «افتراضيّ» في DXF، وهو معنى صفرٍ
     في جدولنا (LWS[0] = افتراضي). */
  s+=L2(0,"LAYER")+L2(2,n)+L2(70,r.lk?"4":"0")
   +L2(62,String(r.aci||7))
   +L2(420,String(rgb24(r.css)))
   +L2(6,"CONTINUOUS")
   +L2(370,(r.lw|0)?String(r.lw|0):"-3");
 });
 s+=L2(0,"ENDTAB");
 s+=L2(0,"TABLE")+L2(2,"STYLE")+L2(70,"1")
  +L2(0,"STYLE")+L2(2,"STANDARD")+L2(70,"0")
  +L2(40,"0.0")+L2(41,"1.0")+L2(50,"0.0")+L2(71,"0")
  +L2(42,"2.5")+L2(3,"txt")+L2(4,"")
  +L2(0,"ENDTAB");
 s+=L2(0,"ENDSEC");
 /* الكيانات — join لا s+=: لوحةٌ فيها هاشور تُخرِج مئات الآلاف
    من الأسطر، والإضافة المتكرّرة تبني سلسلةً عملاقة نسخةً نسخة */
 s+=L2(0,"SECTION")+L2(2,"ENTITIES");
 const parts=[];
 (prims||[]).forEach(g=>{const e=emit(g,SO); if(e)parts.push(e)});
 s+=parts.join("")+L2(0,"ENDSEC")+L2(0,"EOF");
 if(O.notes&&SO.hatchCut)O.hatchCut=SO.hatchCut;
 return s;
}
/* ═══ الملفّ بايتاتٍ ═══
   الترويسة تُعلن ANSI_1256، فالبايتات يجب أن تكون بها. وBlob
   يُرمِّز السلاسل UTF-8 دائماً، فلو مرّرنا نصّاً لكان الإعلان
   كاذباً والعربية خربشةً في أوتوكاد — والدورة الداخلية سليمةٌ
   ومغلقةٌ على نفسها فلا تظهر العلّة إلّا حين يفتح رسمَك غيرك.
   ويُعاد عددُ ما لم يُرمَّز وما فُقِد من الهيئة فيُقال للمستخدم. */
export function toDXFBytes(prims,bbox){
 const notes=[];
 const O={notes};
 const txt=toDXF(prims,bbox,O);
 const r=encode(txt);
 return {bytes:r.bytes, bad:r.bad, chars:txt.length, notes,
  hatchCut:O.hatchCut||0};
}
export const dxfStats=prims=>{
 const by={};
 (prims||[]).forEach(g=>{by[g.t]=(by[g.t]||0)+1});
 return by;
};
```

<a id="f-js-io-dxfin-js"></a>

---

## `js/io/dxfin.js`

```javascript
/* ═══ قارئ DXF ═══
   يقرأ R12 وما بعده قراءةً متسامحة، ويعيد أوّليات مرجعية بالمليمتر.
   لا يستنتج جداراً ولا سماكةً ولا محاذاة: يعيد ما في الملفّ.
   وما لا يفهمه يعدّه بنوعه ولا يخترع له هيئة.

   وهو المنفذ الوحيد الذي يستقبل ملفّاً من طرفٍ ثالث، فحرسُه
   ثلاثيّ: حدُّ عملٍ عامّ يمنع التعليق، وسقفُ رؤوسٍ لكل كيان،
   ومدىً نموذجيّ يُنبَذ ما خرج عنه. */
import {clamp,deg,D2R,R2D} from "../core/units.js";
import {decode as cpDecode} from "./cp1256.js";

const R=v=>Math.round(v);
export const MAXENT=60000;
export const MAXOPS=4000000;   /* حدُّ عملٍ عامّ لا حدُّ مستوى */
export const MAXPTS=20000;     /* أكبر عددِ رؤوسٍ لكيانٍ واحد */
export const MAXSEC=20000;     /* أقصى زمنٍ للتحليل بالمللي */

/* ═══ المدى النموذجي ═══
   ±١٠⁹ مم = ألف كيلومتر. ما خرج عنه ليس هندسةً معمارية بل
   قيمةٌ معطوبة أو وحدةٌ خاطئة، وتمريرها يُفسِد الصندوق فيصفّر
   التكبير وتخلو الشاشة بلا رسالة.
   والفحص يقع بعد ضرب معامل الوحدة لا قبله: ملفٌّ بالأقدام
   قيمتُه 1e8 يصير 3e10 مم — صالحٌ خامّاً وشاذٌّ محوَّلاً. */
export const MAXCO=1e9;
const fin=v=>(typeof v==="number"&&isFinite(v));
const okPt=p=>Array.isArray(p)&&fin(p[0])&&fin(p[1])
 &&Math.abs(p[0])<=MAXCO&&Math.abs(p[1])<=MAXCO;

/* الوحدة → معامل التحويل إلى مليمتر */
export const UNITS={
 0:["مجهولة",1], 1:["بوصة",25.4], 2:["قدم",304.8],
 3:["ميل",1609344], 4:["مليمتر",1], 5:["سنتيمتر",10],
 6:["متر",1000], 8:["ميكرون",0.001], 14:["دسيمتر",100]};

/* ═══ الترميز ═══
   $DWGCODEPAGE أوّلاً إن أعلنه الملفّ: نصٌّ عربيٌّ قصير بـCP1256 قد
   يكون صالح البايتات في UTF-8 فيُفَكّ خطأً صامتاً — والعلّة لا
   تظهر إلّا حين تقرأ رسمَ غيرك. وإن لم يُعلن: UTF-8 صارماً ثم
   CP1256، وهو الشائع في الملفّات العربية القديمة. */
const CPRE=/ANSI_1256|CP1256|WINDOWS-1256/i;
function headCP(txt){
 /* الترويسة أوّل الملفّ — ٦٠ ألف حرفٍ تكفي وتزيد */
 const h=txt.slice(0,60000);
 const i=h.indexOf("$DWGCODEPAGE");
 if(i<0)return "";
 const seg=h.slice(i,i+120);
 const m=/\n\s*3\s*\r?\n\s*([^\r\n]+)/.exec(seg);
 return m?m[1].trim():"";
}
export function decodeDXF(buf){
 const b=(buf instanceof ArrayBuffer)?new Uint8Array(buf):buf;
 let head="";
 for(let i=0;i<Math.min(22,b.length);i++)
  head+=String.fromCharCode(b[i]);
 if(/AutoCAD Binary/.test(head))
  throw new Error("ملفّ DXF ثنائي — احفظه نصّياً (ASCII DXF)");
 const cp1256=()=>{
  if(typeof TextDecoder!=="undefined")
   return new TextDecoder("windows-1256").decode(b);
  return cpDecode(b);            /* Node العاري في المِعمَل */
 };
 let u8=null;
 try{
  u8=(typeof TextDecoder!=="undefined")
   ? new TextDecoder("utf-8",{fatal:true}).decode(b)
   : Buffer.from(b).toString("utf8");
 }catch(e){}
 if(u8==null)return {txt:cp1256(),enc:"CP1256"};
 /* البايتات صالحةٌ في UTF-8 — لكن الإعلان يحكم */
 if(CPRE.test(headCP(u8)))return {txt:cp1256(),enc:"CP1256"};
 return {txt:u8,enc:"UTF-8"};
}
/* ═══ أزواج (رمز، قيمة) ═══ */
export function pairs(txt){
 const L=String(txt).split(/\r\n|\r|\n/), P=[];
 let i=0;
 while(i+1<L.length){
  const c=parseInt(L[i].trim(),10);
  if(!isFinite(c)){i++; continue}
  P.push([c,L[i+1]]);
  i+=2;
 }
 return P;
}
const NUM=(g,c,d)=>{
 const a=g[c];
 const v=a?parseFloat(a[0]):NaN;
 return isFinite(v)?v:(d||0);
};
const STR=(g,c,d)=>{const a=g[c]; return a?a[0]:(d==null?"":d)};
const ARR=(g,c)=>(g[c]||[]).map(v=>parseFloat(v));
const INT=(g,c,d)=>{
 const a=g[c];
 const v=a?parseInt(a[0],10):NaN;
 return isFinite(v)?v:(d||0);
};
/* ═══ المصفوفة ═══ */
const mul=(m,n)=>({
 a:m.a*n.a+m.c*n.b, b:m.b*n.a+m.d*n.b,
 c:m.a*n.c+m.c*n.d, d:m.b*n.c+m.d*n.d,
 e:m.a*n.e+m.c*n.f+m.e, f:m.b*n.e+m.d*n.f+m.f});
const ap=(m,p)=>[m.a*p[0]+m.c*p[1]+m.e, m.b*p[0]+m.d*p[1]+m.f];
const sxOf=m=>Math.hypot(m.a,m.b);
const syOf=m=>Math.hypot(m.c,m.d);
const rotOf=m=>Math.atan2(m.b,m.a)*R2D;
const detOf=m=>m.a*m.d-m.b*m.c;
const uni=m=>{
 const x=sxOf(m), y=syOf(m);
 return detOf(m)>0 && Math.abs(x-y)<=1e-6*Math.max(x,1);
};
/* ═══ التجميع الخام ═══
   السقف على القيَم لا على الأسطر: نمضي في الملفّ ولا نجمع.
   ومضلّعٌ بعشرة ملايين رأسٍ كان يمرّ إلى الحالة والتاريخ والحفظ. */
const GCAP=MAXPTS*3;      /* لكل رمزٍ في كيانٍ واحد */
const GKEYS=4096;         /* أقصى عددِ رموزٍ مختلفة في كيان */
function grab(P,i){
 const t=String(P[i][1]).trim(), g={};
 let j=i+1, cut=0, keys=0;
 for(;j<P.length&&P[j][0]!==0;j++){
  const c=P[j][0];
  let a=g[c];
  if(a===undefined){
   /* كيانٌ فيه مليون رمزٍ مختلف يبني مليون مصفوفة ولو كانت
      كلٌّ بقيمةٍ واحدة — فالسقف على الرموز أيضاً */
   if(keys>=GKEYS){cut++; continue}
   a=g[c]=[]; keys++;
  }
  if(a.length>=GCAP){cut++; continue}
  a.push(P[j][1]);
 }
 return {type:t,g,j,cut};
}
function collect(P,from,to){
 const out=[];
 let i=from;
 while(i<to){
  if(P[i][0]!==0){i++; continue}
  const t=String(P[i][1]).trim();
  if(t==="ENDSEC"||t==="ENDBLK"||t==="BLOCK")break;
  const r=grab(P,i);
  i=r.j;
  if(t==="SEQEND")continue;
  if(t==="POLYLINE"){
   const vs=[];
   let vcut=0;
   while(i<to&&P[i][0]===0){
    const nt=String(P[i][1]).trim();
    if(nt==="VERTEX"){
     const v=grab(P,i); i=v.j;
     if(vs.length<MAXPTS)vs.push(v.g); else vcut++;
     continue;
    }
    if(nt==="SEQEND"){const s=grab(P,i); i=s.j}
    break;
   }
   out.push({type:t,g:r.g,vs,vcut});
   continue;
  }
  out.push({type:t,g:r.g});
 }
 return {list:out,end:i};
}
function sections(P){
 const S={};
 for(let i=0;i<P.length;i++){
  if(P[i][0]!==0||String(P[i][1]).trim()!=="SECTION")continue;
  let nm="";
  for(let j=i+1;j<P.length&&P[j][0]!==0;j++)
   if(P[j][0]===2)nm=String(P[j][1]).trim();
  let e=i+1;
  for(;e<P.length;e++)
   if(P[e][0]===0&&String(P[e][1]).trim()==="ENDSEC")break;
  S[nm]=[i+1,e];
  i=e;
 }
 return S;
}
/* ═══ الانتفاخ → قوس مقطَّع ═══
   قسمةٌ على جيبٍ صفريّ تعطي r=∞ ثم نقاط NaN ثم مفاتيح
   "NaN,NaN" في شبكة الالتقاط — فالحرس عند القسمة نفسها. */
function bulgePts(p1,p2,bg){
 const b=+bg||0;
 if(!isFinite(b)||Math.abs(b)<1e-9)return [];
 if(Math.abs(b)>1e6)return [];        /* انتفاخٌ لا معنى له */
 const th=4*Math.atan(b);
 const dx=p2[0]-p1[0], dy=p2[1]-p1[1], ch=Math.hypot(dx,dy);
 if(ch<1e-9)return [];
 const sn=Math.sin(Math.abs(th)/2);
 if(sn<1e-9)return [];                /* قسمةٌ على صفر ⇒ ∞ ⇒ NaN */
 const r=ch/(2*sn);
 if(!isFinite(r)||r>MAXCO)return [];
 const h=r*Math.cos(th/2);
 const mx=(p1[0]+p2[0])/2, my=(p1[1]+p2[1])/2;
 const sg=(th>0)?1:-1;
 const cx=mx-h*(dy/ch)*sg, cy=my+h*(dx/ch)*sg;
 if(!isFinite(cx)||!isFinite(cy))return [];
 const a0=Math.atan2(p1[1]-cy,p1[0]-cx);
 const n=clamp(Math.ceil(Math.abs(th)/(Math.PI/12)),2,48);
 const out=[];
 for(let i=1;i<n;i++)
  out.push([cx+r*Math.cos(a0+th*i/n), cy+r*Math.sin(a0+th*i/n)]);
 return out.filter(okPt);
}
/* ═══ السياق والحدّ العامّ ═══ */
function mkCtx(cap,ops,ms){
 return {out:[],cap:cap||MAXENT,trunc:0,skip:{},
  approx:{arc:0,spline:0,ellipse:0},src:{},
  ops:0, maxOps:ops||MAXOPS, maxMs:ms||MAXSEC,
  stop:"", clipped:0, t0:Date.now()};
}
const skip=(x,t)=>{x.skip[t]=(x.skip[t]||0)+1};
/* ═══ الحدّ العامّ ═══
   الحدود الموضعية (عمق التعشيق ٥ · حجم المصفوفة ٤٠٠) تحدّ كلَّ
   مستوىً ولا تحدّ حاصلها: بلوكٌ يُدرِج بلوكاً بمصفوفة ٢٠×٢٠ في
   كل مستوى يعطي 400⁵ ≈ 10¹³ نداءَ convert. وput تتوقّف عن
   الإضافة بعد ستّين ألفاً ولا توقف المسح — فالنتيجة تعليقٌ تامّ
   للخيط بلا إلغاءٍ ولا مؤقّت.
   والحدُّ هنا واحدٌ للتحليل كلّه، يُفحَص في كل مدخلٍ ومع كل
   تكرار، فيتوقّف التحليل ويُبلَّغ. */
function tick(x,n){
 if(x.stop)return false;
 x.ops+=(n||1);
 if(x.ops>x.maxOps){x.stop="ops"; return false}
 /* حدٌّ زمنيٌّ ثانٍ: ملفٌّ عريضٌ لا عميق قد يبلغ الوقت قبل العدد.
    والقناع يمنع نداء Date.now في كل عملية. */
 if((x.ops&1023)<8&&Date.now()-x.t0>x.maxMs){
  x.stop="time"; return false;
 }
 return true;
}
/* ═══ الحرس الأخير قبل الحالة ═══
   قيمةٌ شاذّة تُنبَذ وتُعَدّ باسمها، ولا تُصحَّح — الاختراع أسوأ
   من النبذ. وهو بعد تحويل الوحدة يقيناً. */
function okEnt(e){
 if(!e)return false;
 if(e.t==="l")return okPt(e.a)&&okPt(e.b);
 if(e.t==="x")return okPt(e.p);
 if(e.t==="p"){
  if(!Array.isArray(e.pts)||e.pts.length<2)return false;
  e.pts=e.pts.filter(okPt);
  return e.pts.length>1;
 }
 if(e.t==="a")return okPt(e.c)&&fin(e.r)&&e.r>0&&e.r<=MAXCO
  &&fin(e.a0)&&fin(e.a1);
 if(e.t==="t")return okPt(e.p)&&fin(e.h)&&e.h>0&&e.h<=MAXCO
  &&fin(e.rot);
 return false;
}
function put(x,e,sl){
 if(x.out.length>=x.cap){x.trunc++; return}
 if(!okEnt(e)){skip(x,"قيمة خارج المدى"); return}
 e.sl=sl||"0";
 x.src[e.sl]=(x.src[e.sl]||0)+1;
 x.out.push(e);
}
function tess(x,M,c,r,a0,a1,sl,dash){
 let sw=((a1-a0)%360+360)%360;
 if(sw<1e-6)sw=360;
 const n=clamp(Math.ceil(sw/7.5),6,96);
 const pts=[];
 for(let i=0;i<=n;i++){
  const a=(a0+sw*i/n)*D2R;
  pts.push(ap(M,[c[0]+r*Math.cos(a), c[1]+r*Math.sin(a)]).map(R));
 }
 const cl=(sw>=359.99)?1:0;
 if(cl)pts.pop();
 tick(x,n);
 put(x,{t:"p",pts,cl,dash:dash||null},sl);
}
function arcOut(x,M,c,r,a0,a1,sl){
 if(!fin(r)||r<=0)return;
 if(uni(M)){
  const k=sxOf(M), rr=rotOf(M);
  put(x,{t:"a",c:ap(M,c).map(R),r:R(r*k),
   a0:deg(a0+rr),a1:deg(a1+rr)},sl);
  return;
 }
 x.approx.arc++;                 /* مقياس غير متساوٍ أو معكوس */
 tess(x,M,c,r,a0,a1,sl);
}
const AL72=["l","c","r","l","c"];
const AL73=["b","b","m","t"];
const ATT=["","tl","tc","tr","ml","mc","mr","bl","bc","br"];
const mtClean=s=>String(s||"")
 .replace(/\\P/g," ").replace(/\\[A-Za-z][^;\\]*;/g,"")
 .replace(/[{}]/g,"").replace(/\\\\/g,"\\").trim();

function convert(list,M,B,depth,x){
 if(x.stop)return;
 for(const E of (list||[])){
  if(!tick(x))return;              /* الخروج من الحلقة لا تخطّيها */
  const g=E.g, sl=(STR(g,8,"0").trim()||"0");
  const T=E.type;
  if(T==="LINE"){
   put(x,{t:"l",a:ap(M,[NUM(g,10),NUM(g,20)]).map(R),
    b:ap(M,[NUM(g,11),NUM(g,21)]).map(R)},sl);
   continue;
  }
  if(T==="POINT"){
   put(x,{t:"x",p:ap(M,[NUM(g,10),NUM(g,20)]).map(R)},sl);
   continue;
  }
  if(T==="CIRCLE"){
   arcOut(x,M,[NUM(g,10),NUM(g,20)],NUM(g,40),0,360,sl);
   continue;
  }
  if(T==="ARC"){
   arcOut(x,M,[NUM(g,10),NUM(g,20)],NUM(g,40),
    NUM(g,50),NUM(g,51),sl);
   continue;
  }
  if(T==="LWPOLYLINE"||T==="POLYLINE"){
   const fl=INT(g,70,0);
   if(T==="POLYLINE"&&(fl&16||fl&64)){skip(x,"POLYMESH"); continue}
   const V=[];
   if(T==="LWPOLYLINE"){
    const xs=ARR(g,10), ys=ARR(g,20), bs=g[42]?ARR(g,42):[];
    const n0=Math.min(xs.length,ys.length);
    const n=Math.min(n0,MAXPTS);
    if(n<n0)x.clipped++;
    for(let i=0;i<n;i++){const pv={p:[xs[i],ys[i]],b:bs[i]||0}; V.push(pv)}
   }else{
    if(E.vcut)x.clipped++;
    (E.vs||[]).forEach(v=>V.push({
     p:[NUM(v,10),NUM(v,20)], b:NUM(v,42,0)}));
   }
   tick(x,V.length);          /* الرؤوس عملٌ يُحتسَب */
   if(V.length<2){
    if(V.length)put(x,{t:"x",p:ap(M,V[0].p).map(R)},sl);
    continue;
   }
   const cl=!!(fl&1);
   const pts=[];
   const n=cl?V.length:V.length-1;
   for(let i=0;i<n;i++){
    const A=V[i].p, Bp=V[(i+1)%V.length].p;
    pts.push(ap(M,A).map(R));
    bulgePts(A,Bp,V[i].b).forEach(q=>pts.push(ap(M,q).map(R)));
   }
   if(!cl)pts.push(ap(M,V[V.length-1].p).map(R));
   put(x,{t:"p",pts,cl:cl?1:0},sl);
   continue;
  }
  if(T==="SOLID"||T==="TRACE"||T==="3DFACE"){
   const Q=[[10,20],[11,21],[13,23],[12,22]]
    .map(([a,b])=>ap(M,[NUM(g,a),NUM(g,b)]).map(R));
   const U=Q.filter((p,i)=>i===0
    ||Math.hypot(p[0]-Q[i-1][0],p[1]-Q[i-1][1])>1);
   if(U.length>2)put(x,{t:"p",pts:U,cl:1},sl);
   continue;
  }
  if(T==="TEXT"){
   const j72=INT(g,72,0), j73=INT(g,73,0);
   const use2=(j72>0||j73>0)&&g[11];
   const p=use2?[NUM(g,11),NUM(g,21)]:[NUM(g,10),NUM(g,20)];
   const hz=AL72[clamp(j72,0,4)]||"l";
   const vt=AL73[clamp(j73,0,3)]||"b";
   put(x,{t:"t",p:ap(M,p).map(R),
    s:String(STR(g,1,"")).replace(/%%[dcpu]/gi,""),
    h:Math.max(1,R(NUM(g,40,2.5)*sxOf(M))),
    rot:deg(NUM(g,50,0)+rotOf(M)),
    al:((vt==="t"||vt==="m")?"m":"b")+hz},sl);
   continue;
  }
  if(T==="MTEXT"){
   const s=mtClean((g[3]||[]).join("")+STR(g,1,""));
   if(!s)continue;
   const at=ATT[clamp(INT(g,71,1),1,9)]||"bl";
   put(x,{t:"t",p:ap(M,[NUM(g,10),NUM(g,20)]).map(R),s,
    h:Math.max(1,R(NUM(g,40,2.5)*sxOf(M))),
    rot:deg(NUM(g,50,0)+rotOf(M)),
    al:(at[0]==="b"?"b":"m")+at[1]},sl);
   continue;
  }
  if(T==="ELLIPSE"){
   x.approx.ellipse++;
   const c=[NUM(g,10),NUM(g,20)];
   const mj=[NUM(g,11),NUM(g,21)];
   const rt=NUM(g,40,1);
   const t0=NUM(g,41,0), t1=NUM(g,42,Math.PI*2);
   const ra=Math.hypot(mj[0],mj[1]);
   const ph=Math.atan2(mj[1],mj[0]);
   const full=Math.abs((t1-t0)-Math.PI*2)<1e-6;
   const n=clamp(Math.ceil(Math.abs(t1-t0)/(Math.PI/24)),8,96);
   tick(x,n);
   const pts=[];
   for(let i=0;i<=n;i++){
    const t=t0+(t1-t0)*i/n;
    const u=ra*Math.cos(t), v=ra*rt*Math.sin(t);
    pts.push(ap(M,[c[0]+u*Math.cos(ph)-v*Math.sin(ph),
     c[1]+u*Math.sin(ph)+v*Math.cos(ph)]).map(R));
   }
   if(full)pts.pop();
   put(x,{t:"p",pts,cl:full?1:0},sl);
   continue;
  }
  if(T==="SPLINE"){
   /* تقريبٌ متقطّع: التقطيع علامةُ التقريب فلا يُلبَس يقيناً */
   x.approx.spline++;
   const fx=ARR(g,11), fy=ARR(g,21);
   const cx=ARR(g,10), cy=ARR(g,20);
   const X=(fx.length>1)?fx:cx, Y=(fx.length>1)?fy:cy;
   const n0=Math.min(X.length,Y.length);
   const n=Math.min(n0,MAXPTS);
   if(n<n0)x.clipped++;
   tick(x,n);
   const pts=[];
   for(let i=0;i<n;i++)pts.push(ap(M,[X[i],Y[i]]).map(R));
   if(pts.length>1)
    put(x,{t:"p",pts,cl:(INT(g,70,0)&1)?1:0,dash:[600,400]},sl);
   continue;
  }
  if(T==="INSERT"||T==="DIMENSION"){
   const nm=STR(g,2,"").trim();
   const blk=B[nm];
   if(!blk){skip(x,T+" بلا بلوك"); continue}
   if(depth>=5){skip(x,"تعشيق عميق"); continue}
   if(T==="DIMENSION"){
    convert(blk.list,M,B,depth+1,x);
    continue;
   }
   const ins=[NUM(g,10),NUM(g,20)];
   const sx=NUM(g,41,1)||1, sy=NUM(g,42,1)||1;
   const rot=NUM(g,50,0)*D2R;
   const ca=Math.cos(rot), sa=Math.sin(rot);
   const cols=clamp(INT(g,70,1)||1,1,200);
   const rows=clamp(INT(g,71,1)||1,1,200);
   const cs=NUM(g,44,0), rs=NUM(g,45,0);
   if(cols*rows>400){skip(x,"مصفوفة INSERT كبيرة"); continue}
   /* الحلقة تُفحَص في كل تكرار: الحدّ الموضعيّ (٤٠٠) يحدّ هذا
      المستوى، والحاصل هو ما يُعلِّق — فالفحص هنا لا هناك. */
   for(let r0=0;r0<rows&&!x.stop;r0++)
   for(let c0=0;c0<cols&&!x.stop;c0++){
    if(!tick(x))break;
    const off={a:1,b:0,c:0,d:1,
     e:ins[0]+c0*cs*ca-r0*rs*sa,
     f:ins[1]+c0*cs*sa+r0*rs*ca};
    const RS={a:ca*sx,b:sa*sx,c:-sa*sy,d:ca*sy,e:0,f:0};
    const BB={a:1,b:0,c:0,d:1,e:-blk.base[0],f:-blk.base[1]};
    convert(blk.list, mul(M,mul(off,mul(RS,BB))), B, depth+1, x);
   }
   continue;
  }
  if(T==="VIEWPORT"||T==="ATTDEF"||T==="ATTRIB")continue;
  skip(x,T);
 }
}
/* ═══ المدخل ═══ */
export function parseDXF(txt,opt){
 const O=Object.assign({cap:MAXENT,unit:null,maxOps:MAXOPS,
  maxMs:MAXSEC},opt||{});
 const P=pairs(txt);
 if(!P.length)throw new Error("الملفّ لا يحمل أزواج DXF");
 const SEC=sections(P);
 if(!SEC.ENTITIES)
  throw new Error("لا قسم ENTITIES — ليس ملفّ DXF صالحاً");
 /* الترويسة */
 let iu=0, cp="";
 if(SEC.HEADER){
  const [a,b]=SEC.HEADER;
  for(let i=a;i<b;i++){
   if(P[i][0]!==9)continue;
   const k=String(P[i][1]).trim();
   for(let j=i+1;j<b&&P[j][0]!==9;j++){
    if(k==="$INSUNITS"&&P[j][0]===70)iu=parseInt(P[j][1],10)||0;
    if(k==="$DWGCODEPAGE"&&P[j][0]===3)cp=String(P[j][1]).trim();
   }
  }
 }
 const U=(O.unit!=null)?[String(O.unit),+O.unit||1]
  :(UNITS[iu]||UNITS[0]);
 /* معامل الوحدة يضرب كل إحداثيّ، فقيمةٌ شاذّة فيه تُفسِد الملفّ
    كلَّه بلا كلمة. وO.unit يأتي من قائمةٍ مغلقة في الواجهة، لكن
    الحرس رخيص — والقسر إلى ١ أصدق من نبذ الملفّ. */
 const f=(isFinite(U[1])&&U[1]>0&&U[1]<1e6)?U[1]:1;
 /* البلوكات */
 const B={};
 if(SEC.BLOCKS){
  const [a,b]=SEC.BLOCKS;
  let i=a;
  while(i<b){
   if(P[i][0]!==0){i++; continue}
   if(String(P[i][1]).trim()!=="BLOCK"){i++; continue}
   const hd=grab(P,i);
   const nm=STR(hd.g,2,"").trim();
   const base=[NUM(hd.g,10),NUM(hd.g,20)];
   const c=collect(P,hd.j,b);
   if(nm)B[nm]={base,list:c.list};
   i=c.end;
   if(i<b&&P[i][0]===0&&String(P[i][1]).trim()==="ENDBLK")
    i=grab(P,i).j;
  }
 }
 /* الكيانات — معامل الوحدة مصفوفةُ الجذر، فالحرس في put يقع
    بعد التحويل يقيناً */
 const x=mkCtx(O.cap,O.maxOps,O.maxMs);
 const [ea,eb]=SEC.ENTITIES;
 const E=collect(P,ea,eb);
 convert(E.list,{a:f,b:0,c:0,d:f,e:0,f:0},B,0,x);
 return {ents:x.out, src:x.src, skip:x.skip, trunc:x.trunc,
  approx:x.approx, blocks:Object.keys(B).length,
  units:{code:iu,name:U[0],f}, codepage:cp,
  guessed:(O.unit==null&&!UNITS[iu]),
  stop:x.stop, ops:x.ops, clipped:x.clipped,
  ms:Date.now()-x.t0};
}
/* drop: مفاتيحُ تُذكَر برسالةٍ خاصّة فلا تُعاد في الملخّص العامّ */
export const skipSummary=(s,drop)=>Object.keys(s||{})
 .filter(k=>!(drop||[]).includes(k))
 .map(k=>`${k} ×${s[k]}`).join(" · ");
```

<a id="f-js-io-elev-js"></a>

---

## `js/io/elev.js`

```javascript
/* ═══ الواجهة صفحةً تُصدَّر ═══
   لا مسارَ ثانياً في io/*: المصدِّرات الثلاثة تقرأ الأوّليات العامّة،
   وelevPrims تُخرجها بنوعٍ تعرفه (poly مغلقة). فهذا الملفّ يبني
   الصندوق والاسم والملاحظات، ويسلّم — لا أكثر.

   والطبقة A-ELEV تُعلَن في DXF من نفسها: toDXF يجمع الطبقات
   المستعملة من أوّليات المشهد ثم يقرأ resolve، فلا جدولَ ثانٍ
   يُحدَّث. */
import {S} from "../core/state.js";
import {ELAY,elevPrims,lastElev} from "../core/elevation.js";
import {vis,plots,filterPrims} from "../core/layers.js";
import {toSVG} from "./svg.js";
import {toDXFBytes} from "./dxf.js";
import {toPDF,toPDFz} from "./pdf.js";

export const PAD=200;            /* هامشٌ نموذجيّ حول الفرد — مم */
const TAG={0:"E",90:"N",180:"W",270:"S"};
export const elevTag=e=>{
 const a=Math.round((e&&e.view)||0);
 return TAG[a]||`${a}deg`;
};
export const safeName=s=>String(s==null?"":s).trim()
 .replace(/[\\/:*?"<>|]+/g,"_").replace(/\s+/g,"_")
 .slice(0,40)||"PLAN";
export const elevName=(e,ext)=>
 `${safeName(S.meta.name)}-ELEV-${elevTag(e)}.${ext}`;

/* ═══ الصفحة ═══
   الهامش يُطبَّق على الصندوق لا بـopt.pad: toDXFBytes لا تأخذ
   خياراتٍ، فلو تُرك لكلٍّ هامشُه لاختلفت الثلاثة في المقاس. */
export function elevPage(e,opt){
 const o=opt||{};
 const t=e||lastElev();
 if(!t)throw new Error("لا واجهة — نفّذ ELEV أوّلاً");
 const pad=(o.pad==null)?PAD:Math.max(0,Math.round(+o.pad||0));
 const raw=elevPrims(t,0,0);
 const prims=filterPrims(raw);
 const box=raw.length
  ? {x0:-pad, y0:-pad, x1:t.w+pad, y1:t.h+pad}
  : {x0:0,y0:0,x1:1000,y1:1000};
 const notes=[];
 if(!prims.length)notes.push("الواجهة خالية — لا شكل يُصدَّر");
 if(!vis(ELAY))
  notes.push(`طبقة ${ELAY} مخفيّة — لن يخرج منها شيء`);
 else if(!plots(ELAY))
  notes.push(`طبقة ${ELAY} لا تُطبَع — الملفّ يخرج فارغاً`);
 return {e:t, prims, box, pad, notes};
}
const infoOf=e=>({
 title:`${S.meta.name||"PLAN"} — ${e.name}`,
 sheet:S.title.sheet, rev:S.title.rev,
 proj:S.title.proj, by:S.title.by});

export function elevSVG(e,opt){
 const P=elevPage(e,opt);
 const r=toSVG(P.prims,P.box,{dark:0,pad:0,showWarn:false,
  page:(opt&&opt.page)||null});
 return {name:elevName(P.e,"svg"), mime:"image/svg+xml;charset=utf-8",
  txt:r.txt, notes:P.notes.concat(r.notes||[]), page:P};
}
export function elevDXF(e,opt){
 const P=elevPage(e,opt);
 const r=toDXFBytes(P.prims,P.box);
 return {name:elevName(P.e,"dxf"), mime:"application/dxf",
  bytes:r.bytes, bad:r.bad,
  notes:P.notes.concat(r.notes||[]), page:P};
}
/* المتزامنة للأمر: الواجهة عشراتُ أشكالٍ لا مئاتُ آلاف، فالضغط
   لا يفيد ويُدخل الانتظار في مسارٍ لحظيّ. وelevPDFz لمن أراده. */
export function elevPDF(e,opt){
 const P=elevPage(e,opt);
 const r=toPDF(P.prims,P.box,{pad:0,showWarn:false,
  info:infoOf(P.e), page:(opt&&opt.page)||null});
 return {name:elevName(P.e,"pdf"), mime:"application/pdf",
  bytes:r.bytes, arabic:r.arabic,
  notes:P.notes.concat(r.notes||[]), page:P};
}
export async function elevPDFz(e,opt){
 const P=elevPage(e,opt);
 const r=await toPDFz(P.prims,P.box,{pad:0,showWarn:false,
  info:infoOf(P.e), page:(opt&&opt.page)||null});
 return {name:elevName(P.e,"pdf"), mime:"application/pdf",
  bytes:r.bytes, arabic:r.arabic, zip:r.zip,
  notes:P.notes.concat(r.notes||[]), page:P};
}
export const elevFile=(fmt,e,opt)=>{
 const f=String(fmt||"svg").toLowerCase();
 if(f==="dxf")return elevDXF(e,opt);
 if(f==="pdf")return elevPDF(e,opt);
 if(f==="svg")return elevSVG(e,opt);
 throw new Error(`صيغةٌ غير معروفة: «${fmt}» — svg أو dxf أو pdf`);
};
```

<a id="f-js-io-export-js"></a>

---

## `js/io/export.js`

```javascript
/* ═══ مسار التصدير الواحد ═══
   أربعةُ أزرارٍ كانت تُكرِّر خمسة أشياء: النطاق · التسمية · الحصيلة ·
   التحذيرات · التنزيل. والتكرارُ انجرف: warnClip في الأربعة
   وsayNotes في اثنين، وزرّان غير متزامنَين واثنان متزامنان، وواحدٌ
   يُعطِّل نفسه أثناء العمل وثلاثةٌ لا.

   وثلاثةُ قراراتٍ مُعلَنة:

   ١ · لا طبعَ من هنا. run تُعيد قائمةَ رسائل {lv,s} والواجهة تطبعها —
       فاستيرادُ ui/bus من io هو الاعتمادُ المعكوس نفسه الذي أُصلح في
       الدفعة ٧ب. والمكسب الثاني أن المسار يُختبَر في node بلا واجهة.

   ٢ · لا تنزيلَ من هنا. تُعيد الحِمل واسمَه ونوعَه، والواجهة تُنزِّله —
       فالتنزيل حدثُ متصفّحٍ لا صيغةَ ملفّ.

   ٣ · ترتيب الرسائل مُعلَن: الحصيلة أوّلاً (تُسمّي الملفّ فيُعرَف عمّا
       يُتكلَّم) · ثم ما فُقِد أو عُطِب · ثم ما يُشرَح. والفقدُ قبل
       الشرح لأن أخطر ما في التصدير أن تمضي وأنت تحسبه تمّ.

   والمشروع (‏xSave) ليس مخرَجاً فلا يمرّ بهنا: يحمل كل شيء بلا هيئةٍ
   ولا نطاقٍ ولا مقياس — الإخفاء عرضٌ لا حذف، والملفّ يحمل المخفيّ. */
import {S} from "../core/state.js";
import {clamp,dim2,m2} from "../core/units.js";
import {humanSize} from "./project.js";
import {scene,sceneBBox,sceneBBoxInk,
        sceneBBoxPlot} from "../core/render.js";
import {sheetRect} from "../core/sheet.js";
import {vis,plots,anyHidden,hiddenCount,
        hiddenLayers} from "../core/layers.js";
import {toDXFBytes,dxfStats} from "./dxf.js";
import {toSVG} from "./svg.js";
import {toPNGBlob} from "./png.js";
import {toPDFz} from "./pdf.js";

/* ═══ جدول الصيغ ═══
   قرارٌ معلَنٌ لا سلوكٌ مُستنتَج — كجدول CAPS في io/style.js.
   paper=0 لـDXF بقصد: هو فضاءُ نموذجٍ بالمليمتر لا صفحةً، فمقاس
   الورقة لا معنى له فيه. والورقة تُصدَّر إليه هندسةً (خطوطُ إطارها
   وبلوكها في المشهد) لا وسمَ صفحة. */
export const FMT={
 dxf:{ext:"dxf", mime:"application/dxf",  n:"DXF R2000",
      vector:1, paper:0, notes:1},
 svg:{ext:"svg", mime:"image/svg+xml",    n:"SVG متّجه",
      vector:1, paper:1, notes:1},
 png:{ext:"png", mime:"image/png",        n:"PNG",
      vector:0, paper:1, notes:1},
 pdf:{ext:"pdf", mime:"application/pdf",  n:"PDF متّجه",
      vector:1, paper:1, notes:1}
};

/* ═══ المقاسات الاسمية ═══
   تكرارٌ مُعلَنٌ لجدول core/sheet.js: sheetRect يعيد المليمتر
   النموذجيَّ مُدوَّراً (مقاسٌ × مقياسٌ ثم تدوير)، والصفحة تحتاج
   الاسميَّ نفسه — و٤١٩٫٩٨ مم ليست A3 عند الطابعة.
   وحالةٌ في run.js تفحص أن الجدولين يتّفقان لكل مقاسٍ واتجاه، فلا
   ينجرف أحدهما عن الآخر بلا أن يسقط الاختبار. */
const APS={A0:[841,1189],A1:[594,841],A2:[420,594],
 A3:[297,420],A4:[210,297]};
export function paperMM(){
 const a=APS[S.sheet.size]||APS.A3;
 return (S.sheet.orient==="p")
  ? {w:a[0],h:a[1],n:S.sheet.size,or:"عمودي"}
  : {w:a[1],h:a[0],n:S.sheet.size,or:"أفقي"};
}
export const onSheet=()=>!!(+S.sheet.on&&vis("A-SHET"));

/* ═══ النطاق ═══
   نُقل من ui/inspector: قرارُ نطاقٍ لا قرارُ زرّ — والدليل أنه كان
   يُقرَأ أربع مرّات ويُعدَّل مرّتين (الدفعتان ٧أ و٩).
   وصندوقُ ما يُطبَع لا ما يُرى: طبقةٌ أُوقِف طبعها كانت تُوسِّع
   الورقة فتخرج بهامشٍ خالٍ. والمخفيّ خارج الاثنين أصلاً — الصناديق
   تُحسَب بعد التصفية منذ الدفعة ٩. */
export function exportBox(){
 if(onSheet())
  return {box:sheetRect(sceneBBox()),pad:0,mode:"sheet"};
 const B=sceneBBoxPlot();
 const pad=Math.max(1,S.meta.scale)*8;
 return {box:B||{x0:0,y0:0,x1:1000,y1:1000},pad,mode:"fit"};
}
const padded=(B,pad)=>pad
 ? {x0:B.x0-pad,y0:B.y0-pad,x1:B.x1+pad,y1:B.y1+pad} : B;

/* ═══ الاحتواء ═══
   الورقة تقصّ ما خرج عنها، والقياسُ على صندوق الحبر بلا الورقة —
   وإلّا قِيسَت الورقة مقابل نفسها فلا تتجاوز أبداً.
   ويُعاد عددُ الورقات ومقياسٌ يكفي: «كبّر أو صغّر» نصيحةٌ بلا رقم،
   والرقم هو ما يُنفَّذ. */
const SCALES=[20,25,50,100,200,250,500,1000,1250,2500,5000];
export function fits(){
 if(!onSheet())return null;
 const p=paperMM(), k=Math.max(1,S.meta.scale);
 const r=sheetRect(sceneBBox()), ink=sceneBBoxInk();
 if(!r||!ink)return null;
 const over=Math.max(0, r.x0-ink.x0, ink.x1-r.x1,
                        r.y0-ink.y0, ink.y1-r.y1);
 const iw=(ink.x1-ink.x0)/1, ih=(ink.y1-ink.y0)/1;
 const nx=Math.max(1,Math.ceil(iw/(p.w*k)));
 const ny=Math.max(1,Math.ceil(ih/(p.h*k)));
 /* أصغرُ مقياسٍ قياسيّ يتّسع — بهامشٍ ٢٪ فلا يلامس الحدّ */
 const need=Math.max(iw/p.w, ih/p.h)*1.02;
 const fitK=SCALES.find(s=>s>=need)||Math.ceil(need/100)*100;
 return {over, nx, ny, n:nx*ny, fitK, paper:p};
}
/* ═══ الصفحة ═══
   اسميّةٌ حين تكون الورقة قائمةً والصيغة تحمل صفحة. والهندسة
   تُتَمركَز فيها: فرقُ التدوير دون المليمتر ويُقسَم على الجانبين،
   والنسبة تبقى 1:k بالضبط — لا 1:k±خطأً. */
export function pageOf(fmt){
 const F=FMT[fmt];
 if(!F||!F.paper||!onSheet())return null;
 const p=paperMM();
 return {w:p.w, h:p.h, name:p.n, or:p.or};
}
/* ═══ التسمية ═══
   في موضعٍ واحد للأربعة. وبلوكُ العنوان يدخلها: رقمُ اللوحة
   والمراجعة يُطبَعان على الورق، فثلاثُ لوحاتٍ من مشروعٍ واحد كانت
   تخرج بثلاثة أسماء متطابقة.
   والحرس على أسماء الملفّات لا على النصّ: ما يمنعه ويندوز
   (< > : " / \ | ? *) والمحارف الضابطة والنقطة الأخيرة والأسماء
   المحجوزة. والعربية تبقى كما هي — لا تحويلَ إلى لاتينية. */
const BADN=/^(con|prn|aux|nul|com[1-9]|lpt[1-9])$/i;
export function safeName(s,ext){
 let n=String(s==null?"":s)
  .replace(/[\u0000-\u001f\u007f]/g,"")
  .replace(/[<>:"/\\|?*]/g,"-")
  .replace(/[\u200e\u200f\u2066-\u2069]/g,"")
  .replace(/\s+/g,"-")
  .replace(/-{2,}/g,"-")
  .replace(/^[-.\s]+|[-.\s]+$/g,"")
  .slice(0,90);
 if(!n||BADN.test(n))n="لوحة";
 if(!ext)return n;
 const e=String(ext).replace(/^\./,"");
 return new RegExp(`\\.${e}$`,"i").test(n)?n:`${n}.${e}`;
}
export function fileName(fmt){
 const t=S.title||{};
 const P=[String(S.meta.name||"PLAN").trim()||"PLAN"];
 const sh=String(t.sheet||"").trim();
 const rv=String(t.rev||"").trim();
 if(sh)P.push(sh);
 if(rv&&rv!=="0")P.push("مر"+rv);
 return safeName(P.join("-"),FMT[fmt]?FMT[fmt].ext:"txt");
}
/* ═══ الخطّة ═══ ما سيُنتَج بلا إنتاج — تقرؤه الواجهة قبل النقر ═══ */
export function plan(fmt){
 const F=FMT[fmt]||FMT.pdf;
 const {box,pad,mode}=exportBox();
 return {fmt, name:fileName(fmt), mode, box, pad,
  page:pageOf(fmt), scale:Math.max(1,S.meta.scale),
  fit:fits(), vector:!!F.vector};
}
export function summary(){
 const k=Math.max(1,S.meta.scale);
 const f=fits();
 const L=[];
 if(onSheet()){
  const p=paperMM();
  L.push(`الورقة ${p.n} ${p.or} ${dim2(p.w,p.h,"مم")}`);
 }else L.push("النطاق: كل ما يُطبَع + هامش");
 L.push(`1:${k}`);
 L.push(`«${fileName("pdf").replace(/\.pdf$/,"")}»`);
 if(f&&f.over>0)
  L.push(`⚠ يتجاوز ${m2(f.over)} م — ${f.n} ورقة أو 1:${f.fitK}`);
 return L.join(" · ");
}
/* ═══ التحذيرات المشتركة ═══
   الفقدُ أوّلاً ثم العطب: الأوّل يُنقِص ما يخرج، والثاني يُخرِج
   ما ليس صحيحاً — وكلاهما يقع في مسار التسليم للعميل. */
function lossOf(fmt,pl){
 const out=[], c=scene();
 if(pl.mode==="sheet"&&pl.fit&&pl.fit.over>0)
  out.push({lv:"wr",s:`يتجاوز الورقة بـ ${m2(pl.fit.over)} م `
   +`فيُقصّ في المخرَج — بهذا المقياس يحتاج `
   +`${pl.fit.nx}×${pl.fit.ny} ورقة. كبّر الورقة أو انزل إلى `
   +`1:${pl.fit.fitK} أو أزِحها.`});
 if(anyHidden())
  out.push({lv:"wr",s:`${hiddenCount()} طبقةً مخفيّة ليست في `
   +`${FMT[fmt].n}: ${hiddenLayers().join(" · ")} — الإخفاء عرضٌ `
   +`لا حذف، وملفّ المشروع يحملها`});
 const np=[...new Set((c.P||[]).map(g=>g.L||"0"))]
  .filter(n=>!plots(n));
 if(np.length)
  out.push({lv:"in",s:`طبقاتٌ تُرى ولا تُطبَع فاستُثنيت: `
   +`${np.join(" · ")}`});
 /* عطبُ الرسم — يُقال قبل التنزيل لا بعده */
 if(c.bad)out.push({lv:"wr",s:`${c.bad} فتحةً معطوبة (خارج جدارها `
  +`أو متراكبة) صُدِّرت كما هي`});
 if(c.over)out.push({lv:"wr",s:`${c.over} بُعداً نصُّه مُستبدَل — `
  +`الرقم المطبوع لا يطابق الهندسة`});
 if(c.stale)out.push({lv:"wr",s:`${c.stale} منطقةً قديمة: بصمة `
  +`جوارها تبدّلت ومساحتُها لم تُحدَّث`});
 if(c.loose)out.push({lv:"in",s:`${c.loose} بُعداً معلَّقاً — طرفٌ `
  +`لا يصادف عقدةً ولا وجهاً`});
 if(c.open)out.push({lv:"wr",s:`${c.open} قطعةً لم تُخَط في اتحاد `
  +`الأجسام — جداران يتلامسان بمقدارٍ دون المليمتر، فحدٌّ ينفتح`});
 return out;
}
const pageStr=pl=>pl.page
 ? `${pl.page.name} ${pl.page.or} ${dim2(pl.page.w,pl.page.h,"مم")}`
 : null;
const sizeOf=v=>{
 if(!v)return 0;
 if(typeof v==="string")
  return (typeof TextEncoder!=="undefined")
   ? new TextEncoder().encode(v).length : v.length;
 if(v.length!=null)return v.length;
 if(v.size!=null)return v.size;
 return 0;
};
const blobOf=(raw,mime)=>(typeof Blob==="undefined")
 ? null : new Blob([raw],{type:mime});

/* ═══ التنفيذ ═══
   تعيد {ok · name · mime · blob · raw · size · report · plan}.
   وترمي فيما لا يُتوقَّع وحده؛ وما يُتوقَّع فشلُه (قماشٌ يتجاوز حدَّه)
   يعود ok:0 برسالةٍ تقول ما يُفعَل. */
export async function run(fmt,opt){
 const F=FMT[fmt];
 if(!F)throw new Error(`صيغةٌ مجهولة: ${fmt}`);
 const O=Object.assign({dark:0,showWarn:false,dpi:300},opt||{});
 const pl=plan(fmt);
 const P=scene().P;
 const bb=padded(pl.box,pl.pad);
 const name=pl.name;
 const notes=[], extra=[];
 let raw=null, blob=null, head="";

 if(fmt==="dxf"){
  const r=toDXFBytes(P,bb);
  raw=r.bytes; blob=blobOf(raw,F.mime);
  (r.notes||[]).forEach(m=>notes.push(m));
  if(r.hatchCut)extra.push({lv:"wr",s:`هاشورٌ في ${r.hatchCut} `
   +`موضعاً تجاوز حدّ الخطوط فلم يُصدَّر`});
  if(r.bad)extra.push({lv:"wr",s:`${r.bad} محرفاً لا وجود له في `
   +`CP1256 وكُتب «؟» — صفحةُ الرمز تحمل العربية والفرنسية ولا `
   +`تحمل ما عداهما. للنصّ الكامل استعمل SVG.`});
  const st=dxfStats(P);
  head=`${F.n} · ${name} · ${humanSize(sizeOf(raw))} · فضاء `
   +`النموذج بالمليمتر · `
   +Object.keys(st).map(k=>`${st[k]} ${k}`).join(" · ");
  notes.push("الشرطة مُقطَّعة قطعاً حقيقية · الهاشور خطوط مولَّدة "
   +"(HATCH غير مكتوبة) · الترميز CP1256 بايتاً بايتاً");
 }
 else if(fmt==="svg"){
  const r=toSVG(P,pl.box,{pad:pl.pad,dark:O.dark?1:0,
   showWarn:!!O.showWarn, page:pl.page});
  raw=r.txt; blob=blobOf(raw,F.mime);
  (r.notes||[]).forEach(m=>notes.push(m));
  if(r.hatchCut)extra.push({lv:"wr",s:`هاشورٌ في ${r.hatchCut} `
   +`موضعاً لم يُصدَّر`});
  /* الحجم بايتاتٌ لا محارف: العربية محرفان في UTF-8، وطولُ
     السلسلة كان يُنقِص الرقم إلى الثلث في لوحةٍ عربية */
  head=`${F.n} · ${name} · ${humanSize(sizeOf(raw))}`
   +(pageStr(pl)?` · ${pageStr(pl)}`:"")+` · 1:${pl.scale}`;
  notes.push("العربية نصٌّ متّجه بخطّ النظام — لا صورةَ ولا تنقيط");
 }
 else if(fmt==="png"){
  const dpi=clamp(parseInt(O.dpi,10)||300,72,1200);
  const nn=[];
  const {blob:b,info}=await toPNGBlob(P,pl.box,
   {pad:pl.pad,dpi,dark:O.dark?1:0,notes:nn,
    showWarn:!!O.showWarn, page:pl.page});
  nn.forEach(m=>notes.push(m));
  if(!b)return {ok:0,fmt,name,plan:pl,report:[{lv:"er",
   s:"تعذّر التنقيط — الأبعاد تتجاوز حدّ القماش في هذا "
    +"المتصفّح. قلّل الدقّة أو صغّر النطاق."}]};
  blob=b; raw=null;
  head=`${F.n} · ${name} · ${dim2(info.px,info.py,"بكسل")} · `
   +`${info.dpi} نقطة/بوصة · ${humanSize(sizeOf(b))}`
   +(pageStr(pl)?` · ${pageStr(pl)}`:"");
  if(info.scaled)extra.push({lv:"in",s:`خُفِّضت الدقّة من ${dpi} `
   +`لحدّ الأبعاد والمساحة`});
 }
 else{
  const r=await toPDFz(P,pl.box,{pad:pl.pad,
   showWarn:!!O.showWarn, page:pl.page,
   info:{title:S.meta.name, sheet:S.title.sheet,
    rev:S.title.rev, by:S.title.by, proj:S.title.proj}});
  raw=r.bytes; blob=blobOf(raw,F.mime);
  (r.notes||[]).forEach(m=>notes.push(m));
  if(r.hatchCut)extra.push({lv:"wr",s:`هاشورٌ في ${r.hatchCut} `
   +`موضعاً لم يُصدَّر`});
  head=`${F.n} · ${name} · ${humanSize(sizeOf(raw))} · `
   +(pageStr(pl)||dim2(r.pw.toFixed(0),r.ph.toFixed(0),"مم"))
   +` · 1:${pl.scale}`
   +(r.zip?` · ضُغِط المحتوى `
     +`${(r.zip.from/Math.max(1,r.zip.to)).toFixed(1)}×`:"");
  if(r.arabic)notes.push(`${r.arabic} نصّاً عربياً أُدرج قناعاً `
   +`بلون طبقته (${r.images} قناعاً فريداً) — الخطوط القياسية لا `
   +`تحمل العربية. للنصّ المتّجه استعمل SVG.`);
 }
 const report=[{lv:"ok",s:head}]
  .concat(extra, lossOf(fmt,pl),
   notes.map(s=>({lv:"in",s:`${FMT[fmt].n}: ${s}`})));
 return {ok:1, fmt, name, mime:F.mime, blob, raw,
  size:sizeOf(raw||blob), report, plan:pl};
}
```

<a id="f-js-io-pdf-js"></a>

---

## `js/io/pdf.js`

```javascript
/* ═══ كاتب PDF 1.4 يدوياً ═══
   الهندسة متّجهة كاملةً. النصّ: ما كان لاتينياً أو رقمياً يُكتَب
   بخطّ Helvetica المدمَج في القارئ، وما كان عربياً يُدرَج قناعاً
   أحاديّ البِت لكل نصّ فريد — لأن الخطوط الأربعة عشر القياسية لا
   تحمل العربية، وتضمين خطٍّ يحتاج ملفّ خطّ.

   والهيئة من io/style.js — مصدرٌ واحد يقرأه الأربعة. وكان هنا
   جدولُ ألوانٍ ثالث، ونسخةٌ ثانية من hatchLines بتبريرٍ غير صحيح،
   وتجاهلٌ لـplots() وللشرطة والشفافية، ولونٌ معتمٌ حرفيّ لصبغة
   المنطقة يخالف الشاشة اختلافاً لا يُخطَأ.

   لا اعتماديات: الضغط بـCompressionStream المدمج في المتصفّح. */
import {S} from "../core/state.js";
import {clamp} from "../core/units.js";
import {styleOf,fillOf,hatchOf,hatchLines,ctxOf,
        TINT_A} from "./style.js";

const MM2PT=72/25.4;
const ASCII=/^[\x20-\x7E]*$/;
/* عروض Helvetica لأشيع المحارف (لكل ١٠٠٠) */
const WID={32:278,37:889,40:333,41:333,43:584,44:278,45:333,46:278,
 47:278,48:556,49:556,50:556,51:556,52:556,53:556,54:556,55:556,
 56:556,57:556,58:278,88:667,120:500};
const strW=(s,size)=>{
 let w=0;
 for(let i=0;i<s.length;i++){
  const c=s.charCodeAt(i);
  w+=(WID[c]!=null?WID[c]:((c>=65&&c<=90)?722:556));
 }
 return w/1000*size;
};
const esc=s=>String(s).replace(/\\/g,"\\\\")
 .replace(/\(/g,"\\(").replace(/\)/g,"\\)");
const N=v=>String(Math.round((+v||0)*100)/100);
const enc=s=>{
 const a=new Uint8Array(s.length);
 for(let i=0;i<s.length;i++)a[i]=s.charCodeAt(i)&0xff;
 return a;
};
/* المُحلّ يعيد css نصّاً، وPDF يريد ٠–١ */
const hex2rgb=h=>{
 const s=String(h||"#000000").replace("#","");
 const n=(s.length===3)
  ? s.split("").map(c=>parseInt(c+c,16))
  : [parseInt(s.slice(0,2),16),parseInt(s.slice(2,4),16),
     parseInt(s.slice(4,6),16)];
 return n.map(v=>(isFinite(v)?v:0)/255);
};
/* ═══ قناع نصٍّ عربي ═══
   الخطوط الأربعة عشر القياسية لا تحمل العربية، وتضمين خطٍّ يحتاج
   ملفّ خطّ. فالنصّ صورة — لكن قناعاً لا صورةً ملوّنة:
   · بلا خلفية معتمة تحجب ما تحتها (كانت JPEG أبيض)
   · بلون طبقته لا أسودَ ثابتاً (كان الأسود، فأرقام الأبعاد
     اللاتينية تخرج بلون A-DIMS وأسماء الغرف سوداء)
   · بِتٌّ لكل بكسل بدل ثلاثة بايتات — نحو ٣٪ من الحجم */
function textMask(str,px){
 const h=clamp(Math.round(px),10,220);
 const m=document.createElement("canvas").getContext("2d");
 m.font=`${h}px Tahoma,Arial,sans-serif`;
 m.direction="rtl";
 const w=Math.max(4,
  Math.ceil(m.measureText(str).width)+Math.ceil(h*0.3));
 const H=Math.ceil(h*1.42);
 const c=document.createElement("canvas");
 c.width=w; c.height=H;
 const x=c.getContext("2d");
 x.fillStyle="#ffffff"; x.fillRect(0,0,w,H);
 x.font=`${h}px Tahoma,Arial,sans-serif`;
 x.direction="rtl"; x.textAlign="center";
 x.textBaseline="alphabetic";
 x.fillStyle="#000000";
 x.fillText(str,w/2,H-Math.round(h*0.30));
 /* أحاديّ البِت مصفوفاً بالصفوف: ١ يُطلى (Decode [1 0]).
    والصفّ الأول في بيانات الصورة هو أعلاها، كما في القماش. */
 const im=x.getImageData(0,0,w,H).data;
 const rowB=Math.ceil(w/8);
 const bits=new Uint8Array(rowB*H);
 for(let y=0;y<H;y++)for(let xx=0;xx<w;xx++){
  const a=im[(y*w+xx)*4];                /* الرمادي = R */
  if(a<128)bits[y*rowB+(xx>>3)]|=(0x80>>(xx&7));
 }
 return {data:bits, w, h:H, base:Math.round(h*0.30)/H,
  mask:1, rowB};
}
/* نصُّ PDF بالعربية يحتاج UTF-16BE ببادئة BOM: البايتات الخام
   تُقرأ خربشةً في القارئ — العلّة نفسها التي كانت في CP1256
   (الدفعة ٦)، وهنا الحلّ سلسلةٌ سِتّ عشريّة. */
const hex16=s=>{
 let h="FEFF";
 for(const ch of String(s==null?"":s).slice(0,120)){
  let c=ch.codePointAt(0);
  if(c>0xFFFF){
   c-=0x10000;
   h+=(0xD800+(c>>10)).toString(16).toUpperCase().padStart(4,"0");
   h+=(0xDC00+(c&0x3FF)).toString(16).toUpperCase().padStart(4,"0");
   continue;
  }
  h+=c.toString(16).toUpperCase().padStart(4,"0");
 }
 return "<"+h+">";
};
/* ═══ البناء ═══ */
export function toPDF(prims,box,opt){
 const O=Object.assign({pad:0,showWarn:false},opt||{});
 const B=box||{x0:0,y0:0,x1:1000,y1:1000};
 const pad=O.pad||0;
 const x0=B.x0-pad, y0=B.y0-pad;
 const W=(B.x1-B.x0)+pad*2, H=(B.y1-B.y0)+pad*2;
 const k=Math.max(1,S.meta.scale);
 /* ═══ الصفحة الاسمية ═══
    sheetRect يعيد المليمتر النموذجيَّ مُدوَّراً، فالصفحة المشتقّة
    منه ٤١٩٫٩٨ مم لا ٤٢٠ — والطابعة تُقيس على الاسميّ فتُصغِّر إلى
    «احتواء»، فالمقياس المطبوع ليس ١:١٠٠.
    فالمقاس اسميٌّ والهندسة تُتَمركَز فيه: الفرق دون المليمتر
    ويُقسَم على الجانبين، والنسبة تبقى 1:k بالضبط. */
 const cW=W/k*MM2PT, cH=H/k*MM2PT;
 const pg=(O.page&&+O.page.w>0&&+O.page.h>0)
  ? {pw:+O.page.w*MM2PT, ph:+O.page.h*MM2PT} : null;
 const pw=pg?pg.pw:cW, ph=pg?pg.ph:cH;
 const ox=pg?(pw-cW)/2:0, oy=pg?(ph-cH)/2:0;
 const S2=MM2PT/k;                       /* نموذج مم → نقطة */
 const X=v=>N((v-x0)*S2+ox), Y=v=>N((v-y0)*S2+oy);
 const notes=O.notes||[];
 /* px=S2: الوزن والشرطة بالنقاط مباشرةً */
 const SO=ctxOf("pdf",{dark:0,k,px:S2,minLw:0.15,notes,
  showWarn:O.showWarn!==false&&!!O.showWarn,
  hs:Math.max(8,(S.meta.txtMM*k)*2.5), hatchCut:0});
 const sty=g=>{
  const s=styleOf(g,"pdf",SO);
  if(!s.skip)s.rgb=hex2rgb(s.css);
  return s;
 };
 let c="";
 let cc=null, cw=null, cd=null;
 const setCol=v=>{
  const s=`${N(v[0])} ${N(v[1])} ${N(v[2])}`;
  if(s===cc)return;
  cc=s; c+=`${s} RG\n${s} rg\n`;
 };
 const setW=v=>{
  const s=N(v);
  if(s===cw)return;
  cw=s; c+=`${s} w\n`;
 };
 const setDash=d=>{
  /* الشرطة بالنقاط سلفاً (px=S2 في المُحلّ) */
  const s=(d&&d.length)?`[${d.map(N).join(" ")}] 0`:"[] 0";
  if(s===cd)return;
  cd=s; c+=`${s} d\n`;
 };
 /* ═══ حالات الشفافية ═══
    واحدةٌ لكل قيمةٍ مستعملة: CAPS تُعلن أن PDF يحمل الشفافية،
    فلا يجوز أن تُطبَّق على الصبغة وحدها. */
 const GS=new Map();
 const gsOf=a=>{
  const key=N(a);
  let g=GS.get(key);
  if(!g){g={name:"GA"+GS.size, a:+a}; GS.set(key,g)}
  return g.name;
 };
 const poly=(pts,cl)=>{
  pts.forEach((p,i)=>{
   c+=`${X(p[0])} ${Y(p[1])} ${i?"l":"m"}\n`;
  });
  if(cl!==0)c+="h\n";
 };
 const arc=(cx,cy,r,a0,a1)=>{
  let sw=a1-a0;
  while(sw<0)sw+=360;
  if(sw<0.05)sw=360;
  const n=Math.max(1,Math.ceil(sw/90));
  const st=sw/n, rd=a=>a*Math.PI/180;
  const P=a=>[cx+r*Math.cos(rd(a)), cy+r*Math.sin(rd(a))];
  const p=P(a0);
  c+=`${X(p[0])} ${Y(p[1])} m\n`;
  const t=(4/3)*Math.tan(rd(st)/4);
  for(let i=0;i<n;i++){
   const A=a0+st*i, Bg=A+st;
   const pa=P(A), pb=P(Bg);
   const c1=[pa[0]-t*r*Math.sin(rd(A)), pa[1]+t*r*Math.cos(rd(A))];
   const c2=[pb[0]+t*r*Math.sin(rd(Bg)),pb[1]-t*r*Math.cos(rd(Bg))];
   c+=`${X(c1[0])} ${Y(c1[1])} ${X(c2[0])} ${Y(c2[1])} `
    +`${X(pb[0])} ${Y(pb[1])} c\n`;
  }
 };
 const imgs=new Map();
 let arabic=0;

 c+=`q\n1 1 1 rg\n0 0 ${N(pw)} ${N(ph)} re f\n`;
 setW(0.5); setDash(null);

 /* ═══ التعبئات ═══ نقشُ خطوطٍ مولَّد صريحاً — لا أنماط PDF ═══ */
 (prims||[]).forEach(g=>{
  if(g.t==="fill"){
   const f=fillOf(g,"pdf",SO);
   if(f.skip)return;
   const r=g.ring||[];
   if(r.length<3)return;
   if(f.hatch){
    /* المنطقة المهشَّرة: خطوطٌ بلون طبقتها — وكانت تُصدَّر
       لوناً معتماً واحداً يخالف الشاشة */
    setCol(hex2rgb(f.css)); setW(f.lw);
    const H2=hatchLines([r],45,f.sp);
    SO.hatchCut+=H2.cut;
    H2.lines.forEach(s=>{
     c+=`${X(s[0][0])} ${Y(s[0][1])} m `
      +`${X(s[1][0])} ${Y(s[1][1])} l S\n`;
    });
    return;
   }
   /* لون الطبقة بشفافيةٍ حقيقية */
   c+=`q /${gsOf(f.a)} gs\n`;
   setCol(hex2rgb(f.css));
   poly(r,1); c+="f\nQ\n";
   cc=null;
   return;
  }
  if(g.t!=="hatch")return;
  const hs=hatchOf(g,"pdf",SO);        /* g.L لا اسمٌ حرفيّ */
  if(hs.skip)return;
  setCol(hex2rgb(hs.css)); setW(hs.lw);
  hs.sets.forEach(([ang,d])=>{
   const H2=hatchLines(g.loops,ang,d);
   SO.hatchCut+=H2.cut;
   H2.lines.forEach(s=>{
    c+=`${X(s[0][0])} ${Y(s[0][1])} m `
     +`${X(s[1][0])} ${Y(s[1][1])} l S\n`;
   });
  });
 });
 /* ═══ خطوط ونصوص ═══ */
 (prims||[]).forEach(g=>{
  if(g.t==="hatch"||g.t==="fill")return;
  const st=sty(g);
  if(st.skip)return;                   /* المخفيّ وما لا يُطبَع */
  const tr=(st.alpha<1);
  if(tr)c+=`q /${gsOf(st.alpha)} gs\n`;
  setCol(st.rgb);
  setW(st.lw);
  setDash(st.dash);            /* دَشّ الطبقة صار مُطبَّقاً */
  if(g.t==="line"){
   c+=`${X(g.a[0])} ${Y(g.a[1])} m ${X(g.b[0])} ${Y(g.b[1])} l S\n`;
  }else if(g.t==="poly"){
   if(g.pts&&g.pts.length>1){poly(g.pts,g.cl); c+="S\n"}
  }else if(g.t==="arc"){
   arc(g.cx,g.cy,Math.max(0.1,g.r),g.a0,g.a1); c+="S\n";
  }else if(g.t==="text"){
   const size=g.h*S2;
   if(size>=1.2){
    const s=String(g.s);
    const a=(g.rot||0)*Math.PI/180;
    const ca=Math.cos(a), sa=Math.sin(a);
    if(ASCII.test(s)){
     const w=strW(s,size);
     const ax=/l$/.test(g.al||"")?0:(/r$/.test(g.al||"")?-w:-w/2);
     const ay=/^m/.test(g.al||"")?-size*0.36:0;
     const px=(g.x-x0)*S2+ax*ca-ay*sa+ox;
     const py=(g.y-y0)*S2+ax*sa+ay*ca+oy;
     c+=`BT /F1 ${N(size)} Tf ${N(ca)} ${N(sa)} ${N(-sa)} ${N(ca)} `
      +`${N(px)} ${N(py)} Tm (${esc(s)}) Tj ET\n`;
    }else{
     /* عربي → قناع بلون الطبقة */
     arabic++;
     const key=s+"|"+Math.round(size*4);
     if(!imgs.has(key))imgs.set(key,
      Object.assign(textMask(s,Math.max(14,size*4)),
       {name:"Im"+(imgs.size+1)}));
     const im=imgs.get(key);
     const hpt=size*1.42;
     const wpt=hpt*(im.w/im.h);
     const ax=/l$/.test(g.al||"")?0
      :(/r$/.test(g.al||"")?-wpt:-wpt/2);
     const ay=/^m/.test(g.al||"")?(-hpt*0.5):(-im.base*hpt);
     const px=(g.x-x0)*S2+ax*ca-ay*sa+ox;
     const py=(g.y-y0)*S2+ax*sa+ay*ca+oy;
     /* القناع يُطلى بلون التعبئة الجاري — فيُضبَط قبله */
     const cs=`${N(st.rgb[0])} ${N(st.rgb[1])} ${N(st.rgb[2])}`;
     c+=`q ${cs} rg `
      +`${N(wpt*ca)} ${N(wpt*sa)} ${N(-hpt*sa)} ${N(hpt*ca)} `
      +`${N(px)} ${N(py)} cm /${im.name} Do Q\n`;
     cc=null; cw=null; cd=null;
    }
   }
  }
  if(tr){c+="Q\n"; cc=null; cw=null; cd=null}
 });
 c+="Q\n";

 /* ═══ تجميع الملفّ ═══ */
 const chunks=[], offs=[];
 let len=0;
 const push=u=>{chunks.push(u); len+=u.length};
 const obj=(n,body,bin)=>{
  offs[n]=len;
  push(enc(`${n} 0 obj\n`));
  push(enc(body));
  if(bin){push(bin); push(enc("\nendstream\n"))}
  push(enc("endobj\n"));
 };
 const IM=[...imgs.values()];
 const GL=[...GS.values()];
 const nImg0=6;
 const nGs0=6+IM.length;
 const nInfo=6+IM.length+GL.length;
 const total=nInfo;
 push(enc("%PDF-1.4\n%\xE2\xE3\xCF\xD3\n"));
 obj(1,`<< /Type /Catalog /Pages 2 0 R >>\n`);
 obj(2,`<< /Type /Pages /Kids [3 0 R] /Count 1 >>\n`);
 const xo=IM.length
  ? ` /XObject << ${IM.map((im,i)=>
     `/${im.name} ${nImg0+i} 0 R`).join(" ")} >>` : "";
 const xg=GL.length
  ? ` /ExtGState << ${GL.map((g,i)=>
     `/${g.name} ${nGs0+i} 0 R`).join(" ")} >>` : "";
 obj(3,`<< /Type /Page /Parent 2 0 R /MediaBox `
  +`[0 0 ${N(pw)} ${N(ph)}] /Resources << /Font << /F1 5 0 R >>`
  +`${xo}${xg} >> /Contents 4 0 R >>\n`);
 const cs=enc(c);
 let body=cs, filt="";
 if(O.deflated&&O.deflated.length){
  body=O.deflated; filt=" /Filter /FlateDecode";
 }
 obj(4,`<< /Length ${body.length}${filt} >>\nstream\n`,body);
 obj(5,`<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica `
  +`/Encoding /WinAnsiEncoding >>\n`);
 IM.forEach((im,i)=>{
  obj(nImg0+i,`<< /Type /XObject /Subtype /Image /Width ${im.w} `
   +`/Height ${im.h} /ImageMask true /Decode [1 0] `
   +`/BitsPerComponent 1 /Length ${im.data.length} >>\nstream\n`,
   im.data);
 });
 GL.forEach((g,i)=>{
  obj(nGs0+i,`<< /Type /ExtGState /ca ${N(g.a)} /CA ${N(g.a)} >>\n`);
 });
 /* بلوك العنوان يبلغ القارئ: اسمُ اللوحة يُرى في شريطه، ورقمُها
    ومراجعتُها في خصائص الملفّ — وكانت تُطبَع على الورق ولا تصل
    البيانات الوصفية. */
 const IF=O.info||{};
 const ttl=[IF.title,IF.sheet,IF.rev&&IF.rev!=="0"?"مر"+IF.rev:""]
  .filter(Boolean).join(" — ");
 obj(nInfo,`<< /Title ${hex16(ttl||"لوحة")} `
  +`/Subject ${hex16(IF.proj||"")} `
  +`/Author ${hex16(IF.by||"")} `
  +`/Creator ${hex16("CivilDraft")} /Producer ${hex16("CivilDraft")} >>\n`);
 const xref=len;
 let x=`xref\n0 ${total+1}\n0000000000 65535 f \n`;
 for(let i=1;i<=total;i++)
  x+=String(offs[i]||0).padStart(10,"0")+" 00000 n \n";
 x+=`trailer\n<< /Size ${total+1} /Root 1 0 R `
  +`/Info ${nInfo} 0 R >>\n`
  +`startxref\n${xref}\n%%EOF\n`;
 push(enc(x));

 const out=new Uint8Array(len);
 let p=0;
 chunks.forEach(u=>{out.set(u,p); p+=u.length});
 return {bytes:out, arabic, images:IM.length, gs:GL.length,
  pw:pw/MM2PT, ph:ph/MM2PT, notes, hatchCut:SO.hatchCut,
  stream:cs};
}
/* ═══ الضغط ═══
   CompressionStream في المتصفّح بلا مكتبة، فالوعد المُعلَن
   («لا اعتماديات») قائم. ومجرى المحتوى نصٌّ فيضغط خمسةً إلى
   عشرة أضعاف — والهندسة الكثيفة تُخرِج ملفّاتٍ بالميغابايتات.

   بناءٌ أوّل لاستخراج المجرى، ثم ضغطُه، ثم بناءٌ ثانٍ به. البناء
   رخيصٌ مقابل الضغط، والبديل تفكيكُ toPDF إلى مرحلتين. */
export async function toPDFz(prims,box,opt){
 const O=Object.assign({},opt||{});
 const r1=toPDF(prims,box,O);
 if(typeof CompressionStream==="undefined"
  ||typeof Blob==="undefined"
  ||typeof Response==="undefined"
  ||!r1.stream)return r1;
 let z=null;
 try{
  const cz=new Blob([r1.stream]).stream()
   .pipeThrough(new CompressionStream("deflate"));
  z=new Uint8Array(await new Response(cz).arrayBuffer());
 }catch(e){return r1}
 if(!z||z.length>=r1.stream.length)return r1;   /* لم يفد */
 const r2=toPDF(prims,box,Object.assign({},O,{deflated:z}));
 r2.zip={from:r1.stream.length, to:z.length};
 return r2;
}
```

<a id="f-js-io-png-js"></a>

---

## `js/io/png.js`

```javascript
/* ═══ تصدير PNG ═══
   رسّامٌ مستقلّ يقرأ الأوّليات نفسها، بلا اعتماد على قماش الشاشة —
   فيمكن التنقيط بأي دقّة دون تغيير العرض.

   والهيئة من io/style.js — مصدرٌ واحد يقرأه الأربعة. وكانت هنا
   نسخةٌ رابعة تقرأ theme.PRINT (اعتمادٌ من io على ui، معكوسٌ)،
   وتُهمِل plots() فتُصدِّر طبقةً أُوقِف طبعها، وتُهمِل شرطة الطبقة
   وشفافيتها، وتكتب لون صبغة المنطقة حرفياً، وتقرأ طبقة الهاشور
   باسمٍ ثابت فيخرج هاشور العمود بلونٍ خاطئ. */
import {S} from "../core/state.js";
import {clamp} from "../core/units.js";
import {styleOf,fillOf,hatchOf,ctxOf} from "./style.js";
import {paperMM} from "../core/sheet.js";

/* ═══ كاشُ النقوش ═══
   بمفتاح (مقاس · لون · تشابك): كان يُبنى قماشٌ جديد لكل تعبئةٍ
   ولكل هاشور — مئةُ منطقةٍ مهشَّرة تعني مئة قماشٍ في إطارٍ واحد. */
const PC=new Map();
function patFor(ctx,px,color,cross){
 const s=clamp(Math.round(px),4,120);
 const key=`${s}|${color}|${cross?1:0}`;
 const hit=PC.get(key);
 if(hit)return hit;
 const c=document.createElement("canvas");
 c.width=c.height=s;
 const x=c.getContext("2d");
 x.strokeStyle=color; x.lineWidth=Math.max(1,s*0.06);
 x.beginPath(); x.moveTo(0,s); x.lineTo(s,0);
 /* solid تشابكٌ في اتجاهين — كان خطّاً واحداً هنا واتجاهين في
    DXF وPDF، فالمخرَجات تختلف في النقش نفسه */
 if(cross){x.moveTo(0,0); x.lineTo(s,s)}
 x.stroke();
 const p=ctx.createPattern(c,"repeat");
 if(PC.size>60)PC.clear();
 PC.set(key,p);
 return p;
}
export const clearPatCache=()=>PC.clear();

/* ═══ الرسم على أي سياق ═══ tr: model → pixel ═══ */
export function paintTo(ctx,prims,tr,opt){
 const O=Object.assign({dark:0,minLW:0.6,showWarn:false,
  notes:null},opt||{});
 const k=Math.max(1,S.meta.scale);
 /* px=tr.k: الهيئة تُحسَب بالبكسل مباشرةً — الوزن والشرطة معاً،
    فلا يُضرَب أيٌّ منهما مرّةً ثانية هنا */
 const SO=ctxOf("png",{dark:!!O.dark,k,px:tr.k,minLw:O.minLW,
  notes:O.notes,showWarn:O.showWarn,
  hs:Math.max(8,(S.meta.txtMM*k)*2.5)});
 const M=p=>[(p[0]-tr.x0)*tr.k, (tr.y1-p[1])*tr.k];
 const path=(pts,cl)=>{
  ctx.beginPath();
  pts.forEach((q,i)=>{
   const p=M(q);
   if(i)ctx.lineTo(p[0],p[1]); else ctx.moveTo(p[0],p[1]);
  });
  if(cl!==0)ctx.closePath();
 };
 ctx.lineCap="round"; ctx.lineJoin="round";

 /* ═══ التعبئات أوّلاً ═══ */
 (prims||[]).forEach(g=>{
  if(g.t==="fill"){
   const f=fillOf(g,"png",SO);
   if(f.skip)return;
   const r=g.ring||[];
   if(r.length<3)return;
   ctx.save();
   try{
    path(r,1);
    if(f.hatch){
     ctx.fillStyle=patFor(ctx,Math.max(4,f.sp*tr.k),f.css,0);
    }else{
     /* لون طبقتها بشفافية TINT_A — كان rgba حرفياً يخالف
        الشاشة والمخرَجات الأخرى */
     ctx.globalAlpha=f.a;
     ctx.fillStyle=f.css;
    }
    ctx.fill();
   }finally{ctx.restore()}
   return;
  }
  if(g.t!=="hatch")return;
  const hs=hatchOf(g,"png",SO);       /* g.L لا اسمٌ حرفيّ */
  if(hs.skip)return;
  const loops=(g.loops||[]).filter(l=>l&&l.length>2);
  if(!loops.length)return;
  ctx.save();
  try{
   ctx.beginPath();
   loops.forEach(lp=>{
    lp.forEach((q,i)=>{
     const p=M(q);
     if(i)ctx.lineTo(p[0],p[1]); else ctx.moveTo(p[0],p[1]);
    });
    ctx.closePath();
   });
   ctx.fillStyle=patFor(ctx,
    Math.max(4,(hs.solid?hs.sp*0.22:hs.sp)*tr.k), hs.css, hs.solid);
   ctx.fill("evenodd");
  }finally{ctx.restore()}
 });
 /* ═══ ثم الخطوط والنصوص ═══ */
 (prims||[]).forEach(g=>{
  if(g.t==="hatch"||g.t==="fill")return;
  const st=styleOf(g,"png",SO);
  if(st.skip)return;                 /* المخفيّ وما لا يُطبَع */
  ctx.save();
  try{
   /* try/finally لا if/else: الشفافية المضبوطة لا تُصفَّر إن
      خرج فرعٌ بـreturn، فتُبهِت كل ما يُرسَم بعدها. والبنية
      تحرس من إضافةٍ لاحقة لا من علّةٍ قائمة. */
   ctx.strokeStyle=st.css; ctx.fillStyle=st.css;
   ctx.globalAlpha=st.alpha;      /* بلا شرط: العودة إلى ١ لازمة */
   ctx.lineWidth=st.lw;
   /* الشرطة بالبكسل سلفاً (px=tr.k في المُحلّ) */
   ctx.setLineDash((st.dash||[]).map(v=>Math.max(1,v)));
   if(g.t==="line"){
    const a=M(g.a), b=M(g.b);
    ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
    ctx.stroke();
   }else if(g.t==="poly"){
    if(g.pts&&g.pts.length>1){
     path(g.pts,g.cl);
     ctx.stroke();
    }
   }else if(g.t==="arc"){
    const c2=M([g.cx,g.cy]), r=Math.max(0.4,g.r*tr.k);
    ctx.beginPath();
    ctx.arc(c2[0],c2[1],r,-g.a1*Math.PI/180,-g.a0*Math.PI/180);
    ctx.stroke();
   }else if(g.t==="text"){
    const px=g.h*tr.k;
    if(px>=3){
     const p=M([g.x,g.y]);
     ctx.translate(p[0],p[1]);
     if(g.rot)ctx.rotate(-g.rot*Math.PI/180);
     ctx.font=`${px.toFixed(1)}px Tahoma,Arial,sans-serif`;
     ctx.direction="rtl";
     ctx.textAlign=/l$/.test(g.al||"")?"left"
      :(/r$/.test(g.al||"")?"right":"center");
     ctx.textBaseline=/^m/.test(g.al||"")?"middle":"alphabetic";
     ctx.setLineDash([]);
     ctx.fillText(String(g.s),0,0);
    }
   }
  }finally{ctx.restore()}
 });
 ctx.setLineDash([]);
 ctx.globalAlpha=1;
}
/* ═══ التنقيط ═══
   حدّ الضلع وحدّ المساحة معاً: 12000×12000 = ١٤٤ مليون بكسل ×
   أربعة بايتات = ٥٧٦ م.ب، وحدُّ القماش في سفاري المحمول ١٦٫٧
   مليون بكسل — فيعيد toBlob قيمةً فارغة أو قماشاً أبيض بلا خطأ. */
export const MAXSIDE=12000, MAXAREA=64e6;
export function renderCanvas(prims,box,opt){
 const O=Object.assign({dpi:300,dark:0,pad:0,
  max:MAXSIDE,maxArea:MAXAREA,notes:null,showWarn:false},opt||{});
 const B=box||{x0:0,y0:0,x1:1000,y1:1000};
 const pad=O.pad||0;
 const x0=B.x0-pad, y1=B.y1+pad;
 const W=(B.x1-B.x0)+pad*2, H=(B.y1-B.y0)+pad*2;
 const k=Math.max(1,S.meta.scale);
 /* الورقة الاسمية إن أُعلنت: البكسل يُشتَقّ منها فينطبق المطبوع
    على المتّجه — وكانت تُشتَقّ من الصندوق المُدوَّر، وبلا صفحةٍ
    معلَنة كانت تُشتَقّ من صندوق الهندسة مقسوماً على المقياس، فرسمٌ
    صغيرٌ يُخرِج ورقةً مجهريّة لا تبلغ حدَّي الضلع والمساحة مهما
    عَلَت الدقّة. والافتراضُ الآن مقاسُ الورقة القياسيّ نفسه الذي
    يُصدَّر عليه فعلاً حين لا صفحةَ صريحة — لا الصندوق. */
 const dflt=paperMM();
 const nom=(O.page&&+O.page.w>0&&+O.page.h>0)?O.page:null;
 const pw=nom?+nom.w:dflt[0], ph=nom?+nom.h:dflt[1];  /* مليمتر ورقي */
 const MW=pw*k, MH=ph*k;                    /* مليمتر نموذجي */
 const X0=x0-(MW-W)/2, Y1=y1+(MH-H)/2;
 let px=Math.round(pw/25.4*O.dpi);
 let py=Math.round(ph/25.4*O.dpi);
 let f=Math.min(1, O.max/Math.max(px,py,1));
 const area=(px*f)*(py*f);
 if(area>O.maxArea)f*=Math.sqrt(O.maxArea/area);
 /* الطرحُ لا التقريب في الخطوة الأخيرة: التقريب لأعلى قد يُعيد
    البكسلَ فوق الحدّين بعد ضربٍ في f — كسرٌ واحد يكفي لتجاوز
    حدّ المساحة بضبطه بالضبط. */
 px=Math.max(1,Math.floor(px*f));
 py=Math.max(1,Math.floor(py*f));
 clearPatCache();          /* النقوش بمقاسٍ يتبع tr.k */
 const cv=document.createElement("canvas");
 cv.width=px; cv.height=py;
 const ctx=cv.getContext("2d");
 ctx.fillStyle=O.dark?"#0e1216":"#ffffff";
 ctx.fillRect(0,0,px,py);
 paintTo(ctx,prims,{x0:X0,y1:Y1,k:px/MW},
  {dark:O.dark,minLW:Math.max(0.6,px/2400),
   notes:O.notes,showWarn:O.showWarn});
 return {canvas:cv,px,py,pw,ph,dpi:Math.round(O.dpi*f),
  scaled:f<1};
}
export const toPNGBlob=(prims,box,opt)=>new Promise(res=>{
 const r=renderCanvas(prims,box,opt);
 r.canvas.toBlob(b=>res({blob:b,info:r}),"image/png");
});
```

<a id="f-js-io-project-js"></a>

---

## `js/io/project.js`

```javascript
/* ═══ حفظ المشروع وفتحه · التنزيل · اختيار الملفّ ═══
   الملفّ نصٌّ صريح: كل ما رسمته وكل خياراتك، بلا حقل مشتقّ واحد. */
import {S,pack,loadState,ensureShape,clearHistory,
        LSK} from "../core/state.js";
import {toJSON as blocksJSON,fromJSON as blocksLoad} from "../core/blocks.js";
import {toJSON as priceJSON,fromJSON as priceLoad} from "../core/pricing.js";
import {toJSON as ulJSON,fromJSON as ulLoad} from "../core/underlay.js";

/* تبقى النسخة 1 لأجل توافق ملفات المشروع واختبارات المستورد الحالية؛
   الحقول الإضافية اختيارية وتُقرأ بلا كسر للملفات القديمة. */
export const VERSION=1;

export function toJSON(){
 const d=pack();
  return JSON.stringify(Object.assign({
  __app:"mistar", __ver:VERSION,
   __saved:new Date().toISOString()},d,{
   blockDefs:blocksJSON(), pricing:priceJSON(), underlay:ulJSON()}),null,1);
}
export function fromJSON(txt){
 let d=null;
 try{d=JSON.parse(txt)}
 catch(e){throw new Error("الملفّ ليس JSON صالحاً")}
 if(!d||typeof d!=="object")throw new Error("الملفّ فارغ");
 if(d.__app&&d.__app!=="mistar")
  throw new Error(`الملفّ من «${d.__app}» لا من CivilDraft`);
 if(!Array.isArray(d.walls))
  throw new Error("لا مصفوفة جدران — ليس ملفّ CivilDraft");
 loadState(d,true);
  /* توافق مع النسخة الأولى من اقتراح العناصر التي سمت التعريفات blocks. */
  if(d.blockDefs)blocksLoad(d.blockDefs);
  else if(d.blocks&&typeof d.blocks==="object"&&!Array.isArray(d.blocks))
   blocksLoad(d.blocks);
  if(d.pricing)priceLoad(d.pricing);
  if(d.underlay)ulLoad(d.underlay);
 ensureShape();
 clearHistory();
 return {walls:S.walls.length, opens:S.opens.length,
  areas:S.areas.length, dims:S.dims.length,
  chains:S.chains.length, anno:S.anno.length,
  cols:S.cols.length, fixt:S.fixt.length,
  stairs:S.stairs.length,
   blocks:Array.isArray(S.blocks)?S.blocks.length:0,
  ref:(S.ref&&S.ref.ents)?S.ref.ents.length:0,
  ver:d.__ver||0};
}
export function dl(name,data,mime){
 const b=(data instanceof Blob)?data
  :new Blob([data],{type:mime||"application/octet-stream"});
 const u=URL.createObjectURL(b);
 const a=document.createElement("a");
 a.href=u; a.download=name;
 document.body.appendChild(a);
 a.click();
 setTimeout(()=>{URL.revokeObjectURL(u); a.remove()},1200);
 return b.size;
}
/* ═══ سقف الحجم ═══
   pairs(txt) يبني مصفوفةً من زوجٍ لكل سطرَين، فملفٌّ ٢٠٠ م.ب يعطي
   ملايين المصفوفات قبل أن يبدأ التحويل — والخيط الرئيسي محتجزٌ بلا
   مؤشّرٍ ولا إلغاء. فالسؤال قبل القراءة لا بعدها. */
export const MAXFILE=24*1024*1024;      /* ٢٤ م.ب */

export function pickFile(cb,max){
 const i=document.createElement("input");
 i.type="file"; i.accept=".json,.mistar,application/json";
 i.onchange=()=>{
  const f=i.files&&i.files[0];
  if(!f){cb(null,null);return}
  const lim=max||MAXFILE;
  if(f.size>lim&&!confirm(
   `الملفّ ${humanSize(f.size)} — أكبر من ${humanSize(lim)}. `
   +`متابعة؟`)){cb(null,null); return}
  const r=new FileReader();
  r.onload=()=>cb(String(r.result||""),f.name);
  r.onerror=()=>cb(null,null);
  r.readAsText(f,"utf-8");
 };
 i.click();
}
/* قارئ ثنائي — DXF قد يكون CP1256 فلا يُقرأ نصّاً مباشرةً */
export function pickBin(accept,cb,max){
 const i=document.createElement("input");
 i.type="file";
 i.accept=accept||".dxf";
 i.onchange=()=>{
  const f=i.files&&i.files[0];
  if(!f){cb(null,null);return}
  const lim=max||MAXFILE;
  if(f.size>lim&&!confirm(
   `الملفّ ${humanSize(f.size)} — أكبر من ${humanSize(lim)}. `
   +`تحليله قد يُجمِّد الصفحة دقائق ولا يمكن إلغاؤه. متابعة؟`)){
   cb(null,null); return;
  }
  const r=new FileReader();
  r.onload=()=>cb(r.result,f.name);
  r.onerror=()=>cb(null,null);
  r.readAsArrayBuffer(f);
 };
 i.click();
}
export const humanSize=n=>(n<1024)?`${n} بايت`
 :((n<1048576)?`${(n/1024).toFixed(1)} ك.ب`
 :`${(n/1048576).toFixed(2)} م.ب`);
```

<a id="f-js-io-sect-js"></a>

---

## `js/io/sect.js`

```javascript
/* ═══ المقطع صفحةً تُصدَّر ═══
   مرآةُ io/elev.js حرفاً بحرف في بنيتها.

   ═══ التسمية ═══ TEST-SECT-0deg.svg
   الرمز زاويةُ خطّ القطع بالدرجات لا حرفَ جهة. وback يُلحَق بـB.
   وخطّان متوازيان في موضعين مختلفين يتشاركان الرمز نفسه — من أراد
   التمييز مرّر opt.mark ("A-A") فيُكتَب بدلها. */
import {S} from "../core/state.js";
import {SLAY,sectPrims,lastSect} from "../core/section.js";
import {vis,plots,filterPrims} from "../core/layers.js";
import {toSVG} from "./svg.js";
import {toDXFBytes} from "./dxf.js";
import {toPDF,toPDFz} from "./pdf.js";

export const PAD=200;            /* هامشٌ نموذجيّ حول المقطع — مم */

export const safeName=s=>String(s==null?"":s).trim()
 .replace(/[\\/:*?"<>|]+/g,"_").replace(/\s+/g,"_")
 .slice(0,40)||"PLAN";
export const sectTag=(e,mark)=>{
 const m=String(mark==null?"":mark).trim();
 if(m)return safeName(m);
 return `${Math.round((e&&e.ang)||0)}deg`+((e&&e.back)?"B":"");
};
export const sectFileName=(e,ext,mark)=>
 `${safeName(S.meta.name)}-SECT-${sectTag(e,mark)}.${ext}`;

export function sectPage(e,opt){
 const o=opt||{};
 const t=e||lastSect();
 if(!t)throw new Error("لا مقطع — نفّذ SECTION أوّلاً");
 const pad=(o.pad==null)?PAD:Math.max(0,Math.round(+o.pad||0));
 const raw=sectPrims(t,0,0);
 const prims=filterPrims(raw);
 const box=raw.length
  ? {x0:-pad, y0:-pad, x1:t.w+pad, y1:t.h+pad}
  : {x0:0,y0:0,x1:1000,y1:1000};
 const notes=[];
 if(!raw.length)notes.push("المقطع خالٍ — لا شكل يُصدَّر");
 if(!vis(SLAY))
  notes.push(`طبقة ${SLAY} مخفيّة — لن يخرج منها شيء`);
 else if(!plots(SLAY))
  notes.push(`طبقة ${SLAY} لا تُطبَع — الملفّ يخرج فارغاً`);
 return {e:t, prims, box, pad, notes};
}
const infoOf=e=>({
 title:`${S.meta.name||"PLAN"} — ${e.name}`,
 sheet:S.title.sheet, rev:S.title.rev,
 proj:S.title.proj, by:S.title.by});

export function sectSVG(e,opt){
 const o=opt||{};
 const P=sectPage(e,o);
 const r=toSVG(P.prims,P.box,{dark:0,pad:0,showWarn:false,
  page:o.page||null});
 return {name:sectFileName(P.e,"svg",o.mark),
  mime:"image/svg+xml;charset=utf-8",
  txt:r.txt, notes:P.notes.concat(r.notes||[]), page:P};
}
export function sectDXF(e,opt){
 const o=opt||{};
 const P=sectPage(e,o);
 const r=toDXFBytes(P.prims,P.box);
 return {name:sectFileName(P.e,"dxf",o.mark),
  mime:"application/dxf",
  bytes:r.bytes, bad:r.bad,
  notes:P.notes.concat(r.notes||[]), page:P};
}
export function sectPDF(e,opt){
 const o=opt||{};
 const P=sectPage(e,o);
 const r=toPDF(P.prims,P.box,{pad:0,showWarn:false,
  info:infoOf(P.e), page:o.page||null});
 return {name:sectFileName(P.e,"pdf",o.mark),
  mime:"application/pdf",
  bytes:r.bytes, arabic:r.arabic,
  notes:P.notes.concat(r.notes||[]), page:P};
}
export async function sectPDFz(e,opt){
 const o=opt||{};
 const P=sectPage(e,o);
 const r=await toPDFz(P.prims,P.box,{pad:0,showWarn:false,
  info:infoOf(P.e), page:o.page||null});
 return {name:sectFileName(P.e,"pdf",o.mark),
  mime:"application/pdf",
  bytes:r.bytes, arabic:r.arabic, zip:r.zip,
  notes:P.notes.concat(r.notes||[]), page:P};
}
export const sectFile=(fmt,e,opt)=>{
 const f=String(fmt||"svg").toLowerCase();
 if(f==="dxf")return sectDXF(e,opt);
 if(f==="pdf")return sectPDF(e,opt);
 if(f==="svg")return sectSVG(e,opt);
 throw new Error(`صيغةٌ غير معروفة: «${fmt}» — svg أو dxf أو pdf`);
};
```

<a id="f-js-io-snaps-js"></a>

---

## `js/io/snaps.js`

```javascript
/* ═══ اللقطات الزمنية ═══
   نسخة كاملة تعبر إغلاق الصفحة. الاستعادة تمرّ بـ edit() لتصبح
   خطوة تراجع واحدة، وتبقى قائمة الفهرس خفيفة في localStorage. */
import {S,pack,loadState,edit,editFailed} from "../core/state.js";
import {snapPut,snapGet,snapDel} from "./store.js";

const K="mistar.snaps";
export const SNAP={max:12,list:[]};
const read=()=>{
 try{const a=JSON.parse(localStorage.getItem(K)||"[]");return Array.isArray(a)?a:[]}
 catch(e){return []}
};
const write=a=>{try{localStorage.setItem(K,JSON.stringify(a))}catch(e){}};
export const snapList=()=>SNAP.list.slice();
export const snapsLoad=()=>{SNAP.list=read();return SNAP.list};
export async function snapTake(why){
 if(!S.walls.length&&!S.areas.length)return {ok:0,err:"فارغ"};
 const id="s"+Date.now();
 const r=await snapPut(id,pack());
 if(!r.ok)return r;
 SNAP.list.unshift({id,t:Date.now(),why:String(why||"يدوية").slice(0,40),
  w:S.walls.length,o:S.opens.length,a:S.areas.length});
 while(SNAP.list.length>SNAP.max){
  const old=SNAP.list.pop(); await snapDel(old.id);
 }
 write(SNAP.list);
 return {ok:1,id};
}
export async function snapRestore(id){
 const d=await snapGet(id);
 if(!d||!Array.isArray(d.walls))return {ok:0,err:"اللقطة مفقودة"};
 edit(()=>{loadState(d,false)},"استعادة لقطة");
 if(editFailed())return {ok:0,err:"تعذّر التطبيق"};
 return {ok:1,w:d.walls.length};
}
export async function snapDrop(id){
 await snapDel(id);
 SNAP.list=SNAP.list.filter(x=>x.id!==id); write(SNAP.list);
}
let T=null,lastV=-1;
export function snapAutoStart(min){
 if(T)clearInterval(T);
 lastV=S.__ver;
 T=setInterval(()=>{
  if(S.__ver===lastV)return;
  lastV=S.__ver; snapTake("تلقائية");
 },Math.max(2,+min||10)*60000);
}
```

<a id="f-js-io-store-js"></a>

---

## `js/io/store.js`

```javascript
/* ═══ التخزين الدائم ═══
   IndexedDB أوّلاً و localStorage بديلاً. ثلاثة مكاسب:

   ١ · لا حدَّ عملياً — كان الحدّ ٥ م.ب، ومرجعٌ مستورد بستّين ألف
       كيانٍ يتجاوزه، فيفشل الحفظ التلقائي.
   ٢ · لا JSON.stringify — النسخ البنيوي يقبل الكائن كما هو، فيسقط
       تسلسلُ كل شيءٍ كل سبعمئة مللي.
   ٣ · الفشل يُقال. كان catch(e){} صامتاً، فيفقد المستخدم الحفظ
       التلقائي ولا يعلم.

   وليس فيه استيرادٌ واحد بقصد: ورقةٌ في شجرة الاعتماد، فاستيراده
   من core/state.js لا يصنع دورةً ولا يقلب اتجاهاً. */

const DB="mistar", STORE="state", SNAPS="snaps", KEY="doc", DBV=2;
export const LSK="mistar.v1";

let dbP=null, MODE="?", FAIL=0, LAST="";
export const mode=()=>MODE;
export const failed=()=>FAIL;

function open(){
 if(dbP)return dbP;
 dbP=new Promise(res=>{
  if(typeof indexedDB==="undefined"){MODE="ls"; res(null); return}
  let rq;
  /* المتصفّح في وضعٍ خاصّ قد يرمي من الفتح نفسه لا من الحدث */
  try{rq=indexedDB.open(DB,DBV)}
  catch(e){MODE="ls"; res(null); return}
  rq.onupgradeneeded=()=>{
   const d=rq.result;
   if(!d.objectStoreNames.contains(STORE))d.createObjectStore(STORE);
   if(!d.objectStoreNames.contains(SNAPS))d.createObjectStore(SNAPS);
  };
  rq.onsuccess=()=>{MODE="idb"; res(rq.result)};
  rq.onerror  =()=>{MODE="ls";  res(null)};
  rq.onblocked=()=>{MODE="ls";  res(null)};
 });
 return dbP;
}
/* ═══ الطابع والعلامة ═══
   الطابع يحكم بين النسختين: الكتابة الأخيرة عند الإغلاق تقع في
   localStorage، فلا يجوز أن تحجبها نسخةٌ أقدم في IndexedDB.
   و__lite يقول إن النسخة منقوصة، فلا تحجب كاملةً أقدم منها:
   الأحدث ليس الأصحّ إن كان ناقصاً. وكان غيابُ هذه العلامة يُفقِد
   المرجعَ المستورد كلَّه عند أول إغلاقٍ لا يتّسع فيه localStorage. */
const stamp=(o,lite)=>{
 const r=Object.assign({},o,{__t:Date.now()});
 if(lite)r.__lite=1; else delete r.__lite;
 return r;
};
/* المرجع وحده هو ما يُنقَص، فدمجُه من النسخة الكاملة يُتِمّ الناقصة.
   ويُعاد كائنٌ جديد: النسختان المقروءتان لا تُمَسّان. */
const heal=(lite,full)=>{
 if(!lite||!lite.__lite||!full||!full.ref)return lite;
 const n=(full.ref.ents||[]).length;
 if(!n)return lite;
 const o=Object.assign({},lite,{ref:full.ref});
 delete o.__lite;
 o.__healed=n;
 return o;
};
function saveLS(o){
 if(typeof localStorage==="undefined")return {ok:0,via:"none"};
 try{
  const s=JSON.stringify(o);
  localStorage.setItem(LSK,s);
  FAIL=0; LAST="ls";
  return {ok:1,via:"ls",bytes:s.length};
 }catch(e){
  FAIL++;
  return {ok:0,via:"ls",err:(e&&e.name)||"خطأ",
   refs:(o&&o.ref&&o.ref.ents)?o.ref.ents.length:0};
 }
}
export async function save(obj){
 const o=stamp(obj);                /* كاملةٌ صريحاً — بلا __lite */
 const d=await open();
 if(d){
  try{
   await new Promise((res,rej)=>{
    const tx=d.transaction(STORE,"readwrite");
    tx.objectStore(STORE).put(o,KEY);
    tx.oncomplete=res;
    tx.onerror=()=>rej(tx.error);
    tx.onabort =()=>rej(tx.error);
   });
   FAIL=0; LAST="idb";
   return {ok:1,via:"idb"};
  }catch(e){MODE="ls"}     /* الحصّة أو التلف ⇒ نهبط ونُبلّغ */
 }
 return saveLS(o);
}
export async function load(){
 let idb=null, ls=null;
 const d=await open();
 if(d){
  try{
   idb=await new Promise((res,rej)=>{
    const tx=d.transaction(STORE,"readonly");
    const rq=tx.objectStore(STORE).get(KEY);
    rq.onsuccess=()=>res(rq.result||null);
    rq.onerror  =()=>rej(rq.error);
   });
  }catch(e){}
 }
 if(typeof localStorage!=="undefined"){
  try{
   const raw=localStorage.getItem(LSK);
   if(raw){
    const o=JSON.parse(raw);
    if(o&&Array.isArray(o.walls))ls=o;
   }
  }catch(e){}
 }
 const tI=(idb&&+idb.__t)||0, tL=(ls&&+ls.__t)||0;
 /* الناقصة الأحدث تُرمَّم من الكاملة الأقدم: رسمك من الأحدث،
    والمرجع من الأقدم — فلا يُفقَد أيٌّ منهما.
    والشرط الزمنيّ لازم: الترميم يُثبَّت في IndexedDB فوراً فيصير
    أحدث، فلو رمّمنا بلا شرطٍ لتكرّر التنبيه في كل إقلاع. */
 if(ls&&ls.__lite&&idb&&tL>tI){
  const h=heal(ls,idb);
  if(h.__healed)return {data:h,via:"healed",refs:h.__healed};
 }
 if(idb&&ls&&idb.__lite&&!ls.__lite&&tI>=tL){
  /* الحالة المقابلة: idb ناقصة وls كاملة */
  const h=heal(idb,ls);
  if(h.__healed)return {data:h,via:"healed",refs:h.__healed};
 }
 if(idb&&(!ls||tI>=tL))return {data:idb,via:"idb"};
 /* migrate لا تُعلَن إلّا إن كان IndexedDB متاحاً ولا شيء فيه —
    وهو معنى الكلمة. وكانت تُعلَن كلَّ إقلاعٍ لأن flushSync يجعل
    ls أحدث دائماً، فيُكتَب IndexedDB ولا يُقرَأ في المسار المعتاد
    — وهو الغرض المُعلَن للملفّ كلّه. */
 if(ls)return {data:ls,via:(d&&!idb)?"migrate":"ls"};
 return null;
}
export async function del(){
 const d=await open();
 if(d){
  try{
   await new Promise(res=>{
    const tx=d.transaction(STORE,"readwrite");
    tx.objectStore(STORE).delete(KEY);
    tx.oncomplete=res; tx.onerror=res; tx.onabort=res;
   });
  }catch(e){}
 }
 try{localStorage.removeItem(LSK)}catch(e){}
}
/* ═══ الكتابة الأخيرة ═══
   عند إغلاق الصفحة لا يُعتمَد على IndexedDB: معاملاته غير متزامنة
   وقد تُقطَع. فنكتب متزامناً في localStorage بطابعٍ أحدث.
   وإن لم يتّسع: نكتب بلا المرجع المستورد ونُعلِّمها منقوصةً — فيبقى
   رسمك كلّه، ويُرمَّم المرجع من الكاملة عند الفتح. الصمت هنا هو
   الخطأ الحقيقي، وحجبُ الكاملة بالمنقوصة أسوأ منه. */
export function flushSync(obj){
 if(typeof localStorage==="undefined")return {ok:0,via:"none"};
 try{
  localStorage.setItem(LSK,JSON.stringify(stamp(obj)));
  return {ok:1,via:"ls"};
 }catch(e){}
 const n=(obj&&obj.ref&&obj.ref.ents)?obj.ref.ents.length:0;
 try{
  const lite=stamp(Object.assign({},obj,
   {ref:Object.assign({},obj.ref||{},{ents:[]})}),1);
  localStorage.setItem(LSK,JSON.stringify(lite));
  return {ok:1,via:"ls",lite:1,refs:n};
 }catch(e2){FAIL++; return {ok:0,via:"ls",err:(e2&&e2.name)||"خطأ"}}
}
/* ═══ المسح الكامل ═══
   كل ما يُخزّنه البرنامج في هذا المتصفّح: المشروع وجلسته وتفضيلات
   الواجهة وأسطح العمل وخيارات الأدوات وإعداد المزوّد ومفتاحه.
   ولا سبيل إليه من الواجهة قبل الدفعة ٤ إلّا من أدوات المطوّر — وهو
   مطلبٌ عمليّ لمن يستعمل حاسباً مشتركاً.
   ولا يمسّ ما حفظه المستخدم ملفّاً على قرصه. */
export const LSKEYS=[LSK,"mistar.ui","mistar.opts","mistar.ai",
 "mistar.snaps","mistar.code"];
export async function purge(){
 const out={ls:0,idb:0,keys:[]};
 if(typeof localStorage!=="undefined")LSKEYS.forEach(k=>{
  try{
   if(localStorage.getItem(k)==null)return;
   localStorage.removeItem(k);
   out.ls++; out.keys.push(k);
  }catch(e){}
 });
 try{await del()}catch(e){}
 /* الاتّصال يُغلَق قبل الحذف: قاعدةٌ مفتوحة تحجب deleteDatabase */
 try{
  const d=await open();
  if(d&&d.close)d.close();
 }catch(e){}
 dbP=null; MODE="?";
 try{
  if(typeof indexedDB!=="undefined"&&indexedDB.deleteDatabase){
   indexedDB.deleteDatabase(DB);
   out.idb=1;
  }
 }catch(e){}
 return out;
}

/* ═══ اللقطات ═══ */
export async function snapPut(id,obj){
 const d=await open();
 if(!d)return {ok:0,via:"none"};
 try{
  await new Promise((res,rej)=>{
   const tx=d.transaction(SNAPS,"readwrite");
   tx.objectStore(SNAPS).put(obj,id);
   tx.oncomplete=res; tx.onerror=()=>rej(tx.error); tx.onabort=()=>rej(tx.error);
  });
  return {ok:1,via:"idb"};
 }catch(e){return {ok:0,via:"idb",err:(e&&e.name)||"خطأ"}}
}
export async function snapGet(id){
 const d=await open();
 if(!d)return null;
 try{
  return await new Promise((res,rej)=>{
   const tx=d.transaction(SNAPS,"readonly");
   const rq=tx.objectStore(SNAPS).get(id);
   rq.onsuccess=()=>res(rq.result||null); rq.onerror=()=>rej(rq.error);
  });
 }catch(e){return null}
}
export async function snapDel(id){
 const d=await open();
 if(!d)return;
 try{
  await new Promise(res=>{
   const tx=d.transaction(SNAPS,"readwrite");
   tx.objectStore(SNAPS).delete(id);
   tx.oncomplete=res; tx.onerror=res; tx.onabort=res;
  });
 }catch(e){}
}
```

<a id="f-js-io-style-js"></a>

---

## `js/io/style.js`

```javascript
/* ═══ هيئة الأوّلية ═══
   الأربعة كانوا يترجمون كلٌّ على حدة، فاختلفوا في ستّ تفاصيل:
   شرطة الطبقة تُطبَّق في اثنين وتُهمَل في اثنين · الشفافية في
   اثنين · صبغة المنطقة بأربع صيغ · تعبئة solid بنمطين ·
   طبقة الهاشور مكتوبةٌ حرفياً في ثلاثة · التنبيه في ثلاثة.

   هنا مصدرٌ واحد. والعجز في الصيغة يُعلَن لا يُسكَت عنه: CAPS
   تقول ما تحمله الصيغة، وnotes تجمع ما فُقِد فيُقال للمستخدم.

   ولا يستورد إلّا core/layers، فلا دورة: dxf و svg و png و pdf
   يستوردونه، وهو لا يعرفهم. */
import {resolve,plots} from "../core/layers.js";

/* ما تحمله كل صيغة — قرارٌ معلَنٌ لا سلوكٌ مُستنتَج.
   cut=1 يعني: الشرطة تُقطَّع قطعاً حقيقية بدل نمط LTYPE — فهي
   مُطبَّقةٌ لا مُهمَلة، لكن بوسيلةٍ أخرى. */
export const CAPS={
 dxf:{dash:0, cut:1, alpha:0, fill:0, warn:0, lw:1,
  why:{dash:"الشرطة تُقطَّع قطعاً حقيقية بدل نمط LTYPE",
   alpha:"لا شفافية في DXF",
   fill:"لا تعبئة شفافة — حدُّ المنطقة وحده",
   warn:"ألوان التنبيه لا تُصدَّر"}},
 svg:{dash:1, cut:0, alpha:1, fill:1, warn:1, lw:1, why:{}},
 png:{dash:1, cut:0, alpha:1, fill:1, warn:1, lw:1, why:{}},
 pdf:{dash:1, cut:0, alpha:1, fill:1, warn:1, lw:1, why:{}}
};
/* لون التنبيه والعطب — واحدٌ للأربعة، وكان ثلاثة حرفيّات */
export const WARN={warn:"#b8860b", bad:"#b00020"};
/* شفافية صبغة المنطقة — واحدةٌ للأربعة، وكانت أربع صيغ */
export const TINT_A=0.11;
/* طبقة الهاشور الافتراضية حين لا تُعلَنها الأوّلية */
export const HLAY="A-WALL-PATT";

function note(o,k,C){
 if(!o||!o.notes)return;
 const m=(C.why||{})[k];
 if(m&&!o.notes.includes(m))o.notes.push(m);
}
const capsOf=fmt=>CAPS[fmt]||CAPS.svg;
const modeOf=o=>(o&&o.dark)?"dark":"plot";
const kOf=o=>Math.max(1,(o&&o.k)||1);
const pxOf=o=>((o&&o.px)==null)?1:o.px;
const minOf=o=>((o&&o.minLw)==null)?0.15:o.minLw;

/* ═══ الهيئة ═══
   g   الأوّلية · fmt اسم الصيغة · O.dark وضع اللون · O.k المقياس
   O.px معامل النقطة/البكسل (١ لوحدات النموذج)
   O.showWarn هل تُصدَّر ألوان التنبيه (خيارُ مستخدم)

   تعيد: {skip · css · lw · dash · cut · alpha · aci · lt · dxfLt}
   وpush إلى O.notes ما فُقِد.

   dash تُعاد بوحدات المخرَج (مضروبةً بـpx)، وcut تعني «طبِّقها
   بالتقطيع لا بنمطٍ» — فالقرار في موضعٍ واحد ولا يعود المستدعي
   يقرأ resolve بنفسه. */
export function styleOf(g,fmt,O){
 const C=capsOf(fmt);
 const o=O||{};
 const L=(g&&g.L)||"0";
 if(!plots(L))return {skip:1};
 const r=resolve(L,modeOf(o));
 const k=kOf(o), px=pxOf(o);
 const bad=!!(g&&g.bad), wr=!!(g&&g.warn);
 const wantWarn=(bad||wr)&&(o.showWarn!==false);
 const css=wantWarn
  ? (C.warn?(bad?WARN.bad:WARN.warn):r.css)
  : r.css;
 if(wantWarn&&!C.warn)note(o,"warn",C);
 /* الوزن: (lw/100) مم ورقيّ × المقياس = وحدة نموذج، ثم × px */
 const w=(r.lw||25)/100*k*px;
 const lw=Math.max(minOf(o),w)*((bad||wr)?1.2:1);
 /* الشرطة: دَشّ الأوّلية بوحدات النموذج، ودَشّ الطبقة بالمليمتر
    الورقيّ — كارتفاع النصّ تماماً، فيُضرَب بالمقياس. */
 let dash=null;
 if(g&&g.dash&&g.dash.length)
  dash=g.dash.map(v=>Math.max(0.1,v*px));
 else if(r.dash&&r.dash.length)
  dash=r.dash.map(v=>Math.max(0.1,v*k*px));
 let cut=0;
 if(dash&&!C.dash){
  note(o,"dash",C);
  if(C.cut)cut=1; else dash=null;
 }
 let alpha=(r.a<1)?r.a:1;
 if(alpha<1&&!C.alpha){note(o,"alpha",C); alpha=1}
 return {skip:0, css, lw, dash, cut, alpha,
  aci:r.aci, lt:r.lt, dxfLt:r.dxf, n:L};
}
/* ═══ التعبئات ═══
   صبغة المنطقة كانت أربع صيغ: حدٌّ فقط · لون الطبقة ١٠٪ ·
   rgba حرفيّ · لونٌ معتمٌ حرفيّ. والآن لونُ طبقتها بشفافيةٍ
   واحدة — والصيغة التي لا تحملها تُصدِّر الحدّ وتُبلِّغ.

   والهاشور فيها يأخذ لون المنطقة وتباعُدَ الهاشور: خلطٌ مقصود،
   فالنقش هويّةُ المنطقة لا هويّةُ الجدران. */
export function fillOf(g,fmt,O){
 const C=capsOf(fmt);
 const o=O||{};
 const L=(g&&g.L)||"A-AREA";
 if(!plots(L))return {skip:1};
 const r=resolve(L,modeOf(o));
 if(g&&g.style==="hatch"){
  const h=resolve(HLAY,modeOf(o));
  return {skip:0, hatch:1, css:r.css, a:1,
   sp:Math.max(1,(o.hs||300)),
   lw:Math.max(minOf(o),(h.lw||13)/100*kOf(o)*pxOf(o))};
 }
 if(!C.fill){
  note(o,"fill",C);
  return {skip:0, hatch:0, css:r.css, a:1, edgeOnly:1};
 }
 return {skip:0, hatch:0, css:r.css, a:TINT_A};
}
/* ═══ الهاشور ═══
   الطبقة من g.L لا من اسمٍ حرفيّ: هاشور العمود المنفرد يحمل
   A-COLS، وكان يُصدَّر بلون تعبئة الجدران ويغيب بإيقاف طبعها.
   وsolid تشابكٌ في اتجاهين في الأربعة — كان اثنان بخطٍّ واحد. */
export function hatchOf(g,fmt,O){
 const o=O||{};
 const L=(g&&g.L)||HLAY;
 if(!plots(L))return {skip:1};
 const r=resolve(L,modeOf(o));
 const sp=Math.max(1,(g&&g.sc)||300);
 const solid=(g&&g.pat==="SOLID");
 const sets=solid
  ? [[45,sp*0.22],[135,sp*0.22]] : [[45,sp]];
 return {skip:0, css:r.css, sets, sp, solid:solid?1:0,
  lw:Math.max(minOf(o),(r.lw||13)/100*kOf(o)*pxOf(o))};
}
/* ═══ الهاشور المولَّد خطوطاً ═══
   نسخةٌ واحدة كانت في dxf وpdf بتبريرٍ غير صحيح («تفادي دورة
   الاستيراد») — وهنا لا دورة لأن style.js لا يستورد إلّا layers.

   والحدّ يُعلَن بدل أن يُبتَر صامتاً: كان return [] فيغيب
   الهاشور كلّه بلا كلمة. */
export function hatchLines(loops,angDeg,spacing){
 const a=angDeg*Math.PI/180;
 const ca=Math.cos(-a), sa=Math.sin(-a);
 const cb=Math.cos(a),  sb=Math.sin(a);
 const rot=p=>[p[0]*ca-p[1]*sa, p[0]*sa+p[1]*ca];
 const inv=p=>[p[0]*cb-p[1]*sb, p[0]*sb+p[1]*cb];
 const E=[];
 let y0=1/0, y1=-1/0;
 (loops||[]).forEach(lp=>{
  if(!lp||lp.length<3)return;
  const R=lp.map(rot);
  for(let i=0;i<R.length;i++){
   const A=R[i], B=R[(i+1)%R.length];
   E.push([A,B]);
   if(A[1]<y0)y0=A[1]; if(A[1]>y1)y1=A[1];
  }
 });
 if(!E.length)return {lines:[],cut:0};
 const s=Math.max(1,spacing), out=[];
 const k0=Math.ceil(y0/s), k1=Math.floor(y1/s);
 if(k1-k0>6000)return {lines:[],cut:k1-k0};
 for(let k=k0;k<=k1;k++){
  const y=k*s, xs=[];
  E.forEach(([A,B])=>{
   if((A[1]>y)===(B[1]>y))return;
   const t=(y-A[1])/(B[1]-A[1]);
   xs.push(A[0]+t*(B[0]-A[0]));
  });
  xs.sort((p,q)=>p-q);
  for(let i=0;i+1<xs.length;i+=2){
   if(xs[i+1]-xs[i]<1)continue;
   out.push([inv([xs[i],y]), inv([xs[i+1],y])]);
  }
 }
 return {lines:out,cut:0};
}
/* سياقٌ جاهز — يُغني كل مصدِّرٍ عن تركيبه بيده */
export const ctxOf=(fmt,o)=>Object.assign(
 {dark:0,k:1,px:1,minLw:0.15,notes:[],showWarn:false,
  hs:300,hatchCut:0,fmt},o||{});
```

<a id="f-js-io-svg-js"></a>

---

## `js/io/svg.js`

```javascript
/* ═══ تصدير SVG ═══
   المسار المتّجه الأصدق للعربية: المتصفّح يرسم النصّ بخطّه، فلا
   مشكلة ترميز ولا تنقيط. المقاس بالمليمتر الورقي والإحداثيات
   بالمليمتر النموذجي.

   والهيئة من io/style.js — مصدرٌ واحد يقرأه الأربعة. وكانت
   هنا نسخةٌ ثالثة اختلفت في صبغة المنطقة وتعبئة solid وطبقة
   الهاشور: نمطان عامّان بلون طبقة الهاشور الثابتة (عبر HLAY) دائماً،
   فهاشور العمود المنفرد يخرج بلونٍ خاطئ ويغيب بإيقاف طبع طبقةٍ أخرى. */
import {S} from "../core/state.js";
import {styleOf,fillOf,hatchOf,ctxOf,HLAY} from "./style.js";

const X=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const N=v=>String(Math.round((+v||0)*100)/100);

export function toSVG(prims,box,opt){
 const O=Object.assign({dark:0,pad:0,showWarn:false},opt||{});
 const B=box||{x0:0,y0:0,x1:1000,y1:1000};
 const pad=O.pad||0;
 const x0=B.x0-pad, y1=B.y1+pad;
 const W=(B.x1-B.x0)+pad*2, H=(B.y1-B.y0)+pad*2;
 const k=Math.max(1,S.meta.scale);
 /* الرؤية بمقاس الورقة الاسمية × المقياس، والهندسة تُتَمركَز فيها:
    فالنسبة 1:k بالضبط لا 1:k±خطأَ تدوير. */
 const nom=(O.page&&+O.page.w>0&&+O.page.h>0)?O.page:null;
 const pw=nom?+nom.w:(W/k), ph=nom?+nom.h:(H/k);
 const VW=nom?pw*k:W, VH=nom?ph*k:H;
 const X0=x0-(VW-W)/2, Y1=y1+(VH-H)/2;
 const T=p=>`${N(p[0]-X0)},${N(Y1-p[1])}`;
 const notes=[];
 /* px=1: SVG يعمل في وحدات النموذج مباشرةً */
 const SO=ctxOf("svg",{dark:!!O.dark,k,px:1,minLw:1,notes,
  showWarn:O.showWarn,
  hs:Math.max(8,(S.meta.txtMM*k)*2.5)});
 const out=[];

 out.push(`<?xml version="1.0" encoding="UTF-8"?>\n`);
 out.push(`<svg xmlns="http://www.w3.org/2000/svg" `
  +`width="${N(pw)}mm" height="${N(ph)}mm" `
  +`viewBox="0 0 ${N(VW)} ${N(VH)}">\n`);
 out.push(`<title>${X(S.meta.name||"PLAN")} — `
  +`1:${S.meta.scale}</title>\n`);

 /* ═══ أنماط الهاشور ═══
    نمطٌ لكل طبقةٍ ومقاسٍ يحتاجه، لا نمطان عامّان. وsolid
    تشابكٌ في اتجاهين — كالأربعة الآخرين. */
 const pats=new Map();
 const patKey=(L,solid,sp)=>`${L}|${solid?"s":"h"}|`
  +Math.round(sp);
 const addPat=(L,solid,sp,css,lw)=>{
  const key=patKey(L,solid,sp);
  if(pats.has(key))return key;
  pats.set(key,{id:"p"+pats.size,css,sp,lw,solid:solid?1:0});
  return key;
 };
 (prims||[]).forEach(g=>{
  if(g.t==="hatch"){
   const hs=hatchOf(g,"svg",SO);
   if(hs.skip)return;
   addPat(g.L||HLAY,hs.solid,hs.sp,hs.css,hs.lw);
   return;
  }
  if(g.t==="fill"&&g.style==="hatch"){
   const f=fillOf(g,"svg",SO);
   if(f.skip||!f.hatch)return;
   addPat(g.L||"A-AREA",0,f.sp,f.css,f.lw);
  }
 });
 if(pats.size){
  out.push(`<defs>\n`);
  pats.forEach(p=>{
   const s=p.solid?p.sp*0.22:p.sp;
   const sw=N(Math.max(0.5,p.lw));
   /* التشابك: خطٌّ رأسيّ في نمطٍ مُدوَّر ٤٥، وآخر أفقيّ فيه */
   out.push(`<pattern id="${p.id}" patternUnits="userSpaceOnUse" `
    +`width="${N(s)}" height="${N(s)}" `
    +`patternTransform="rotate(45)">`
    +`<line x1="0" y1="0" x2="0" y2="${N(s)}" `
    +`stroke="${p.css}" stroke-width="${sw}"/>`
    +(p.solid?`<line x1="0" y1="0" x2="${N(s)}" y2="0" `
      +`stroke="${p.css}" stroke-width="${sw}"/>`:"")
    +`</pattern>\n`);
  });
  out.push(`</defs>\n`);
 }
 const patId=key=>{
  const p=pats.get(key);
  return p?p.id:null;
 };
 out.push(`<rect x="0" y="0" width="${N(VW)}" height="${N(VH)}" `
  +`fill="${O.dark?"#0e1216":"#ffffff"}"/>\n`);
 out.push(`<g fill="none" stroke-linecap="round" `
  +`stroke-linejoin="round">\n`);

 (prims||[]).forEach(g=>{
  if(g.t==="fill"){
   const f=fillOf(g,"svg",SO);
   if(f.skip)return;
   const r=g.ring||[];
   if(r.length<3)return;
   const id=f.hatch
    ? patId(patKey(g.L||"A-AREA",0,f.sp)) : null;
   out.push(`<polygon points="${r.map(T).join(" ")}" `
    +`fill="${(f.hatch&&id)?`url(#${id})`:f.css}" `
    +`fill-opacity="${N(f.a)}" stroke="none"/>\n`);
   return;
  }
  if(g.t==="hatch"){
   const hs=hatchOf(g,"svg",SO);
   if(hs.skip)return;
   const id=patId(patKey(g.L||HLAY,hs.solid,hs.sp));
   if(!id)return;
   (g.loops||[]).forEach(lp=>{
    if(!lp||lp.length<3)return;
    out.push(`<polygon points="${lp.map(T).join(" ")}" `
     +`fill="url(#${id})" fill-rule="evenodd" stroke="none"/>\n`);
   });
   return;
  }
  const st=styleOf(g,"svg",SO);
  if(st.skip)return;
  const w=N(st.lw);
  const ds=st.dash
   ?` stroke-dasharray="${st.dash.map(N).join(",")}"`:"";
  const op=(st.alpha<1)?` stroke-opacity="${N(st.alpha)}"`:"";
  if(g.t==="line"){
   const a=T(g.a).split(","), b=T(g.b).split(",");
   out.push(`<line x1="${a[0]}" y1="${a[1]}" x2="${b[0]}" `
    +`y2="${b[1]}" stroke="${st.css}" `
    +`stroke-width="${w}"${ds}${op}/>\n`);
   return;
  }
  if(g.t==="poly"){
   const pts=(g.pts||[]).map(T).join(" ");
   if(!pts)return;
   out.push(`<${g.cl===0?"polyline":"polygon"} points="${pts}" `
    +`fill="none" stroke="${st.css}" `
    +`stroke-width="${w}"${ds}${op}/>\n`);
   return;
  }
  if(g.t==="arc"){
   const cx=g.cx-X0, cy=Y1-g.cy, r=Math.max(0.5,g.r);
   const sw2=(g.a1-g.a0);
   if(Math.abs(sw2)>=359.5){
    out.push(`<circle cx="${N(cx)}" cy="${N(cy)}" r="${N(r)}" `
     +`fill="none" stroke="${st.css}" `
     +`stroke-width="${w}"${ds}${op}/>\n`);
    return;
   }
   const rd=a=>a*Math.PI/180;
   const p0=[cx+r*Math.cos(rd(g.a0)), cy-r*Math.sin(rd(g.a0))];
   const p1=[cx+r*Math.cos(rd(g.a1)), cy-r*Math.sin(rd(g.a1))];
   const big=(((sw2%360)+360)%360>180)?1:0;
   out.push(`<path d="M ${N(p0[0])} ${N(p0[1])} `
    +`A ${N(r)} ${N(r)} 0 ${big} 0 ${N(p1[0])} ${N(p1[1])}" `
    +`fill="none" stroke="${st.css}" `
    +`stroke-width="${w}"${ds}${op}/>\n`);
   return;
  }
  if(g.t==="text"){
   const p=T([g.x,g.y]).split(",");
   const an=/l$/.test(g.al||"")?"start"
    :(/r$/.test(g.al||"")?"end":"middle");
   const bl=/^m/.test(g.al||"")?"central":"alphabetic";
   const rot=g.rot?` rotate(${N(-g.rot)})`:"";
   const fo=(st.alpha<1)?` fill-opacity="${N(st.alpha)}"`:"";
   out.push(`<text transform="translate(${p[0]},${p[1]})${rot}" `
    +`text-anchor="${an}" dominant-baseline="${bl}" `
    +`font-family="Tahoma,Arial,sans-serif" `
    +`font-size="${N(g.h)}" fill="${st.css}" stroke="none" `
    +`direction="rtl"${fo}>${X(g.s)}</text>\n`);
  }
 });
 out.push(`</g>\n</svg>\n`);
 return {txt:out.join(""), notes, hatchCut:0};
}
```

---

# الواجهة (`js/ui/`)

<a id="f-js-ui-ai-js"></a>

---

