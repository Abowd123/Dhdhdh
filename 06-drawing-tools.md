# CivilDraft — Drawing Tools

> أدوات الرسم والتعديل التفاعلية.

**عدد الملفات:** 14

---

## `js/tools/annotate.js`

```javascript
/* ═══ أدوات التأشير ═══
   البُعد ثلاث نقرات: طرف، طرف، موضع الخطّ — كلّها صريحة.
   السلسلة تُبنى من قيَم تكتبها، لا من مطابقة تُخمَّن. */
import {S} from "../core/state.js";
import {m2,m3,M,clamp,mnum} from "../core/units.js";
import {addDim,posFromPt,dimValue,fmtLen,dimGeom,DK,
        parseVals,addChain,chainSum,chainCompare,
        addText,addLead,addLevel,addAxis,
        axLabel,levelStr} from "../core/dims.js";
import {defTool,H,rec,dirty,ov,ovLen,ovNum,ovOn,
        pvLine} from "./registry.js";

const GRN="#5cd98e", YEL="#ffd06b", PNK="#ff9aa2";

/* ═══ بُعد ═══ */
defTool({
 id:"dim", alias:"d1 بعد قياسات", label:"بُعد",
 hint:"طرف · طرف · موضع الخطّ",
 opts:[
  {k:"kind",label:"النوع",type:"sel",
   items:[["h","أفقي"],["v","رأسي"],["al","محاذٍ"]],def:"h"},
  {k:"txt", label:"نصّ بديل",type:"text",def:"",
   hint:"يُعرَض بدل المقاس مع علامة *"}],
 steps:[
  {p:"الطرف الأول"},
  {p:"الطرف الثاني", base:0},
  {p:"موضع خطّ البُعد", base:"none", restart:1,
   each(ctx,p){
    const a=ctx.pts[0], b=ctx.pts[1];
    const kind=ov("dim","kind");
    const d=addDim(kind,a,b,posFromPt(kind,a,b,p),
     String(ov("dim","txt")||"").trim());
    rec(ctx,d,"dims");
    H.rep("ok",`${d.id} ${DK[d.kind]} ${fmtLen(dimValue(d))} م`
     +(d.txt?` · نصّ بديل «${d.txt}» — علامة * تدلّ عليه`:""));
   }}],
 prev(ctx,g){
  const P=ctx.pts;
  if(!g)return [];
  if(P.length===1)return [pvLine(P[0],g,YEL)];
  if(P.length>=2){
   const kind=ov("dim","kind");
   const gm=dimGeom({kind,a:P[0],b:P[1],
    pos:posFromPt(kind,P[0],P[1],g)});
   if(!gm)return [pvLine(P[0],P[1],YEL)];
   return [pvLine(P[0],P[1],"#4b5a6b"),
           pvLine(gm.p1,gm.p2,GRN),
           pvLine(P[0],gm.p1,"#3d4a58"),
           pvLine(P[1],gm.p2,"#3d4a58")];
  }
  return [];
 }});

/* ═══ سلسلة أبعاد بقيَم مكتوبة ═══ */
defTool({
 id:"chain", alias:"ch سلسله", label:"سلسلة",
 hint:"اكتب القيَم في الشريط ثم انقر البداية وموضع الخطّ",
 opts:[
  {k:"vals", label:"القيَم م",type:"text",def:"",
   hint:"مثل: 3 2.5 4 · أو 3*4 لتكرار"},
  {k:"axis", label:"المحور",type:"sel",
   items:[["h","أفقي"],["v","رأسي"]],def:"h"},
  {k:"total",label:"خطّ المجموع",type:"chk",def:1}],
 start(ctx){
  try{ctx.v.vals=parseVals(ov("chain","vals"))}
  catch(e){H.rep("er",e.message); return false}
  H.rep("in",`${ctx.v.vals.length} قيمة · المجموع `
   +`${fmtLen(ctx.v.vals.reduce((s,v)=>s+v,0))} م`);
  return true;
 },
 steps:[
  {p:"نقطة بداية السلسلة"},
  {p:"موضع خطّ السلسلة", base:0, restart:1,
   each(ctx,p){
    const ax=ov("chain","axis");
    const c=addChain(ax,ctx.pts[0],(ax==="h")?p[1]:p[0],
     ctx.v.vals, ovOn("chain","total")?1:0);
    rec(ctx,c,"chains");
    H.rep("ok",`${c.id} ${ctx.v.vals.length} قيمة · `
     +`${fmtLen(chainSum(c))} م · القيَم كما كتبتها`);
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g||!ctx.v.vals)return [];
  const ax=ov("chain","axis");
  const b=ctx.pts[0];
  const pos=(ax==="h")?g[1]:g[0];
  const pt=v=>(ax==="h")?[b[0]+v,pos]:[pos,b[1]+v];
  const o=[pvLine(b,pt(0),"#3d4a58")];
  let s=0;
  ctx.v.vals.forEach(v=>{
   o.push(pvLine(pt(s),pt(s+v),GRN));
   s+=v;
  });
  return o;
 }});

/* ═══ مقارنة السلسلة بالهندسة — تقرير ═══ */
defTool({
 id:"chaincmp", alias:"cc قارن", label:"قارن السلسلة",
 hint:"يعرض فرق كل حدٍّ عن أقرب عقدة — بلا تعديل",
 opts:[{k:"tol",label:"التفاوت م",type:"len",def:"0.06"}],
 start(ctx){
  const L=H.sel().filter(s=>s.k==="chain");
  if(!L.length){H.rep("wr","حدّد سلسلة أولاً");return false}
  L.forEach(s=>{
   const c=S.chains.find(x=>x.id===s.id);
   if(!c)return;
   const r=chainCompare(c,ovLen("chaincmp","tol"));
   H.rep(r.off?"wr":"ok",
    `${c.id}: المجموع ${fmtLen(r.sum)} م · `
    +`${r.off?`${r.off} حدّاً خارج التفاوت`:"كل الحدود مطابقة"}`);
   r.rows.forEach(x=>{
    if(x.ok)return;
    H.rep("in",`  الحدّ ${x.i}: على ${m3(x.at)} م · `
     +`أقرب عقدة ${x.near==null?"—":m3(x.near)} م · `
     +`الفرق ${x.d==null?"—":m3(x.d)} م`);
   });
  });
  H.rep("in","تقرير فقط — لم تُعدَّل قيمة واحدة");
  return false;
 },
 steps:[]});

/* ═══ نصّ ═══ */
defTool({
 id:"text", alias:"t نص", label:"نصّ",
 hint:"اكتب النصّ في الشريط ثم انقر موضعه · Enter ينهي",
 opts:[
  {k:"s",  label:"النصّ",type:"text",def:""},
  {k:"hm", label:"الحجم ×",type:"num",def:1},
  {k:"rot",label:"الدوران °",type:"num",def:0},
  {k:"al", label:"المحاذاة",type:"sel",
   items:[["bc","وسط"],["bl","يسار"],["mc","وسط أوسط"]],def:"bc"}],
 steps:[
  {p:"موضع النصّ (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    const a=addText(p,ov("text","s"),ovNum("text","hm"),
     ovNum("text","rot"),ov("text","al"));
    rec(ctx,a,"anno");
    H.rep("ok",`${a.id} «${a.s}»`);
   }}]});

/* ═══ قائد ═══ */
defTool({
 id:"lead", alias:"le قائد", label:"قائد",
 hint:"رأس السهم ثم كسرات ثم Enter · النصّ من الشريط",
 opts:[
  {k:"s", label:"النصّ",type:"text",def:""},
  {k:"hm",label:"الحجم ×",type:"num",def:1}],
 steps:[
  {p:"رأس السهم"},
  {p:"نقطة الكسر (Enter ينهي القائد)", loop:1, base:-1, min:1}],
 done(ctx){
  if(ctx.pts.length<2)return;
  const a=addLead(ctx.pts,ov("lead","s"),ovNum("lead","hm"));
  rec(ctx,a,"anno");
  H.rep("ok",`${a.id} قائد «${a.s}» · ${ctx.pts.length} نقطة`);
 },
 prev(ctx,g){
  const P=ctx.pts.concat(g?[g]:[]), o=[];
  for(let i=0;i<P.length-1;i++)o.push(pvLine(P[i],P[i+1],PNK));
  return o;
 }});

/* ═══ منسوب ═══ */
defTool({
 id:"level", alias:"lv منسوب", label:"منسوب",
 hint:"انقر الموضع · القيمة من الشريط · Enter ينهي",
 opts:[
  {k:"z",  label:"المنسوب م",type:"text",def:"0"},
  {k:"pre",label:"سابقة",   type:"text",def:"",hint:"مثل: ت.م"}],
 steps:[
  {p:"موضع المنسوب (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    const raw=String(ov("level","z")||"0").trim();
    const neg=/^-/.test(raw);
    const z=M(raw.replace(/^-/,""))*(neg?-1:1);
    const a=addLevel(p,z,ov("level","pre"));
    rec(ctx,a,"anno");
    H.rep("ok",`${a.id} ${levelStr(a)}`);
   }}]});

/* ═══ محور ═══ */
defTool({
 id:"axis", alias:"ax محور", label:"محور",
 hint:"انقر موضع المحور · Enter ينهي",
 opts:[
  {k:"dir",label:"الاتجاه",type:"sel",
   items:[["x","رأسي (حرف)"],["y","أفقي (رقم)"]],def:"x"}],
 steps:[
  {p:"موضع المحور (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    const d=ov("axis","dir");
    const v=addAxis(d,(d==="x")?p[0]:p[1]);
    dirty(ctx);
    const A=(d==="y")?S.grid.ys:S.grid.xs;
    H.rep("ok",`محور ${axLabel(d,A.indexOf(v))} على `
     +`${m3(v)} م · ${A.length} محوراً`);
   }}],
 prev(ctx,g){
  if(!g)return [];
  const d=ov("axis","dir"), L=9e5;
  return [(d==="x")
   ? pvLine([g[0],g[1]-L],[g[0],g[1]+L],YEL)
   : pvLine([g[0]-L,g[1]],[g[0]+L,g[1]],YEL)];
 }});
```

<a id="f-js-tools-areas-js"></a>

---

## `js/tools/areas.js`

```javascript
/* ═══ أدوات المناطق ═══
   الخبز أمرٌ يُنفَّذ مرّة، ونتيجته كائن مستقلّ — لا كشفٌ يُعاد
   في كلّ رسمة، ولا حدود وهمية تُخترع ليقتنع كاشف. */
import {S} from "../core/state.js";
import {sqm,m2} from "../core/units.js";
import {addArea,regionAt,areaAt,areaById,rebake,isStale,
        netArea,FILLS} from "../core/areas.js";
import {regionLoops,loopOpen,loopOpenAt} from "../core/render.js";
import {vis} from "../core/layers.js";
import {pArea,centroid} from "../core/geom.js";
import {defTool,H,rec,dirty,ov,ovOn,pvText} from "./registry.js";

const GRN="#5cd98e";
const visW=()=>vis("A-WALL")||vis("A-WALL-LOW");

defTool({
 id:"area", alias:"a منطقه غرفه", label:"منطقة",
 hint:"انقر داخل حلقة مغلقة · Enter ينهي",
 opts:[
  {k:"name", label:"الاسم",       type:"text",def:""},
  {k:"num",  label:"رقّم تلقائياً",type:"chk", def:1},
  {k:"showArea",label:"أظهر المساحة",type:"chk",def:1},
  {k:"fill", label:"التعبئة",     type:"sel",
   items:Object.keys(FILLS).map(k=>[k,FILLS[k]]), def:"tint"}],
 steps:[
  {p:"انقر داخل المنطقة (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    /* الهندسة لا تُخفى، لكن الخبز من جدرانٍ لا تراها يوقعك في
       حلقةٍ لا تفهم مصدرها — فالتنبيه لازم */
    if(!visW())
     H.rep("wr","طبقة الجدران مخفيّة — الخبز يقرأ الجدران كلّها "
      +"على أي حال. أظهرها لترى ما تخبزه.");
    const ex=areaAt(p[0],p[1]);
    if(ex)throw new Error(`توجد ${ex.id} هنا — احذفها أو حدّثها`);
    const ring=regionAt(regionLoops(),p[0],p[1]);
    if(!ring){
     /* السببُ ليس دائماً جداراً مفتوحاً: قد يكون الاتحادُ نفسه
        لم يُخَط. وبعد اللحم لا يقع ذلك لتفاوتٍ دون المليمترين،
        فإن وقع فالمدخلُ معطوبٌ فعلاً — والموضعُ يُقال، لأن
        «أغلق الجدران» بلا مكانٍ نصيحةٌ لا تُنفَّذ. */
     const g=loopOpen(), at=loopOpenAt();
     throw new Error(
      "لا حلقة مغلقة تحيط بهذه النقطة — أغلق الجدران أوّلاً "
      +"(المعيَّنات الحمراء تدلّ على الأطراف غير المتّصلة)"
      +(g?` · وفي اتحاد الأجسام ${g} قطعةً لم تُخَط`
        +(at?` — آخرُها عند (${m2(at[0])}، ${m2(at[1])})`:"")
        +". أزِح أحد الجدارَين هناك بخطوة الالتقاط ثم أعِد "
        +"المحاولة." : ""));
    }
    let nm=String(ov("area","name")||"").trim();
    if(ovOn("area","num")){
     const n=S.areas.length+1;
     nm=nm?`${nm} ${n}`:`منطقة ${n}`;
    }
    const a=addArea(ring,nm,{
     showArea:ovOn("area","showArea")?1:0,
     fill:ov("area","fill")});
    rec(ctx,a,"areas");
    H.rep("ok",`${a.id} ${a.name} · ${sqm(netArea(a))} م² · `
     +`${a.ring.length} ضلعاً · خُبزت كائناً مستقلّاً`);
   }}],
 prev(ctx,g){
  if(!g)return [];
  const ring=regionAt(regionLoops(),g[0],g[1]);
  if(!ring)return [];
  const o=[];
  for(let i=0;i<ring.length;i++)
   o.push({t:"l",a:ring[i],b:ring[(i+1)%ring.length],c:GRN});
  o.push(pvText(centroid(ring),
   `${sqm(Math.abs(pArea(ring)))} م² · ${ring.length} ضلعاً`,GRN));
  return o;
 }});

/* ═══ تحديث المناطق المحدَّدة ═══
   يعيد الخبز بأمرك، ويذكر فرق المساحة.
   destruct: يعيد خبز حلقاتٍ قائمة — فلا يُصدِره المزوّد بلا
   تصريحٍ من المستخدم. */
defTool({
 id:"arearef", alias:"ar حدث تحديث", label:"تحديث المناطق",
 hint:"يعيد خبز المحدَّد من الهندسة الحالية",
 destruct:1,
 opts:[],
 start(ctx){
  const L=H.sel().filter(s=>s.k==="area");
  const T=L.length?L.map(s=>areaById(s.id)).filter(Boolean)
   :S.areas.filter(isStale);
  if(!T.length){
   H.rep("in",L.length?"لا منطقة محدَّدة"
    :"لا منطقة قديمة تحتاج تحديثاً");
   return false;
  }
  const loops=regionLoops();
  let n=0, fail=0;
  T.forEach(a=>{
   try{
    const r=rebake(a,loops);
    n++;
    const d=r.after-r.before;
    H.rep("ok",`${a.id} ${a.name}: ${sqm(r.before)} → `
     +`${sqm(r.after)} م²`
     +(Math.abs(d)>1?` (${d>0?"+":""}${sqm(d)})`:" (بلا تغيّر)"));
   }catch(e){fail++; H.rep("er",e.message)}
  });
  if(n)dirty(ctx);
  H.rep(fail?"wr":"ok",`حُدّثت ${n} منطقة`
   +(fail?` · تعذّرت ${fail}`:"")
   +(L.length?"":" (كانت قديمة)"));
  return false;                    /* أمر لحظي — لا خطوات */
 },
 steps:[]});
```

<a id="f-js-tools-blocks-js"></a>

---

## `js/tools/blocks.js`

```javascript
/* ═══ أداة إدراج العناصر ═══ */
import { makeInstance } from "../core/blocks.js";
import { edit } from "../core/state.js";
import { H } from "./registry.js";

let hooks = null;
let active = null;

export function initBlockTool(h) { hooks = h || null; }
export const isInserting = () => !!active;
export const ghost = () => active;
const snap = w => (hooks && hooks.snap && hooks.snap(w)) || w;
const defaultLayer = () => typeof hooks?.defaultLayer === "function"
  ? hooks.defaultLayer() : (hooks?.defaultLayer || "0");

export function startInsert(name, opts = {}) {
  active = {
    block: name, x: 0, y: 0,
    rot: Number.isFinite(+opts.rot) ? +opts.rot : 0,
    scale: Number.isFinite(+opts.scale) && +opts.scale > 0 ? +opts.scale : 1,
    mirror: !!opts.mirror, layer: opts.layer || defaultLayer()
  };
  H.rep?.("in", "إدراج: انقر للموضع · R تدوير · M مرآة · Esc إلغاء");
  hooks?.redraw?.();
  return active;
}

export function onMove(world) {
  if (!active) return;
  const p = snap(world);
  active.x = p[0] ?? p.x ?? 0;
  active.y = p[1] ?? p.y ?? 0;
  hooks?.redraw?.();
}

export function onClick(world) {
  if (!active) return null;
  const p = snap(world);
  const x = p[0] ?? p.x ?? 0, y = p[1] ?? p.y ?? 0;
  const inst = makeInstance(active.block, {
    x, y, rot: active.rot, scale: active.scale,
    mirror: active.mirror, layer: active.layer
  });
  edit(() => hooks?.addInstance?.(inst), "إدراج " + active.block);
  H.rep?.("in", "أُدرج عنصر");
  hooks?.redraw?.();
  return inst;
}

export function onKey(e) {
  if (!active) return false;
  if (e.key === "Escape") { cancel(); return true; }
  if (e.key === "r" || e.key === "R") {
    active.rot += (e.shiftKey ? -1 : 1) * Math.PI / 2;
    hooks?.redraw?.(); return true;
  }
  if (e.key === "m" || e.key === "M") {
    active.mirror = !active.mirror;
    hooks?.redraw?.(); return true;
  }
  if (e.key === "+" || e.key === "=") {
    active.scale *= 1.1; hooks?.redraw?.(); return true;
  }
  if (e.key === "-" || e.key === "_") {
    active.scale /= 1.1; hooks?.redraw?.(); return true;
  }
  return false;
}

export function cancel() {
  active = null;
  H.rep?.("in", "أُلغي الإدراج");
  hooks?.redraw?.();
}
```

<a id="f-js-tools-boq-js"></a>

---

## `js/tools/boq.js`

```javascript
/* ═══ أمر جدول الكميات ═══
   أداةٌ فوريّة بلا خطوات — كarearef: منطقُها في start وتعود
   false، فلا تدخل وضعَ التقاط نقاط.

   ولا تعدّل شيئاً: لا rec ولا dirty، ولا edit في core/state.
   فالقراءةُ لا تدخل التاريخ، وضغطُ التراجع بعدها يتراجع عمّا
   قبلها — وهو الصواب. */
import {S} from "../core/state.js";
import {boq,boqLine} from "../core/boq.js";
import {toCSV,csvName} from "../io/boq.js";
import {dl} from "../io/project.js";
import {sqm,m2} from "../core/units.js";
import {price} from "../core/pricing.js";
import {boqToCSV} from "../io/boqcsv.js";
import {defTool,H,ovOn} from "./registry.js";

defTool({
 id:"boq", alias:"كميات جدولكميات",
 label:"جدول الكميات",
 hint:"يقرأ الحالة الحالية ويصدّر CSV",
 opts:[
  {k:"save", label:"نزّل الملفّ", type:"chk", def:1},
  {k:"price",label:"أضف التسعير والضريبة",type:"chk",def:1}],
 start(ctx){
  const B=boq();
  if(!B.walls.n&&!B.opens.total&&!B.areas.n){
   H.rep("in","المشروع فارغ — لا كميّات تُحصى");
   return false;
  }
  /* الحصيلةُ تُقال دائماً، والتنزيلُ خيار: من أراد النظرَ وحده
     أطفأ الخيار فقرأ الأسطر في اللوحة بلا ملفّ. */
  H.rep("ok",boqLine(B));
  B.areas.rows.forEach(r=>{
   H.rep(r.stale?"wr":"in",`${r.id} ${r.name}: `
    +`${sqm(r.area)} م²${r.stale?" · قديمة":""}`);
  });
  B.opens.rows.forEach(r=>{
   H.rep("in",`${r.name}: ${r.n}`);
  });
  B.walls.rows.forEach(r=>{
   H.rep("in",`${r.name}: ${m2(r.len)} م · `
    +`سماكة ${m2(r.t)} م${r.tn>1?` (${r.tn} سماكات)`:""}`);
  });
   const C=toCSV(B);
  C.notes.forEach(s=>H.rep("wr",s));
  if(ovOn("boq","save")){
   const nm=csvName(B);
    let text=C.txt, finalName=nm;
    if(ovOn("boq","price")){
     const items=[
      ...B.walls.rows.map(r=>({key:"wall",qty:r.face/1000000})),
      ...B.areas.rows.map(r=>({key:"area",qty:r.area/1000000})),
      ...B.opens.rows.map(r=>({key:/window|fixed/.test(r.kind)?"window":"door",qty:r.n}))
     ];
     const P=price(items);
     text=boqToCSV(P); finalName=nm.replace(/\.csv$/i,"-مسعّر.csv");
     H.rep("ok",`الإجمالي المسعّر: ${P.total.toFixed(2)} ${P.currency}`);
    }
    const size=dl(finalName,text,"text/csv;charset=utf-8");
    H.rep("ok",`نُزّل ${finalName} · ${size} بايت`);
  }
  return false;                    /* أمر لحظي — لا خطوات */
 },
 steps:[]});
```

<a id="f-js-tools-draw-js"></a>

---

## `js/tools/draw.js`

```javascript
/* ═══ أدوات الرسم ═══
   كل أداة تُنشئ ما طلبتَه بالحرف: جدار واحد بسماكته ومحاذاته،
   ولا لحم ولا كائن مشتقّ ولا تعديل على ما سبق. */
import {S} from "../core/state.js";
import {m2,m3,mm,dm2} from "../core/units.js";
import {addWall,ALIGN,dir,wallLen} from "../core/walls.js";
import {defTool,H,rec,finish,undoStep,ov,ovLen,ovOn,
        pvLine,pvRect,pvBand} from "./registry.js";

const GRN="#5cd98e", YEL="#ffd06b", PNK="#ff8f8f";
const TY=[["int","داخلي"],["ext","خارجي"],["low","سترة"]];
const AL=[["c","مركزي"],["l","الوجه الأيسر"],["r","الوجه الأيمن"]];

/* ═══ جدار ═══ */
function mkWall(ctx,a,b){
 const w=addWall(a,b,
  ovLen("wall","t"), ov("wall","type"), ov("wall","align"),
  ovLen("wall","h"));
 rec(ctx,w,"walls");
 const d=dir(w);
 H.rep("ok",`${w.id} · ${m2(wallLen(w))} م · `
  +`${d?d.ang.toFixed(1):"0"}° · ${mm(w.t).toFixed(2)} م `
  +`${ALIGN[w.align]}`
  +(w.type==="low"?` · سترة ${m2(w.h)} م`:""));
 return w;
}
defTool({
 id:"wall", alias:"w جدار خط", label:"جدار",
 hint:"نقطتان لكل جدار · C يغلق · U يتراجع خطوة",
 opts:[
  {k:"t",    label:"السماكة م", type:"len", def:"0.15"},
  {k:"type", label:"النوع",     type:"sel", items:TY, def:"int"},
  {k:"align",label:"المسار على",type:"sel", items:AL, def:"c",
   hint:"يسار ويمين بالنسبة لاتجاه الرسم"},
  {k:"h",    label:"ارتفاع السترة م", type:"len", def:"1",
   when:o=>o.type==="low"},
  {k:"chain",label:"متّصل",     type:"chk", def:1}],
 steps:[
  {p:"نقطة البداية"},
  {p:"النقطة التالية", loop:1, base:-1,
   opts:{
    c:{n:"إغلاق",run(ctx){
     if(ctx.pts.length<3)throw new Error("الإغلاق يحتاج ثلاث نقاط");
     mkWall(ctx,ctx.pts[ctx.pts.length-1],ctx.pts[0]);
     finish("أُغلق المضلع");
    }},
    u:{n:"تراجع",run(){undoStep()}}},
   each(ctx,p){
    const P=ctx.pts;
    if(P.length<2)return;
    mkWall(ctx,P[P.length-2],p);
    /* غير المتّصل: كل جدار مستقلّ بنقطتيه */
    if(!ovOn("wall","chain")){
     ctx.pts.length=0;
     ctx.pts.push(p);
    }
   }}],
 prev(ctx,g){
  const o=[], P=ctx.pts, t=ovLen("wall","t")||150;
  for(let i=0;i<P.length-1;i++)o.push(pvLine(P[i],P[i+1],"#4b5a6b"));
  if(P.length&&g){
   o.push(pvBand(P[P.length-1],g,t,GRN));
   o.push(pvLine(P[P.length-1],g,YEL));
  }
  return o;
 }});

/* ═══ مستطيل ═══
   المحاذاة تُطبَّق على كل ضلع بحيث تنتظم كلها إلى الداخل أو الخارج.
   الاتجاه يُوحَّد عكس عقارب الساعة، فيكون «اليسار» هو الخارج. */
defTool({
 id:"rect", alias:"r مستطيل", label:"مستطيل",
 hint:"ركنان متقابلان · أو اكتب مقاساً مثل 9x14",
 opts:[
  {k:"t",    label:"السماكة م", type:"len", def:"0.25"},
  {k:"type", label:"النوع",     type:"sel", items:TY, def:"ext"},
  {k:"align",label:"القياس",    type:"sel",
   items:[["c","محوري"],["l","داخلي صافٍ"],["r","خارجي كلّي"]],
   def:"c"}],
 steps:[
  {p:"الركن الأول"},
  {p:"الركن المقابل أو المقاس", base:0, restart:1,
   each(ctx,p){
    const a=ctx.pts[0];
    const x0=Math.min(a[0],p[0]), x1=Math.max(a[0],p[0]);
    const y0=Math.min(a[1],p[1]), y1=Math.max(a[1],p[1]);
    if(x1-x0<500||y1-y0<500)
     throw new Error("المستطيل أصغر من 0.50 م");
    const t=ovLen("rect","t"), ty=ov("rect","type");
    const al=ov("rect","align");
    /* عكس الساعة مع Y للأعلى: الداخل يسار كل ضلعٍ موجَّه.
       فalign=l يعني المسار على الوجه الداخلي والجسم يمتدّ خارجاً
       ⇒ المقاس المرسوم هو الصافي. وr عكسه: المقاس كلّيّ. */
    const Q=[[x0,y0],[x1,y0],[x1,y1],[x0,y1]];
    for(let i=0;i<4;i++)rec(ctx,
     addWall(Q[i],Q[(i+1)%4],t,ty,al),"walls");
    const nm={c:"محورياً",l:"صافياً",r:"كلّياً"}[al];
    H.rep("ok",`4 جدران · ${dm2(x1-x0,y1-y0,"م")} ${nm}`);
   }}],
 prev(ctx,g){
  return (ctx.pts.length&&g)?[pvRect(ctx.pts[0],g,GRN)]:[];
 }});

/* ═══ قياس — لا يُنشئ شيئاً ═══ */
defTool({
 id:"measure", alias:"mi قياس مسافه", label:"قياس",
 hint:"نقاط متتالية · Enter ينهي ويعرض النتيجة",
 opts:[],
 steps:[
  {p:"النقطة الأولى"},
  {p:"النقطة التالية (Enter ينهي)", loop:1, base:-1}],
 prev(ctx,g){
  const P=ctx.pts.concat(g?[g]:[]), o=[];
  for(let i=0;i<P.length-1;i++)o.push(pvLine(P[i],P[i+1],PNK));
  return o;
 },
 done(ctx){
  const P=ctx.pts;
  if(P.length<2)return;
  let tot=0;
  for(let i=0;i<P.length-1;i++)
   tot+=Math.hypot(P[i+1][0]-P[i][0],P[i+1][1]-P[i][1]);
  let s=`المسافة ${m3(tot)} م`;
  if(P.length===2){
   const a=((Math.atan2(P[1][1]-P[0][1],P[1][0]-P[0][0])
    *180/Math.PI)%360+360)%360;
   s+=` · الزاوية ${a.toFixed(1)}°`;
  }
  if(P.length>3){
   let ar=0;
   for(let i=0,n=P.length;i<n;i++){
    const q=P[i], r=P[(i+1)%n];
    ar+=q[0]*r[1]-r[0]*q[1];
   }
   s+=` · المساحة ${(Math.abs(ar/2)/1e6).toFixed(3)} م²`;
  }
  H.rep("ok",s);
 }});
```

<a id="f-js-tools-elev-js"></a>

---

## `js/tools/elev.js`

```javascript
/* ═══ أمر الواجهة ═══
   أداةٌ فوريّة بلا خطوات — كboq: منطقُها في start وتعود false، فلا
   تدخل وضعَ التقاط نقاط.

   ولا تعدّل شيئاً: لا rec ولا dirty ولا edit. القراءةُ لا تدخل
   التاريخ، وضغطُ التراجع بعدها يتراجع عمّا قبلها.

   arg:1 يُعلن أنها تقبل وسيطاً من سطر الإدخال — «ELEV S» و
   «ELEV 315». */
import {S} from "../core/state.js";
import {elevCmd,elevSay} from "../core/elevation.js";
import {elevFile} from "../io/elev.js";
import {dl} from "../io/project.js";
import {m2} from "../core/units.js";
import {defTool,H,ov,ovLen,ovNum,ovOn} from "./registry.js";

/* الوسيط يسبق الخيار: ما كتبتَه في السطر أصدقُ من لوحةٍ لزجة */
export function viewArg(arg){
 if(arg!=null&&String(arg).trim()!=="")return arg;
 const v=ov("elev","view");
 return (v==="ang")?ovNum("elev","ang"):(v||"S");
}
export function runElev(arg){
 const gap=Math.max(0,ovLen("elev","gap")||0);
 const e=elevCmd(viewArg(arg),{gap});     /* يرمي إن لا جدار يواجه */

 H.rep("ok",elevSay(e));
 e.runs.forEach(r=>{
  H.rep("in",`${r.id}: ${m2(r.x0)} → ${m2(r.x1)} م · `
   +`${r.opens} فتحة${r.flip?" · معكوس":""}`
   +(r.sure?"":" · جهةُ خارجه غير محسومة"));
 });
 e.warn.forEach(w=>H.rep("wr",w.msg));

 const fmt=String(ov("elev","fmt")||"none");
 if(fmt==="none")return e;
 const F=elevFile(fmt,e);
 F.notes.forEach(s=>H.rep("wr",s));
 if(F.bad)H.rep("wr",`${F.bad} محرفاً تعذّر ترميزه`);
 if(ovOn("elev","save")){
  const size=dl(F.name,(F.txt!=null)?F.txt:F.bytes,F.mime);
  H.rep("ok",`نُزّل ${F.name}`+(size?` · ${size} بايت`:""));
 }else{
  H.rep("in",`${F.name} جاهز — شغّل «نزّل الملفّ» ليُكتَب`);
 }
 return e;
}
defTool({
 id:"elev", alias:"واجهة الواجهة elevation",
 label:"واجهة", arg:1,
 hint:"يفرد الجدران الخارجية المواجهة · ELEV S أو ELEV 315 — "
  +"ولا تتحدّث تلقائياً بعدها",
 opts:[
  {k:"view", label:"الاتجاه", type:"sel", def:"S", items:[
   ["S","جنوبية"],["N","شمالية"],["E","شرقية"],["W","غربية"],
   ["ang","زاوية حرّة"]]},
  {k:"ang",  label:"الزاوية (°)", type:"num", def:"0"},
  {k:"gap",  label:"فاصل بين الفرود", type:"len", def:"0"},
  {k:"fmt",  label:"التصدير", type:"sel", def:"none", items:[
   ["none","لا شيء"],["svg","SVG"],["dxf","DXF"],["pdf","PDF"]]},
  {k:"save", label:"نزّل الملفّ", type:"chk", def:1}],
 start(ctx){
  if(!S.walls.some(w=>w.type==="ext")){
   H.rep("in",'لا جدار خارجيّ — الواجهة تُبنى من type="ext" وحدها');
   return false;
  }
  runElev(ctx.arg);        /* الرمي يبلغ H.rep عبر مصيدة begin */
  return false;            /* أمرٌ لحظيّ — لا خطوات */
 },
 steps:[]});
```

<a id="f-js-tools-modify-js"></a>

---

## `js/tools/modify.js`

```javascript
/* ═══ أدوات التعديل ═══
   كلّها تعمل على تحديد قائم أو على عناصر تنقرها صراحةً.
   لا حدود ضمنية ولا «كل الجدران» — تحدّد الحدّ بيدك.
   الفتحات تتبع النسخ افتراضاً، ويمنعها خيار في الشريط،
   والسجل يذكر عددها دائماً.

   النقل والنسخ والدوران والمرآة تعمل على الأنواع كلّها.
   والإزاحة والقطع والقصّ والتمديد والشدّ واللحم للجدران وحدها. */
import {S,touch} from "../core/state.js";
import {m2,m3,mnum,clamp,norm,pt2,arrow,dim2} from "../core/units.js";
import {angOf} from "../core/coords.js";
import {wallById,wallLen,dir,MINW} from "../core/walls.js";
import {NAME,COLL,findById,pickEnts,delSay} from "../core/ents.js";
import {pickable} from "../core/layers.js";
import {FLD,readField,applyField,fldName} from "../core/batch.js";
import {grab,segsOf,moveAll,copyAll,rotP,rotateAll,mirrorAll,
        offsetWall,breakWall,trimWall,extendWall,
        stretchGrab,stretchApply,stretchPrev,
        arrayRect,arrayPolar,chamferPlan,chamferApply,
        weldPlan,weldApply,WHY} from "../core/modify.js";
import {defTool,H,T,rec,dirty,finish,pushSteps,nextStep,
        ov,ovLen,ovNum,ovOn,pvLine,pvRect,pvBand} from "./registry.js";

const GRN="#5cd98e", YEL="#ffd06b", RED="#ff6f6f";
const OPTS_OPEN={k:"opens",label:"انسخ الفتحات",type:"chk",def:1};

/* التحديد المطلوب — يُقرأ مرّة عند بدء الأداة */
function needSel(ctx,msg,wallOnly){
 let L=H.sel();
 if(wallOnly)L=L.filter(s=>s.k==="wall");
 if(!L.length){
  H.rep("wr",msg||(wallOnly
   ? "حدّد جدراناً أولاً ثم نفّذ الأداة"
   : "حدّد عناصر أولاً ثم نفّذ الأداة"));
  return false;
 }
 ctx.v.list=L;
 ctx.v.G=grab(L);
 if(!ctx.v.G.length){
  H.rep("wr","لا عنصر قابل للتحويل — المخفيّ والمقفل خارج");
  return false;
 }
 return true;
}
const sayR=(r,verb)=>{
 const P=[`${r.walls} عنصر`];
 if(r.opens)P.push(`${r.opens} فتحة`);
 H.rep("ok",`${verb} ${P.join(" و ")}`);
 (r.refused||[]).slice(0,6).forEach(x=>H.rep("wr",
  `  رُفض ${x} — التحويل يفسد قياسه`));
 if((r.refused||[]).length>6)
  H.rep("in",`  … و ${r.refused.length-6} رفضاً آخر`);
};
/* ═══ نقل ═══ */
/* هادمة: تحرّك ما هو مرسوم سلفاً */
defTool({
 id:"move", alias:"m انقل", label:"نقل",
 destruct:1,
 hint:"نقطة أساس ثم وجهة — المحدّد وحده يتحرّك",
 opts:[],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"نقطة الأساس"},
  {p:"نقطة الوجهة أو الإزاحة", base:0,
   each(ctx,p){
    const a=ctx.pts[0];
    const n=moveAll(ctx.v.G,p[0]-a[0],p[1]-a[1]);
    dirty(ctx);
    H.rep("ok",`نُقل ${n} عنصر ${pt2([p[0]-a[0],p[1]-a[1]])} م`
     +` · الفتحات تبعت جدرانها`);
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  const a=ctx.pts[0], dx=g[0]-a[0], dy=g[1]-a[1];
  const o=[pvLine(a,g,YEL)];
  segsOf(ctx.v.G).forEach(s=>o.push(
   pvLine([s[0][0]+dx,s[0][1]+dy],[s[1][0]+dx,s[1][1]+dy],GRN)));
  return o;
 }});

/* ═══ نسخ ═══ */
/* ليست هادمة: تُنشئ نسخاً ولا تمسّ الأصل */
defTool({
 id:"copy", alias:"cp انسخ", label:"نسخ",
 hint:"نقطة أساس ثم نقطة لكل نسخة · Enter ينهي",
 opts:[
  {k:"n",label:"عدد النسخ",type:"num",def:1,hint:"لكل نقرة"},
  OPTS_OPEN],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"نقطة الأساس"},
  {p:"نقطة النسخة (Enter ينهي)", loop:1, base:0,
   each(ctx,p){
    const a=ctx.pts[0];
    const r=copyAll(ctx.v.G,p[0]-a[0],p[1]-a[1],
     Math.round(ovNum("copy","n"))||1, ovOn("copy","opens"));
    /* النسخ كائنات جديدة — تُسجَّل ليتراجع عنها Esc أو U */
    (r.made||[]).forEach(s=>rec(ctx,{id:s.id},COLL[s.k]));
    sayR(r,"نُسخ");
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  const a=ctx.pts[0], dx=g[0]-a[0], dy=g[1]-a[1];
  const n=clamp(Math.round(ovNum("copy","n"))||1,1,20);
  const o=[pvLine(a,g,YEL)];
  for(let i=1;i<=n;i++)
   segsOf(ctx.v.G).forEach(s=>o.push(pvLine(
    [s[0][0]+dx*i,s[0][1]+dy*i],[s[1][0]+dx*i,s[1][1]+dy*i],GRN)));
  return o;
 }});

/* ═══ دوران ═══ */
function applyRot(ctx){
 const c=ctx.pts[0];
 const d=(ctx.v.a1||0)-(ctx.v.a0||0);
 const r=rotateAll(ctx.v.G,c,d,ovOn("rotate","copy"),
  ovOn("rotate","opens"));
 if(ovOn("rotate","copy"))
  (r.made||[]).forEach(s=>rec(ctx,{id:s.id},COLL[s.k]));
 else dirty(ctx);
 H.rep("ok",`دُوِّر ${r.walls} عنصر ${d.toFixed(1)}°`
  +(r.opens?` · ${r.opens} فتحة`:"")
  +(ovOn("rotate","copy")?" (نسخة)":""));
 (r.refused||[]).forEach(x=>H.rep("wr",
  `  رُفض ${x} — البُعد الأفقي أو الرأسي لا يدور إلا بمضاعفات 90°`));
}
defTool({
 id:"rotate", alias:"ro دور تدوير", label:"دوران",
 destruct:1,
 hint:"نقطة الدوران ثم الزاوية · R للزاوية المرجعية",
 opts:[
  {k:"copy",label:"نسخة",type:"chk",def:0},
  OPTS_OPEN],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"نقطة الدوران"},
  {p:"الزاوية أو انقر الاتجاه", k:"a1", ang:1,
   opts:{r:{n:"مرجع",run(){
    pushSteps([
     {p:"الزاوية المرجعية",k:"a0",ang:1},
     {p:"الزاوية الجديدة",k:"a1",ang:1,each(c){applyRot(c)}}]);
    nextStep();
   }}},
   each(ctx){applyRot(ctx)}}],
 prev(ctx,g){
  if(!ctx.v.G||!ctx.pts.length||!g)return [];
  const c=ctx.pts[0];
  const a=angOf(c,g)-(ctx.v.a0||0);
  const o=[pvLine(c,g,YEL)];
  segsOf(ctx.v.G).forEach(s=>o.push(
   pvLine(rotP(s[0],c,a),rotP(s[1],c,a),GRN)));
  return o;
 }});

/* ═══ مرآة ═══ */
/* هادمة: «أبقِ الأصل» خيارٌ قد يُطفأ */
defTool({
 id:"mirror", alias:"mr مراه اعكس", label:"مرآة",
 destruct:1,
 hint:"نقطتان على محور المرآة · جهة فتح الأبواب تُقلَب",
 opts:[
  {k:"keep",label:"أبقِ الأصل",type:"chk",def:1},
  OPTS_OPEN],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"أول نقطة على محور المرآة"},
  {p:"ثاني نقطة على المحور", base:0,
   each(ctx,p){
    const keep=ovOn("mirror","keep");
    const r=mirrorAll(ctx.v.G,ctx.pts[0],p,keep,
     ovOn("mirror","opens"));
    if(keep)(r.made||[]).forEach(s=>rec(ctx,{id:s.id},COLL[s.k]));
    else dirty(ctx);
    H.rep("ok",`انعكس ${r.walls} عنصر`
     +(r.opens?` و ${r.opens} فتحة`:"")
     +(keep?" · بقي الأصل":""));
    (r.refused||[]).forEach(x=>H.rep("wr",
     `  رُفض ${x} — يحتاج محوراً قائماً أو قطرياً`));
   }}],
 prev(ctx,g){
  const P=ctx.pts;
  if(P.length===1&&g)return [pvLine(P[0],g,YEL)];
  if(P.length<2)return [];
  const a=P[0], b=P[1];
  const dx=b[0]-a[0], dy=b[1]-a[1], L=Math.hypot(dx,dy);
  const o=[pvLine(a,b,YEL)];
  if(L<1)return o;
  const ux=dx/L, uy=dy/L;
  const M=p=>{
   const px=p[0]-a[0], py=p[1]-a[1], t=px*ux+py*uy;
   return [Math.round(a[0]+2*ux*t-px),Math.round(a[1]+2*uy*t-py)];
  };
  segsOf(ctx.v.G).forEach(s=>o.push(pvLine(M(s[0]),M(s[1]),GRN)));
  return o;
 }});

/* ═══ إزاحة ═══ */
/* ليست هادمة: تُنشئ موازياً ولا تمسّ الأصل */
defTool({
 id:"offset", alias:"of ازح موازي", label:"إزاحة",
 hint:"اختر جداراً ثم انقر الجهة · Enter ينهي",
 opts:[
  {k:"d",    label:"المسافة م",type:"len",def:"1"},
  {k:"clear",label:"صافية",    type:"chk",def:0,
   hint:"بين الوجهَين لا المحورين"},
  {k:"t",    label:"سماكة الجديد م",type:"len",def:"",
   hint:"فارغ = مثل الأصل"},
  OPTS_OPEN],
 steps:[
  {p:"اختر الجدار", ent:"wall", entName:"جدار", k:"w"},
  {p:"انقر الجهة (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    const w=wallById(ctx.v.w.id);
    if(!w)throw new Error("الجدار غير موجود");
    const u=dir(w);
    if(!u)throw new Error("الجدار صفري");
    const sg=(u.nx*(p[0]-w.a[0])+u.ny*(p[1]-w.a[1]))>0?1:-1;
    const t=ovLen("offset","t");
    const r=offsetWall(w.id, ovLen("offset","d"), sg,
     ovOn("offset","clear"), t||null, null,
     ovOn("offset","opens"));
    rec(ctx,r.wall,"walls");
    /* السلسلة: الجديد يصير أصلاً للإزاحة التالية */
    ctx.v.w={k:"wall",id:r.wall.id};
    H.rep("ok",`${r.wall.id} موازٍ ${m2(r.d)} م `
     +`${ovOn("offset","clear")?"صافياً":"محورياً"}`
     +(r.opens?` · ${r.opens} فتحة`:"")
     +` · الأطراف لم تُلحَم — استعمل «لحم» إن أردت`);
   }}],
 prev(ctx,g){
  const s=ctx.v.w;
  if(!s||!g)return [];
  const w=wallById(s.id);
  if(!w)return [];
  const u=dir(w);
  if(!u)return [];
  const sg=(u.nx*(g[0]-w.a[0])+u.ny*(g[1]-w.a[1]))>0?1:-1;
  const t2=ovLen("offset","t")||w.t;
  const D=ovLen("offset","d")
   +(ovOn("offset","clear")?(w.t+t2)/2:0);
  const px=u.nx*sg*D, py=u.ny*sg*D;
  return [pvBand([w.a[0]+px,w.a[1]+py],[w.b[0]+px,w.b[1]+py],
   t2, GRN)];
 }});

/* ═══ قطع ═══ */
defTool({
 id:"break", alias:"br اقطع", label:"قطع",
 destruct:1,
 hint:"جدار ثم نقطة القطع · الفتحة العابرة تُحذَف ويُذكر عددها",
 opts:[],
 steps:[
  {p:"اختر الجدار", ent:"wall", entName:"جدار", k:"w"},
  {p:"نقطة القطع", base:"none", restart:1,
   each(ctx,p){
    const r=breakWall(ctx.v.w.id,p);
    rec(ctx,r.nw,"walls");
    H.rep(r.lost?"wr":"ok",
     `قُطع إلى ${m2(r.a)} + ${m2(r.b)} م → ${r.nw.id}`
     +(r.moved?` · ${r.moved} فتحة انتقلت`:"")
     +(r.lost?` · حُذفت ${r.lost} فتحة تعبر نقطة القطع`:""));
   }}],
 prev(ctx,g){
  const s=ctx.v.w;
  if(!s||!g)return [];
  const w=wallById(s.id);
  if(!w)return [];
  const u=dir(w);
  if(!u)return [];
  const t=clamp((g[0]-w.a[0])*u.ux+(g[1]-w.a[1])*u.uy,0,u.L);
  const q=[w.a[0]+u.ux*t, w.a[1]+u.uy*t];
  const h=Math.max(w.t,300);
  return [pvLine([q[0]+u.nx*h,q[1]+u.ny*h],
                 [q[0]-u.nx*h,q[1]-u.ny*h],RED)];
 }});

/* ═══ قصّ ═══ */
defTool({
 id:"trim", alias:"tr قص", label:"قصّ",
 destruct:1,
 hint:"حدّد الحدود بالنقر (Enter ينهي) ثم انقر الجزء المُزال",
 opts:[],
 steps:[
  {p:"انقر حدّاً (Enter ينهي التحديد)",
   ent:"wall", entName:"جدار", loop:1, min:1,
   each(ctx,hit){
    (ctx.v.cut=ctx.v.cut||[]).push(hit.id);
    H.rep("in",`${hit.id} حدّ قصّ · المجموع ${ctx.v.cut.length}`);
   }},
  {p:"انقر الجزء المراد إزالته",
   ent:"wall", entName:"جدار", loop:1,
   each(ctx,hit,p){
    const r=trimWall(hit.id,ctx.v.cut,p);
    dirty(ctx);
    const AR={start:"من البداية",end:"من النهاية",
     mid:"وسطاً — صار جدارين"};
    if(r.nw)rec(ctx,r.nw,"walls");
    H.rep(r.lost?"wr":"ok",
     `قُصّ ${hit.id} ${AR[r.mode]} · ${m2(r.cut)} م`
     +(r.lost?` · حُذفت ${r.lost} فتحة في المقطوع`:""));
   }}]});

/* ═══ تمديد ═══ */
defTool({
 id:"extend", alias:"ex مدد وسع", label:"تمديد",
 destruct:1,
 hint:"حدّد الحدود بالنقر (Enter ينهي) ثم انقر الطرف المُمَدّ",
 opts:[],
 steps:[
  {p:"انقر حدّاً (Enter ينهي التحديد)",
   ent:"wall", entName:"جدار", loop:1, min:1,
   each(ctx,hit){
    (ctx.v.bnd=ctx.v.bnd||[]).push(hit.id);
    H.rep("in",`${hit.id} حدّ · المجموع ${ctx.v.bnd.length}`);
   }},
  {p:"انقر الطرف المراد تمديده",
   ent:"wall", entName:"جدار", loop:1,
   each(ctx,hit,p){
    const r=extendWall(hit.id,ctx.v.bnd,p);
    dirty(ctx);
    H.rep("ok",`مُدّد ${hit.id} `
     +`${r.mode==="end"?"من النهاية":"من البداية"} · `
     +`+${m2(r.add)} م`);
   }}]});

/* ═══ شدّ ═══ */
defTool({
 id:"stretch", alias:"str شد", label:"شدّ",
 destruct:1,
 hint:"إطار يحوي الأطراف · ثم أساس ووجهة",
 opts:[],
 steps:[
  {p:"الزاوية الأولى لإطار الشدّ"},
  {p:"الزاوية المقابلة", base:0,
   each(ctx,p){
    const a=ctx.pts[0];
    const r={x0:Math.min(a[0],p[0]),y0:Math.min(a[1],p[1]),
             x1:Math.max(a[0],p[0]),y1:Math.max(a[1],p[1])};
    ctx.v.G=stretchGrab(r);
    if(!ctx.v.G.length)throw new Error("لا أطراف داخل الإطار");
    const both=ctx.v.G.filter(g=>g.a&&g.b).length;
    H.rep("in",`${ctx.v.G.length} جدار متأثّر`
     +(both?` · ${both} منها بطرفَيه (سينتقل كاملاً)`:""));
   }},
  {p:"نقطة الأساس", base:"none"},
  {p:"نقطة الوجهة أو الإزاحة", base:2,
   each(ctx,p){
    const b=ctx.pts[2];
    const n=stretchApply(ctx.v.G,p[0]-b[0],p[1]-b[1]);
    dirty(ctx);
    H.rep("ok",`شُدّ ${n} جدار ${pt2([p[0]-b[0],p[1]-b[1]])} م`
     +` · الفتحات لم تُمَسّ`);
   }}],
 prev(ctx,g){
  const P=ctx.pts;
  if(!g)return [];
  if(P.length===1)return [pvRect(P[0],g,GRN)];
  if(P.length===3&&ctx.v.G){
   const o=stretchPrev(ctx.v.G,g[0]-P[2][0],g[1]-P[2][1])
    .map(s=>pvLine(s[0],s[1],GRN));
   o.push(pvLine(P[2],g,YEL));
   return o;
  }
  return [];
 }});

/* ═══ لحم — معاينة ثم تأكيد ═══
   العقد يقول: لا يتحرّك إحداثيٌّ إلا بأمرك. فالخطة تُعرَض بالمليمتر
   قبل التنفيذ، والتنفيذ لا يتجاوزها. */
defTool({
 id:"weld", alias:"wl لحم", label:"لحم",
 destruct:1,
 hint:"يعرض ما سيتحرّك ثم ينتظر تأكيدك",
 opts:[{k:"tol",label:"التفاوت م",type:"len",def:"0.03"}],
 start(ctx){
  const L=H.sel().filter(s=>s.k==="wall");
  if(!L.length){
   H.rep("wr","حدّد جدرانَ اللحم أولاً — اللحم لا يعمل على الكل");
   return false;
  }
  const plan=weldPlan(L.map(s=>s.id), ovLen("weld","tol"));
  if(!plan.moves.length){
   H.rep("in",`لا طرف يحتاج لحماً عند تفاوت `
    +`${m2(plan.tol)} م · جرّب تفاوتاً أوسع`);
   return false;
  }
  ctx.v.plan=plan;
  H.rep("wr",`${plan.moves.length} طرف سيتحرّك — راجع القائمة `
   +`ثم Enter للتأكيد أو Esc للإلغاء:`);
  plan.moves.slice(0,24).forEach(m=>H.rep("in",
   `  ${m.id}/${m.end}  ${m2(m.d)} م  `
   +arrow(pt2(m.from), pt2(m.to))
   +`  ${WHY[m.why]||m.why}`));
  if(plan.moves.length>24)
   H.rep("in",`  … و ${plan.moves.length-24} طرفاً آخر`);
  return true;
 },
 steps:[
  {p:"Enter يؤكّد اللحم · Esc يلغي", confirm:1,
   each(ctx){
    const r=weldApply(ctx.v.plan);
    dirty(ctx);
    H.rep("ok",`لُحم ${r.moved} طرف`);
    if(r.short.length)
     H.rep("wr",`${r.short.length} جدار صار أقصر من الحدّ الأدنى `
      +`(${r.short.slice(0,6).join(" ")}) — لم يُحذَف، احذفه بيدك `
      +`إن شئت`);
   }}],
 prev(ctx){
  const P=ctx.v.plan;
  if(!P)return [];
  /* السهم من الموضع الحالي إلى المخطَّط — تراه قبل التنفيذ */
  return P.moves.map(m=>pvLine(m.from,m.to,RED));
 }});

/* ═══ كسرُ الركن — معاينةٌ ثم تأكيد ═══
   كاللحم: الخطّة تُعرَض بالمليمتر قبل التنفيذ، والتنفيذُ لا
   يتجاوزها. وانقر كلَّ جدارٍ في الجهة التي تريد إبقاءها. */
defTool({
 id:"chamfer", alias:"chm شطف كسر_الركن", label:"كسر الركن",
 destruct:1,
 hint:"انقر الجدارين في جهتهما المُبقاة — تُعرَض الخطّة ثم تنتظر",
 opts:[
  {k:"d", label:"المسافة م",       type:"len", def:"0.5"},
  {k:"d2",label:"مسافة الثاني م",  type:"len", def:"",
   hint:"فارغ = مثل الأولى"}],
 steps:[
  {p:"انقر الجدار الأول في جهته المُبقاة",
   ent:"wall", entName:"جدار", k:"w1",
   each(ctx,hit,p){ctx.v.p1=p}},
  {p:"انقر الجدار الثاني في جهته المُبقاة",
   ent:"wall", entName:"جدار", k:"w2",
   each(ctx,hit,p){
    const d=ovLen("chamfer","d");
    const pl=chamferPlan(ctx.v.w1.id,hit.id,d,
     ovLen("chamfer","d2")||d, ctx.v.p1, p);
    ctx.v.plan=pl;
    H.rep("wr","الخطّة — راجعها ثم Enter للتأكيد أو Esc للإلغاء:");
    [pl.a,pl.b].forEach(x=>H.rep("in",
     `  ${x.id}/${x.end}  ${m3(x.d)} م من الركن  `
     +arrow(pt2(x.from),pt2(x.to))
     +(x.grow>0.5?`  (يُمَدّ ${m3(x.grow)} م ليبلغ الركن)`:"")
     +`  يبقى ${m3(x.remain)} م`));
    H.rep("in",`  الضلعُ الجديد ${m3(pl.len)} م · الزاوية `
     +`${pl.ang}° · سماكتُه ${m3(pl.t)} م مركزيةً`);
   }},
  {p:"Enter يؤكّد كسرَ الركن · Esc يلغي", confirm:1,
   each(ctx){
    const r=chamferApply(ctx.v.plan);
    dirty(ctx);
    rec(ctx,r.wall,"walls");
    H.rep(r.lost?"wr":"ok",
     `${r.wall.id} ضلعُ الكسر ${m3(r.len)} م`
     +(r.lost?` · حُذفت ${r.lost} فتحة في المقطوع`:"")
     +` · الأطرافُ لم تُلحَم — استعمل «لحم» إن أردت`);
   }}],
 prev(ctx,g){
  const draw=q=>[pvLine(q.a.to,q.b.to,GRN),
   pvLine(q.a.from,q.a.to,RED), pvLine(q.b.from,q.b.to,RED)];
  if(ctx.v.plan)return draw(ctx.v.plan);
  /* قبل النقرة الثانية: الخطّةُ تُحسَب من الجدار تحت المؤشّر،
     فترى الركنَ قبل أن تنقره. والمتعذّرُ لا يُرسَم. */
  if(!ctx.v.w1||!g)return [];
  const h=H.hit(g[0],g[1]);
  if(!h||h.k!=="wall"||h.id===ctx.v.w1.id)return [];
  const d=ovLen("chamfer","d");
  return draw(chamferPlan(ctx.v.w1.id,h.id,d,
   ovLen("chamfer","d2")||d, ctx.v.p1, g));
 }});

/* ═══ المصفوفة المستطيلة ═══
   الأصلُ خليّةٌ في الشبكة، والتباعدُ من نقرتَيك لا من حقلٍ — فتراه
   قبل أن يقع. وتعمل على ما يعمل عليه «نسخ»: كلُّ الأنواع، والفتحةُ
   تتبع جدارها ولا تُنسَخ وحدها. */
defTool({
 id:"array", alias:"arr مصفوفه", label:"مصفوفة",
 hint:"أساسٌ ثم ركنُ الخليّة · الصفوفُ والأعمدةُ من الشريط",
 opts:[
  {k:"nx",label:"الأعمدة",type:"num",def:3},
  {k:"ny",label:"الصفوف", type:"num",def:1},
  OPTS_OPEN],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"نقطة الأساس"},
  {p:"ركن الخليّة (تباعد X و Y)", base:0,
   each(ctx,p){
    const a=ctx.pts[0];
    const r=arrayRect(ctx.v.G,
     Math.round(ovNum("array","nx")), Math.round(ovNum("array","ny")),
     p[0]-a[0], p[1]-a[1], ovOn("array","opens"));
    (r.made||[]).forEach(s=>rec(ctx,{id:s.id},COLL[s.k]));
    H.rep("ok",`${dim2(r.nx,r.ny)} · ${r.cells} خليّةً منسوخة · `
     +`${r.walls} عنصر`+(r.opens?` و ${r.opens} فتحة`:""));
    if(ctx.v.G.some(x=>x.s.k==="col"))
     H.rep("in","  وأوسامُ الأعمدة لا تُنسَخ — "
      +"استعمل «أعد الترقيم» على المحدَّد");
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  const a=ctx.pts[0], dx=g[0]-a[0], dy=g[1]-a[1];
  const NX=clamp(Math.round(ovNum("array","nx"))||1,1,100);
  const NY=clamp(Math.round(ovNum("array","ny"))||1,1,100);
  const S2=segsOf(ctx.v.G);
  const k=Math.max(120,Math.hypot(dx,dy)*0.05);
  const o=[pvLine(a,g,YEL)];
  let budget=700;                   /* المعاينةُ في كل إطار */
  for(let j=0;j<NY&&budget>0;j++)for(let i=0;i<NX&&budget>0;i++){
   if(!i&&!j)continue;
   const px=a[0]+dx*i, py=a[1]+dy*j;
   /* صليبٌ لكل خليّة: يُرى ولو كان المحدَّد بلا مسارات */
   o.push(pvLine([px-k,py],[px+k,py],YEL));
   o.push(pvLine([px,py-k],[px,py+k],YEL));
   budget-=2;
   S2.forEach(s=>{
    if(budget--<=0)return;
    o.push(pvLine([s[0][0]+dx*i,s[0][1]+dy*j],
                  [s[1][0]+dx*i,s[1][1]+dy*j],GRN));
   });
  }
  return o;
 }});

/* ═══ المصفوفة القطبية ═══
   نقرةٌ واحدةٌ: المركز. والعددُ والزاويةُ من الشريط، والخطوةُ
   تُقال في السجلّ فلا تُخمَّن. */
defTool({
 id:"arraypolar", alias:"arrp مصفوفه_قطبيه", label:"مصفوفة قطبية",
 hint:"انقر مركز الدوران — العددُ والزاويةُ من الشريط",
 opts:[
  {k:"n",    label:"عدد التكرارات",     type:"num",def:6},
  {k:"total",label:"الزاوية الإجمالية °",type:"num",def:360,
   hint:"٣٦٠ توزّع دورةً كاملة · وما دونها تمتدّ من الأصل إلى "
    +"آخر نسخة"},
  {k:"rot",  label:"دوّر النسخ",type:"chk",def:1,
   hint:"مطفأً يدور الموضعُ وتبقى الهيئة"},
  OPTS_OPEN],
 start(ctx){return needSel(ctx)},
 steps:[
  {p:"مركز الدوران",
   each(ctx,p){
    const r=arrayPolar(ctx.v.G,p, ovNum("arraypolar","total"),
     Math.round(ovNum("arraypolar","n")),
     ovOn("arraypolar","rot"), ovOn("arraypolar","opens"));
    (r.made||[]).forEach(s=>rec(ctx,{id:s.id},COLL[s.k]));
    H.rep("ok",`${r.n} تكراراً حول ${pt2(p)} م · الخطوة `
     +`${r.step}°${r.full?" (دورةٌ كاملة)":""} · ${r.walls} عنصر`
     +(r.opens?` و ${r.opens} فتحة`:""));
    (r.refused||[]).slice(0,6).forEach(x=>H.rep("wr",
     `  تُخطّي ${x} — لا يدور إلّا بمضاعفات 90°`));
    if((r.refused||[]).length>6)
     H.rep("in",`  … و ${r.refused.length-6} تخطّياً آخر`);
    if(ctx.v.G.some(x=>x.s.k==="col"))
     H.rep("in","  وأوسامُ الأعمدة لا تُنسَخ — "
      +"استعمل «أعد الترقيم» على المحدَّد");
   }}],
 prev(ctx,g){
  if(!g||!ctx.v.G)return [];
  const N=clamp(Math.round(ovNum("arraypolar","n"))||2,2,200);
  const T=ovNum("arraypolar","total")||0;
  const full=Math.abs(Math.abs(T)-360)<0.05;
  const st=T/(full?N:Math.max(1,N-1));
  const rot=ovOn("arraypolar","rot");
  const S2=segsOf(ctx.v.G);
  const o=[pvLine([g[0]-300,g[1]],[g[0]+300,g[1]],YEL),
           pvLine([g[0],g[1]-300],[g[0],g[1]+300],YEL)];
  let budget=700;
  for(let k=1;k<N&&budget>0;k++){
   const a=st*k;
   S2.forEach(s=>{
    if(budget--<=0)return;
    if(rot){
     o.push(pvLine(rotP(s[0],g,a),rotP(s[1],g,a),GRN));
     return;
    }
    /* بلا دوران: الموضعُ يدور والهيئةُ تبقى — والمرجعُ منتصفُ
       المسار، وهو مركزُ صندوقه في الجدار. */
    const an=[(s[0][0]+s[1][0])/2,(s[0][1]+s[1][1])/2];
    const q=rotP(an,g,a);
    const ddx=q[0]-an[0], ddy=q[1]-an[1];
    o.push(pvLine([s[0][0]+ddx,s[0][1]+ddy],
                  [s[1][0]+ddx,s[1][1]+ddy],GRN));
   });
  }
  return o;
 }});

/* ═══ مطابقة الخصائص ═══
   تقرأ حقول المصدر بـ readField ثم تكتبها بـ applyField، فتنالها
   المُثبِّتات نفسها: ما لا يملكه الهدف يُتخطّى، وما يُرفَض يُذكَر
   بسببه — لا كتابة عمياء.
   النصّ البديل للبُعد لا يُنسَخ افتراضاً: نسخُ رقمٍ يدويّ إلى بُعدٍ
   آخر يعرض مقاساً كاذباً بهيئة يقين. */
defTool({
 id:"match", alias:"ma مطابقه انسخ_الخصائص", label:"مطابقة",
 destruct:1,
 hint:"انقر المصدر ثم الأهداف من نوعه · Enter ينهي",
 opts:[{k:"txt",label:"انسخ النصّ البديل",type:"chk",def:0,
  hint:"للأبعاد — مطفأ لئلّا يُنسَخ رقمٌ يدويّ"}],
 steps:[
  {p:"انقر العنصر المصدر", ent:1, k:"src",
   each(ctx,hit){
    const K=hit.k;
    if(!FLD[K])
     throw new Error(`${NAME[K]||K} بلا حقول تُنسَخ`);
    const drop=ovOn("match","txt")?[]:["txt"];
    const F=[];
    FLD[K].forEach(f=>{
     if(drop.includes(f.k))return;
     const r=readField(K,[hit],f.k);
     if(!r.own)return;
     /* الطول يُقرأ مليمتراً ويُكتَب متراً — M() يفهم الرقم متراً */
     F.push({k:f.k,n:f.n,
      raw:(f.t==="len")?mnum(r.value):r.value});
    });
    if(!F.length)throw new Error(`${hit.id} بلا حقول يملكها`);
    ctx.v.kind=K; ctx.v.flds=F;
    H.rep("in",`المصدر ${hit.id} ${NAME[K]||K} · ${F.length} حقلاً: `
     +F.map(f=>f.n).join(" · "));
   }},
  {p:"انقر الهدف (Enter ينهي)", ent:1, loop:1,
   each(ctx,hit){
    const K=ctx.v.kind;
    if(hit.k!==K)
     throw new Error(`المصدر ${NAME[K]||K} — انقر ${NAME[K]||K} `
      +`مثله`);
    if(hit.id===ctx.v.src.id)
     throw new Error("هذا هو المصدر نفسه");
    let done=0;
    const ref=[];
    ctx.v.flds.forEach(f=>{
     try{
      const r=applyField(K,[hit],f.k,f.raw);
      if(r.done)done++;
      r.refused.forEach(x=>ref.push(`${f.n}: ${x.msg}`));
     }catch(e){ref.push(`${f.n}: ${e.message}`)}
    });
    if(done)dirty(ctx);
    H.rep(ref.length?"wr":"ok",
     `${hit.id} ← ${ctx.v.src.id} · ${done} من `
     +`${ctx.v.flds.length} حقلاً`);
    ref.slice(0,5).forEach(m=>H.rep("er","  "+m));
    if(ref.length>5)H.rep("in",`  … و ${ref.length-5} رفضاً آخر`);
   }}]});

/* ═══ قسمة الجدار ═══
   قطعٌ متكرّر عند نقاطٍ محسوبة على مسار الأصل — بعقد breakWall
   نفسها: الفتحة العابرة نقطة القطع تُحذَف ويُذكَر عددها، والباقية
   تنتقل بموضعها معادَ القياس من البداية الجديدة. */
defTool({
 id:"divide", alias:"dv اقسم قسمه", label:"قسمة",
 destruct:1,
 hint:"اختر جداراً — يُقسَم أجزاءً متساوية · Enter ينهي",
 opts:[{k:"n",label:"عدد الأجزاء",type:"num",def:2}],
 steps:[
  {p:"اختر الجدار المقسوم", ent:"wall", entName:"جدار", loop:1,
   each(ctx,hit){
    const n=clamp(Math.round(ovNum("divide","n"))||2,2,40);
    const w=wallById(hit.id);
    if(!w)throw new Error("الجدار غير موجود");
    const u=dir(w);
    if(!u)throw new Error("الجدار صفري");
    const seg=u.L/n;
    if(seg<MINW)
     throw new Error(`الجزء ${m3(seg)} م — الأدنى ${m3(MINW)} م · `
      +`الأقصى ${Math.floor(u.L/MINW)} جزءاً`);
    const A=w.a.slice();
    let cur=hit.id, lost=0, moved=0;
    for(let i=1;i<n;i++){
     const r=breakWall(cur,
      [Math.round(A[0]+u.ux*seg*i), Math.round(A[1]+u.uy*seg*i)]);
     rec(ctx,r.nw,"walls");
     lost+=r.lost; moved+=r.moved;
     cur=r.nw.id;
    }
    H.rep(lost?"wr":"ok",
     `قُسِم ${hit.id} إلى ${dim2(n,m3(seg),"م")}`
     +(moved?` · ${moved} فتحة انتقلت`:"")
     +(lost?` · حُذفت ${lost} فتحة تعبر نقاط القطع`:"")
     +` · الأطراف لم تُلحَم`);
   }}],
 prev(ctx,g){
  if(!g)return [];
  const h=H.hit(g[0],g[1]);
  if(!h||h.k!=="wall")return [];
  const w=wallById(h.id), u=w&&dir(w);
  if(!u)return [];
  const n=clamp(Math.round(ovNum("divide","n"))||2,2,40);
  const seg=u.L/n, t=Math.max(w.t,300), o=[];
  const c=(seg<MINW)?RED:GRN;
  for(let i=1;i<n;i++){
   const q=[w.a[0]+u.ux*seg*i, w.a[1]+u.uy*seg*i];
   o.push(pvLine([q[0]+u.nx*t,q[1]+u.ny*t],
                 [q[0]-u.nx*t,q[1]-u.ny*t],c));
  }
  return o;
 }});

/* ═══ تحديد بالمعرّف ═══
   أدوات التعديل تقرأ تحديداً قائماً، ولم يكن للتحديد طريقٌ إلا
   الفأرة. هذه تُكملها: تحدّد بالكتابة ثم تنقل أو تنسخ. */
/* ليست هادمة: تقرأ وتكتب في التحديد ولا تمسّ هندسةً */
defTool({
 id:"sel", alias:"se اختر تحديد", label:"تحديد بالمعرّف",
 hint:"W3 · O5 · -K2 يزيل · «الكل» · Enter ينهي",
 opts:[],
 steps:[{p:"معرّف عنصر (Enter ينهي)", text:1, loop:1,
  each(ctx,tok){
   const t=String(tok||"").trim();
   if(!t)return;
   if(/^(الكل|كلها|all)$/.test(norm(t))){
    const L=pickEnts();
    H.setSel(L);
    H.rep("ok",`${L.length} عنصر محدَّد`);
    return;
   }
   const neg=/^-/.test(t);
   const f=findById(t.replace(/^[-+]/,""));
   if(!f)throw new Error(`لا عنصر بالمعرّف «${t}»`);
   if(!pickable(f))throw new Error(`${f.id} مخفيّ أو مقفل`);
   const L=H.sel().filter(x=>!(x.k===f.k&&x.id===f.id));
   if(!neg)L.push(f);
   H.setSel(L);
   H.rep("in",`${neg?"أُزيل":"أُضيف"} ${f.id} `
    +`${NAME[f.k]||f.k} · المجموع ${L.length}`);
  }}]});

/* ═══ حذف ═══
   كان الحذف مفتاحاً في app.js وفعلاً في القائمة السياقية وحدهما:
   لا سطر إدخال يحذف، ولا خطّة مساعدٍ تحذف، ولا لوحة أوامر تجده.
   إدخاله السجلّ يُنيله ما ينال بقيّة الأوامر — اسمٌ يُكتَب، وعلَمٌ
   هادم تقرؤه البوّابة، وموضعٌ في الشريط.

   والتنفيذ يُفوَّض إلى delSel نفسه: لا منطق حذفٍ ثانٍ يتخلّف عن
   الأول. وهو يمرّ بـ edit() فيدير لقطته وتاريخه — فلا dirty(ctx)
   هنا، وإلّا دُفعت خطوتا تراجعٍ لعمليةٍ واحدة. */
defTool({
 id:"del", alias:"احذف امسح حذف", label:"حذف",
 destruct:1,
 hint:"يحذف التحديد القائم · Ctrl+Z يستعيده",
 opts:[],
 start(ctx){
  if(!H.sel().length){
   H.rep("wr","حدّد عناصر أولاً — الحذف يقع على التحديد");
   return false;
  }
  const r=H.del();
  if(!r){
   H.rep("wr","لم يُحذف شيء — المخفيّ والمقفل خارج التحديد");
   return false;
  }
  H.rep("ok","حُذف "+delSay(r)
   +(r.skipped?` · تُخطّي ${r.skipped}`:""));
  return false;                    /* أمر لحظي — لا خطوات */
 },
 steps:[]});
```

<a id="f-js-tools-openings-js"></a>

---

## `js/tools/openings.js`

```javascript
/* ═══ أدوات الفتحات ═══
   النقر يختار الجدار، والموضع من نقرتك أو من حقل «عند».
   لا تقليم صامت: ما لا يتّسع يُرفَض برسالة تذكر المدى المتاح. */
import {S} from "../core/state.js";
import {m2,m3,M,rng3,dim2} from "../core/units.js";
import {wallById,wallLen,wallAt,dir} from "../core/walls.js";
import {addOpen,sAt,openPt,openById,allowed,saySpans,okName,OK} from "../core/opens.js";
import {defTool,H,rec,ov,ovLen,ovNum,ovOn,
        pvLine,pvBand,pvText} from "./registry.js";

const GRN="#5cd98e", RED="#ff6f6f";

/* الموضع على الجدار: حقل «عند» إن كُتب، وإلا إسقاط النقرة.
   c أو 50% يعني المنتصف — أوفر من مطاردة نقطةٍ بالفأرة. */
function sOf(w,p,tid){
 const raw=String(ov(tid,"at")||"").trim();
 const L=wallLen(w);
 if(!raw)return sAt(w,p);
 if(/^(c|م|وسط)$/i.test(raw))return Math.round(L/2);
 const pc=/^(\d*\.?\d+)%$/.exec(raw);
 if(pc)return Math.round(L*parseFloat(pc[1])/100);
 const v=M(raw);
 return (ov(tid,"from")==="b")?(L-v):v;
}
function place(ctx,hit,p,tid,kind){
 const w=wallById(hit.id);
 if(!w)throw new Error("انقر على جدار");
 const W=ovLen(tid,"w"), Hh=ovLen(tid,"h"), sl=ovLen(tid,"sill");
 const s=sOf(w,p,tid);
 const ex={hinge:ov(tid,"hinge"),swing:ov(tid,"swing")};
 if(OK[kind].pan)ex.pan=Math.round(ovNum(tid,"pan"))||1;
 if(kind==="niche"){
  ex.dep=ovLen(tid,"dep");
  ex.face=ov(tid,"face");
 }
 const o=addOpen(w,s,kind,W,Hh,sl,ex);
 rec(ctx,o,"opens");
 H.rep("ok",`${o.id} ${okName(kind)} ${dim2(m2(o.w),m2(o.h))} م `
  +`على ${w.id} عند ${m2(o.s)} م`
  +(o.sill?` · جلسة ${m2(o.sill)} م`:"")
  +(o.dep!=null?` · عمق ${m2(o.dep)} م`:""));
 return o;
}
function mk(id,alias,label,kind,dw,dh,ds,extra){
 const opts=[
  {k:"w",   label:"العرض م",   type:"len", def:dw},
  {k:"h",   label:"الارتفاع م",type:"len", def:dh},
  {k:"sill",label:"الجلسة م",  type:"len", def:ds},
  {k:"at",  label:"عند م",     type:"text", def:"",
   hint:"فارغ = من النقر · c = المنتصف · 50% · أو طول بالمتر"},
  {k:"from",label:"من",        type:"sel",
   items:[["a","بداية الجدار"],["b","نهايته"]], def:"a"}
 ].concat(extra||[]);
 defTool({
  id, alias, label,
  hint:"انقر على جدار — الموضع من نقرتك أو من حقل «عند»",
  opts,
  steps:[{p:"انقر موضع الفتحة على جدار (أو اكتب W7@2.4)",
   ent:"wall", entName:"جدار", loop:1,
   entVia:{open:h=>{
    const o=openById(h.id);
    return o?{k:"wall",id:o.wall}:null;
   }},
   each(ctx,hit,p){
    place(ctx,hit,p,id,
     (id==="door")?(ov("door","kind")||"door"):kind);
   }}],
  prev(ctx,g){
   if(!g)return [];
   const w=wallAt(g[0],g[1],250);
   if(!w)return [];
   const d=dir(w);
   if(!d)return [];
   const W=ovLen(id,"w")||900;
   const s=Math.round(sOf(w,g,id));
   const A=allowed(w,W,null);
   const ok=A.fits&&A.spans.some(([a,b])=>s>=a&&s<=b);
   const P=t=>[Math.round(w.a[0]+d.ux*t),
               Math.round(w.a[1]+d.uy*t)];
   return [
    pvLine(w.a,P(s),"#ffd06b"),
    pvBand(openPt(w,s-W/2),openPt(w,s+W/2),w.t,ok?GRN:RED),
    pvText(openPt(w,s),`${w.id} · ${m3(s)} م`
     +(ok?"":` ✗ الحرّ ${saySpans(A)}`), ok?GRN:RED)];
  }});
}
mk("door","d باب","باب","door","0.9","2.1","0",[
 {k:"kind", label:"النوع",   type:"sel",
  items:[["door","مفرد"],["double","مزدوج"],["sliding","سحب"]],
  def:"door"},
 {k:"hinge",label:"المفصّلة",type:"sel",
  items:[["start","البداية"],["end","النهاية"]],def:"start"},
 {k:"swing",label:"جهة الفتح",type:"sel",
  items:[["left","يسار المسار"],["right","يمينه"]],def:"left"}]);

mk("win","n window شباك نافذه","شباك","window","1.5","1.4","0.9",[
 {k:"pan",label:"المصاريع",type:"num",def:1}]);

mk("fixed","ثابت زجاج","شباك ثابت","fixed","1.2","1.4","0.9",[
 {k:"pan",label:"المصاريع",type:"num",def:1}]);

mk("opening","op فتحه","فتحة صافية","opening","2","2.1","0");
mk("arch","قنطره","فتحة مقنطرة","arch","1.2","2.2","0");
mk("niche","كوه حنيه","كوّة","niche","0.6","1.2","0.9",[
 {k:"dep", label:"العمق م",type:"len",def:"0.12",
  hint:"الأقصى = سماكة الجدار − 4 سم"},
 {k:"face",label:"الوجه",  type:"sel",
  items:[["l","يسار المسار"],["r","يمينه"]],def:"l"}]);
```

<a id="f-js-tools-parts-js"></a>

---

## `js/tools/parts.js`

```javascript
/* ═══ أدوات الأعمدة والأدوات الصحية والدرج ═══
   كلّها تبقى فعّالة حتى Esc، وكلّها تخزّن إحداثيات صريحة:
   «ألصِق بالجدار» أمرٌ يُنفَّذ عند الوضع لا رابطةٌ تُحفَظ. */
import {S} from "../core/state.js";
import {m2,m3,clamp,dm2} from "../core/units.js";
import {pip} from "../core/geom.js";
import {band} from "../core/walls.js";
import {axLabel} from "../core/dims.js";
import {addCol,nextTag,colLabel,CK,CT,colOnWall} from "../core/cols.js";
import {addFix,snapToWall,FK,fixName} from "../core/fixt.js";
import {addStair,stCheck} from "../core/stairs.js";
import {defTool,H,rec,ov,ovLen,ovNum,ovOn,
        pvLine} from "./registry.js";

const GRN="#5cd98e", YEL="#ffd06b", BLU="#5aa9ff", RED="#ff6f6f";
const RCT=(p,w,h,rot,c)=>{
 const a=(rot||0)*Math.PI/180, ca=Math.cos(a), sa=Math.sin(a);
 const Q=[[-w/2,-h/2],[w/2,-h/2],[w/2,h/2],[-w/2,h/2]]
  .map(q=>[p[0]+q[0]*ca-q[1]*sa, p[1]+q[0]*sa+q[1]*ca]);
 return Q.map((q,i)=>({t:"l",a:q,b:Q[(i+1)%4],c}));
};
/* ═══ عمود ═══ */
defTool({
 id:"col", alias:"k عمود", label:"عمود",
 hint:"انقر مركز العمود · Enter ينهي",
 opts:[
  {k:"kind",label:"الشكل",type:"sel",
   items:[["rect","مستطيل"],["circ","دائري"]],def:"rect"},
  {k:"w",   label:"العرض / القطر م",type:"len",def:"0.3"},
  {k:"h",   label:"العمق م",        type:"len",def:"0.3",
   when:o=>o.kind!=="circ"},
  {k:"rot", label:"الدوران °",      type:"num",def:0,
   when:o=>o.kind!=="circ"},
  {k:"type",label:"المادة",type:"sel",
   items:[["conc","خرسانة"],["steel","حديد"],["stone","حجر"]],
   def:"conc"},
  {k:"tag", label:"رقّم تلقائياً",type:"chk",def:1}],
 steps:[
  {p:"مركز العمود (Enter ينهي)", base:"none", loop:1,
   each(ctx,p){
    const kind=ov("col","kind");
    const c=addCol(kind,p,ovLen("col","w"),ovLen("col","h"),
     ovNum("col","rot"),ov("col","type"),
     ovOn("col","tag")?nextTag("C"):"");
    rec(ctx,c,"cols");
    const on=colOnWall(c,2);
    H.rep("ok",`${c.id}${c.tag?" "+c.tag:""} ${CK[c.kind]} `
     +`${colLabel(c)} · ${CT[c.type]}`
     +(on?` · يُدمَج مع ${on}`:` · منفرد`));
   }}],
 prev(ctx,g){
  if(!g)return [];
  const w=ovLen("col","w");
  if(ov("col","kind")==="circ"){
   const r=w/2, o=[];
   let pr=null;
   for(let i=0;i<=24;i++){
    const a=i/24*Math.PI*2;
    const q=[g[0]+r*Math.cos(a), g[1]+r*Math.sin(a)];
    if(pr)o.push({t:"l",a:pr,b:q,c:GRN});
    pr=q;
   }
   return o;
  }
  return RCT(g,w,ovLen("col","h")||w,ovNum("col","rot"),GRN);
 }});

/* ═══ أداة صحية ═══ */
function mkFix(id,alias,label,kind){
 const d=FK[kind];
 defTool({
  id, alias, label,
  hint:"انقر الموضع — يُلصَق بأقرب جدار إن فُعّل الخيار",
  opts:[
   {k:"snap",label:"ألصِق بالجدار",type:"chk",def:1,
    hint:"أمرٌ عند الوضع لا رابطة تُحفَظ"},
   {k:"rot", label:"الدوران °",type:"num",def:0},
   {k:"w",   label:"العرض م",type:"len",def:String(d.w/1000)},
   {k:"d",   label:"العمق م",type:"len",def:String(d.d/1000)},
   {k:"mir", label:"معكوسة",type:"chk",def:0}],
  steps:[
   {p:"موضع الأداة (Enter ينهي)", base:"none", loop:1,
    each(ctx,p){
     let at=p, rot=ovNum(id,"rot"), on=null;
     if(ovOn(id,"snap")){
      const s=snapToWall(p,1500);
      if(s){at=s.p; rot=s.rot; on=s.wall}
     }
     const f=addFix(kind,at,rot,{
      w:ovLen(id,"w"), d:ovLen(id,"d"),
      mir:ovOn(id,"mir")?1:0});
     rec(ctx,f,"fixt");
     H.rep("ok",`${f.id} ${fixName(f)} `
      +`${dm2(f.w,f.d,"م")}`
      +(on?` · مُلصَقة بـ ${on} (إحداثيات صريحة بعدها)`
        :` · حرّة`));
    }}],
  prev(ctx,g){
   if(!g)return [];
   let at=g, rot=ovNum(id,"rot");
   if(ovOn(id,"snap")){
    const s=snapToWall(g,1500);
    if(s){at=s.p; rot=s.rot}
   }
   const w=ovLen(id,"w")||d.w, dd=ovLen(id,"d")||d.d;
   const a=rot*Math.PI/180, ca=Math.cos(a), sa=Math.sin(a);
   const m=ovOn(id,"mir")?-1:1;
   const P=(u,v)=>[at[0]+(u*m)*ca-v*sa, at[1]+(u*m)*sa+v*ca];
   const Q=[P(-w/2,0),P(w/2,0),P(w/2,dd),P(-w/2,dd)];
   const o=Q.map((q,i)=>({t:"l",a:q,b:Q[(i+1)%4],c:GRN}));
   o.push({t:"l",a:P(0,0),b:P(0,dd*0.3),c:YEL});  /* دلالة الظهر */
   return o;
  }});
}
mkFix("wc",    "كرسي مرحاض", "كرسي",   "wc");
mkFix("lav",   "مغسله",      "مغسلة",  "lav");
mkFix("shower","دش",         "دُش",     "shower");
mkFix("tub",   "بانيو حوض",  "بانيو",  "tub");
mkFix("sink",  "مجلى",       "حوض مطبخ","sink");
mkFix("bidet", "شطاف",       "شطّاف",   "bidet");
mkFix("ur",    "مبوله",      "مبولة",  "ur");
mkFix("wm",    "غساله",      "غسّالة",  "wm");
mkFix("fd",    "صفايه",      "صفاية",  "fd");

/* ═══ أعمدة على تقاطعات المحاور ═══
   أمرٌ يُنفَّذ مرّة على S.grid: لكل تقاطعٍ عمودٌ بمقاسك، بترتيب
   القراءة العربية (من أعلى اليمين) فيأتي الترقيم منتظماً.
   ما وُجد عمودُه يُتخطّى ويُذكَر — لا تكرار ولا إزاحة صامتة. */
defTool({
 id:"gridcols", alias:"gk اعمده_المحاور شبكه_اعمده",
 label:"أعمدة المحاور",
 hint:"عمود على كل تقاطع محورين — الموجود يُتخطّى ولا يُزاح",
 opts:[
  {k:"kind",label:"الشكل",type:"sel",
   items:[["rect","مستطيل"],["circ","دائري"]],def:"rect"},
  {k:"w",   label:"العرض / القطر م",type:"len",def:"0.4"},
  {k:"h",   label:"العمق م",type:"len",def:"0.4",
   when:o=>o.kind!=="circ"},
  {k:"rot", label:"الدوران °",type:"num",def:0,
   when:o=>o.kind!=="circ"},
  {k:"type",label:"المادة",type:"sel",
   items:[["conc","خرسانة"],["steel","حديد"],["stone","حجر"]],
   def:"conc"},
  {k:"pre", label:"سابقة الوسم",type:"text",def:"C"},
  {k:"tag", label:"رقّم",type:"chk",def:1},
  {k:"inside",label:"داخل الجدران وحدها",type:"chk",def:0,
   hint:"يشترط أن يقع التقاطع في جسم جدار"}],
 start(ctx){
  const X=S.grid.xs, Y=S.grid.ys;
  if(!X.length||!Y.length){
   H.rep("wr","تحتاج محوراً رأسياً وأفقياً على الأقلّ — "
    +"استعمل أداة «محور»");
   return false;
  }
  const N=X.length*Y.length;
  if(N>400){
   H.rep("er",`${N} تقاطعاً — أكثر من 400. امسح محاور أو نفّذها `
    +`على دفعات`);
   return false;
  }
  const inside=ovOn("gridcols","inside");
  const bands=inside?S.walls.map(band).filter(Boolean):null;
  const P=[];
  X.forEach((x,i)=>Y.forEach((y,j)=>P.push({x,y,i,j})));
  /* من أعلى اليمين: y نازلاً ثم x نازلاً — كترقيم renumberCols */
  P.sort((a,b)=>(b.y-a.y)||(b.x-a.x));
  const kind=ov("gridcols","kind");
  const pre=String(ov("gridcols","pre")||"C");
  let made=0, dup=0, out=0;
  const dupIds=[];
  P.forEach(q=>{
   if(bands&&!bands.some(bp=>pip(bp,q.x,q.y))){out++; return}
   try{
    const c=addCol(kind,[q.x,q.y],
     ovLen("gridcols","w"), ovLen("gridcols","h"),
     ovNum("gridcols","rot"), ov("gridcols","type"),
     ovOn("gridcols","tag")?nextTag(pre):"");
    rec(ctx,c,"cols");
    made++;
   }catch(e){
    dup++;
    dupIds.push(`${axLabel("x",q.i)}${axLabel("y",q.j)}`);
   }
  });
  H.rep(made?"ok":"wr",`${made} عموداً على ${N} تقاطعاً · `
   +`${colLabel({kind,w:ovLen("gridcols","w"),
     h:ovLen("gridcols","h")})}`);
  if(dup)H.rep("in",`تُخطّي ${dup} تقاطعاً عليه عمود سلفاً`
   +(dupIds.length<=12?`: ${dupIds.join(" · ")}`:""));
  if(out)H.rep("in",`تُخطّي ${out} تقاطعاً خارج الجدران`);
  return false;                    /* أمر لحظي — لا خطوات */
 },
 steps:[]});

/* ═══ درج ═══ */
defTool({
 id:"stair", alias:"st درج سلم", label:"درج",
 hint:"بداية القِلعة ثم نهايتها · القياسات تُقاس ولا تُصحَّح",
 opts:[
  {k:"w",  label:"العرض م",   type:"len",def:"1.1"},
  {k:"n",  label:"عدد القوائم",type:"num",def:16},
  {k:"h",  label:"ارتفاع الدور م",type:"len",def:"",
   hint:"فارغ = من إعداد المشروع"},
  {k:"up", label:"الاتجاه",type:"sel",
   items:[["up","صاعد"],["dn","هابط"]],def:"up"},
  {k:"cut",label:"خطّ القطع",type:"num",def:0,
   hint:"0 = بلا · 0.6 = عند 60٪"}],
 steps:[
  {p:"بداية القِلعة"},
  {p:"نهاية القِلعة", base:0, restart:1,
   each(ctx,p){
    const st=addStair(ctx.pts[0],p, ovLen("stair","w"),
     ovNum("stair","n"),
     {h:ovLen("stair","h")||0, up:ov("stair","up"),
      cut:ovNum("stair","cut")});
    rec(ctx,st,"stairs");
    const c=stCheck(st);
    H.rep(c.ok?"ok":"wr",
     `${st.id} ${c.n} قائمة · ق ${m3(c.rise)} · ن ${m3(c.tread)} م `
     +`· 2ق+ن ${m3(c.rule)} م`);
    c.msgs.forEach(m=>H.rep("wr","  "+m));
    if(!c.ok)H.rep("in","  القياسات كما رسمتها — عدّل الطول أو "
     +"عدد القوائم إن شئت");
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  const a=ctx.pts[0];
  const w=ovLen("stair","w")||1100;
  const n=clamp(Math.round(ovNum("stair","n"))||2,2,80);
  const dx=g[0]-a[0], dy=g[1]-a[1], L=Math.hypot(dx,dy);
  if(L<1)return [];
  const ux=dx/L, uy=dy/L, nx=-uy, ny=ux, hw=w/2;
  const P=(s,v)=>[a[0]+ux*s+nx*v, a[1]+uy*s+ny*v];
  const o=[pvLine(P(0,-hw),P(L,-hw),GRN),
           pvLine(P(0, hw),P(L, hw),GRN)];
  const t=L/(n-1);
  for(let i=0;i<=n-1;i++)
   o.push(pvLine(P(t*i,-hw),P(t*i,hw),(t<250)?RED:BLU));
  return o;
 }});
```

<a id="f-js-tools-presets-js"></a>

---

## `js/tools/presets.js`

```javascript
/* ═══ المقاسات والقوالب ═══
   المقاس يضبط خيارات أداة موجودة ثم يتركها فعّالة. القالب يمرّ
   من بوابة الخطط نفسها قبل تثبيته كي لا يكون له مسار تنفيذ خاص. */
import {setOpt,begin} from "./registry.js";

export const SIZES=[
 {id:"s.door.room",tool:"door",cat:"أبواب",label:"باب غرفة",
  sub:"0.90 × 2.10",set:{w:"0.9",h:"2.1",kind:"door",hinge:"start"}},
 {id:"s.door.main",tool:"door",cat:"أبواب",label:"باب مدخل",
  sub:"1.20 × 2.20 مزدوج",set:{w:"1.2",h:"2.2",kind:"double"}},
 {id:"s.door.wet",tool:"door",cat:"أبواب",label:"باب دورة مياه",
  sub:"0.70 × 2.00",set:{w:"0.7",h:"2",kind:"door"}},
 {id:"s.door.slide",tool:"door",cat:"أبواب",label:"باب سحب",
  sub:"1.60 × 2.10",set:{w:"1.6",h:"2.1",kind:"sliding"}},
 {id:"s.win.majlis",tool:"win",cat:"شبابيك",label:"شباك مجلس",
  sub:"2.00 × 1.60 · جلسة 0.90",set:{w:"2",h:"1.6",sill:"0.9",pan:2}},
 {id:"s.win.room",tool:"win",cat:"شبابيك",label:"شباك غرفة نوم",
  sub:"1.50 × 1.40 · جلسة 0.90",set:{w:"1.5",h:"1.4",sill:"0.9",pan:2}},
 {id:"s.win.wet",tool:"win",cat:"شبابيك",label:"شباك دورة مياه",
  sub:"0.60 × 0.60 · جلسة 1.60",set:{w:"0.6",h:"0.6",sill:"1.6",pan:1}},
 {id:"s.win.kitchen",tool:"win",cat:"شبابيك",label:"شباك مطبخ",
  sub:"1.20 × 1.00 · جلسة 1.10",set:{w:"1.2",h:"1",sill:"1.1",pan:2}},
 {id:"s.wall.ext",tool:"wall",cat:"جدران",label:"جدار خارجي",
  sub:"0.25 م",set:{t:"0.25",type:"ext",align:"c"}},
 {id:"s.wall.int",tool:"wall",cat:"جدران",label:"جدار داخلي",
  sub:"0.12 م",set:{t:"0.12",type:"int",align:"c"}},
 {id:"s.wall.part",tool:"wall",cat:"جدران",label:"قاطع خفيف",
  sub:"0.07 م",set:{t:"0.07",type:"int",align:"c"}},
 {id:"s.wall.low",tool:"wall",cat:"جدران",label:"سترة",
  sub:"0.15 م · ارتفاع 1.10",set:{t:"0.15",type:"low",h:"1.1"}}
];

export function applySize(p){
 Object.keys(p.set).forEach(k=>setOpt(p.tool,k,p.set[k]));
 return begin(p.tool);
}

const F=v=>(Math.round(v*1000)/1000).toFixed(2);
const P=(at,dx,dy)=>`${F(at[0]+dx)},${F(at[1]+dy)}`;
export const TPL=[
 {id:"t.room",label:"قالب: غرفة 4 × 3",sub:"مستطيل صافٍ 0.25 + منطقة",
  lines:at=>["rect","t=0.25","type=ext","align=l",P(at,0,0),"4x3","esc",
   "area","num=1","showArea=1",P(at,2,1.5),"esc"]},
 {id:"t.wet",label:"قالب: دورة مياه 2.4 × 1.6",sub:"قاطع 0.12 + منطقة مسمّاة",
  lines:at=>["rect","t=0.12","type=int","align=l",P(at,0,0),"2.4x1.6","esc",
   "area","name=دورة مياه","num=1",P(at,1.2,0.8),"esc"]},
 {id:"t.flat",label:"قالب: شقّة بثلاثة فراغات 10 × 8",
  sub:"خارجي 0.25 · قواطع 0.12 · ثلاث مناطق",
  lines:at=>["rect","t=0.25","type=ext","align=l",P(at,0,0),"10x8","esc",
   "wall","t=0.12","type=int","align=c",P(at,6,0),P(at,6,8),"esc",
   "wall",P(at,6,4),P(at,10,4),"esc",
   "area","num=1","showArea=1",P(at,3,4),P(at,8,2),P(at,8,6),"esc"]}
];
```

<a id="f-js-tools-ref-js"></a>

---

## `js/tools/ref.js`

```javascript
/* ═══ أدوات المرجع ═══
   ثلاثتها تكتب في التحويل المخزَّن ولا تلمس إحداثيات المرجع —
   فلا تراكم ولا فقدان للأصل.
   وثلاثتها destruct: تحرّك ما هو معروضٌ سلفاً وتُبطِل معايرةً
   قائمة، فلا يُصدِرها المزوّد بلا تصريح. */
import {S} from "../core/state.js";
import {m2,m3,arrow,pt2} from "../core/units.js";
import {hasRef,alignRef,calRef,moveRef} from "../core/ref.js";
import {defTool,H,dirty,ovLen,pvLine} from "./registry.js";

const YEL="#ffd06b", GRN="#5cd98e";
const need=()=>{
 if(hasRef())return true;
 H.rep("wr","لا مرجع مستورد — استورد DXF أوّلاً من لوحة المرجع");
 return false;
};
/* ═══ محاذاة: نقطتان على المرجع ثم موضعهما الصحيح ═══ */
defTool({
 id:"refalign", alias:"ra حاذِ محاذاه", label:"محاذاة المرجع",
 hint:"نقطتان على المرجع ثم وجهتاهما — يُحسَب النقل والدوران والمقياس",
 destruct:1,
 opts:[],
 start(){return need()},
 steps:[
  {p:"النقطة الأولى على المرجع"},
  {p:"النقطة الثانية على المرجع", base:0},
  {p:"وجهة النقطة الأولى", base:"none"},
  {p:"وجهة النقطة الثانية", base:2,
   each(ctx,p){
    const r=alignRef(ctx.pts[0],ctx.pts[1],ctx.pts[2],p);
    dirty(ctx);
    H.rep("ok",`المرجع: مقياس ×${r.k.toFixed(5)} · دوران `
     +`${r.rot.toFixed(2)}° · ${arrow(m3(r.from),m3(r.to))} م`);
    H.rep("in","الإحداثيات المستوردة لم تُمَسّ — التحويل مخزَّن "
     +"فالمحاذاة التالية لا تتراكم على هذه");
   }}],
 prev(ctx,g){
  const P=ctx.pts;
  if(!g)return [];
  if(P.length===1)return [pvLine(P[0],g,YEL)];
  if(P.length===2)return [pvLine(P[0],P[1],YEL),
   pvLine(P[1],g,"#3d4a58")];
  if(P.length===3)return [pvLine(P[0],P[1],YEL),
   pvLine(P[2],g,GRN)];
  return [];
 }});

/* ═══ معايرة: مسافة تعرفها ═══ */
defTool({
 id:"refcal", alias:"rc عاير معايره", label:"معايرة المرجع",
 hint:"نقطتان على المرجع ثم المسافة الحقيقية بينهما",
 destruct:1,
 opts:[{k:"d",label:"المسافة الحقيقية م",type:"len",def:"1"}],
 start(){return need()},
 steps:[
  {p:"النقطة الأولى"},
  {p:"النقطة الثانية", base:0,
   each(ctx,p){
    const r=calRef(ctx.pts[0],p,ovLen("refcal","d"));
    dirty(ctx);
    H.rep("ok",`عُوير المرجع ×${r.k.toFixed(5)} · `
     +`${arrow(m3(r.was),m3(r.now))} م`);
    if(S.ref.guessed)
     H.rep("in","الوحدة كانت مفترضة مليمتراً — صارت معايرتك هي "
      +"المرجع");
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  return [pvLine(ctx.pts[0],g,GRN)];
 }});

/* ═══ نقل ═══ */
defTool({
 id:"refmove", alias:"rm انقل_المرجع", label:"نقل المرجع",
 hint:"نقطة أساس ثم وجهتها",
 destruct:1,
 opts:[],
 start(){return need()},
 steps:[
  {p:"نقطة الأساس"},
  {p:"الوجهة", base:0,
   each(ctx,p){
    const r=moveRef(ctx.pts[0],p);
    dirty(ctx);
    H.rep("ok",`نُقل المرجع ${pt2([r.dx,r.dy])} م`);
   }}],
 prev(ctx,g){
  if(!ctx.pts.length||!g)return [];
  return [pvLine(ctx.pts[0],g,YEL)];
 }});
```

<a id="f-js-tools-registry-js"></a>

---

## `js/tools/registry.js`

```javascript
/* ═══ سجلّ الأدوات ═══
   الأداة = خيارات + خطوات + معاينة. تبقى فعّالة بعد كل عنصر
   حتى Esc، وخياراتها لزجة بين الجلسات.

   الخطوة: {p, k?, loop?, min?, restart?, base?, ent?, ang?,
            confirm?, opts?, each?}
   لا يوجد نظام سكربت: سطر الإدخال يقبل إحداثيات وأسماء أدوات فقط. */

import {S,snapshot,pushHistory,touch,autosave} from "../core/state.js";
import {norm,M,Mx,Nx,m2,clamp} from "../core/units.js";
import {parsePt,angOf,lenOf,D2R} from "../core/coords.js";
import {findById,shapeOf} from "../core/ents.js";
import {jrAdd,jrTaint} from "../core/journal.js";
/* لا دورة: ents.js لا يستورد tools/* */

export const H={
 draw:()=>{}, rep:()=>{}, prompt:()=>{}, refresh:()=>{},
 hit:()=>null, sel:()=>[], setSel:()=>{}, del:()=>null};

/* وضع الدفعة: تنفيذٌ متسلسل بخطوة تراجعٍ واحدة يديرها المُنادي */
export const BATCH={on:0};
export const setBatch=v=>{BATCH.on=v?1:0};

export const TOOLS={};
export const T={id:null,def:null,steps:null,i:0,ctx:null,
 snap:null,ghost:null,lock:null,lenLock:null,last:null,lastArg:null,
 hist:[]};

export function defTool(d){
 TOOLS[d.id]=d;
 (d.alias||"").split(/\s+/).filter(Boolean)
  .forEach(a=>{TOOLS[norm(a)]=d});
 return d;
}
export const findTool=n=>TOOLS[norm(n||"")]||null;
export const toolList=()=>[...new Set(Object.values(TOOLS))];
export const active=()=>!!T.def;
export const step=()=>T.steps?T.steps[T.i]:null;

/* ═══ الأداة الهادمة ═══
   تقصّ أو تحرّك ما هو مرسوم سلفاً. العلَم مُعلَنٌ في تعريف الأداة
   فلا تتخلّف قائمةٌ ثانية عن السجلّ — وكان جدولٌ يدويّ في
   ai/plan.js يفوته «نقل» و«دوران» و«مرآة»، وثلاثتها تحرّك ما هو
   مرسوم. ويُعلَن في lang.js فيراه المزوّد نصّاً، والبوّابة تقرؤه
   كوداً — والتعليمات يمكن التحدّث حولها، والكود لا.
   وموضعُه بعد toolList بقصد: destructList تنادِيه. */
export const isDestruct=d=>!!(d&&d.destruct);
export const destructList=()=>toolList()
 .filter(d=>d&&d.id&&d.destruct)
 .map(d=>({id:d.id,label:d.label}));

/* ═══ خيارات الأدوات — لزجة ═══ */
const OKEY="mistar.opts";
export const OPT={};
export function loadOpts(){
 try{
  const raw=localStorage.getItem(OKEY);
  if(raw)Object.assign(OPT,JSON.parse(raw)||{});
 }catch(e){}
 toolList().forEach(d=>{
  const o=OPT[d.id]=OPT[d.id]||{};
  (d.opts||[]).forEach(f=>{if(o[f.k]===undefined)o[f.k]=f.def});
 });
}
export function saveOpts(){
 try{localStorage.setItem(OKEY,JSON.stringify(OPT))}catch(e){}
}
export function setOpt(tid,k,v){
 (OPT[tid]=OPT[tid]||{})[k]=v;
 saveOpts();
}
/* قارئات القيَم — يستعملها كل أمر رسم */
export const ov=(tid,k)=>{
 const o=OPT[tid]||{};
 return (o[k]===undefined)?null:o[k];
};
export const ovLen=(tid,k)=>M(String(ov(tid,k)==null?"":ov(tid,k)));
export const ovNum=(tid,k)=>{
 const v=parseFloat(ov(tid,k));
 return isFinite(v)?v:0;
};
export const ovOn=(tid,k)=>{
 const v=ov(tid,k);
 return v===1||v===true||v==="1"||v==="true";
};
/* ═══ نقطة الأساس والاتجاه ═══ */
export function baseOf(){
 const st=step();
 if(!st)return null;
 const P=T.ctx?T.ctx.pts:[];
 if(st.base==="none")return null;
 if(typeof st.base==="number")
  return P[st.base<0?P.length+st.base:st.base]||null;
 return P.length?P[P.length-1]:null;
}
export function dirOf(){
 const b=baseOf();
 if(!b)return null;
 if(T.lock!=null)
  return [Math.cos(T.lock*D2R),Math.sin(T.lock*D2R)];
 if(!T.ghost)return null;
 const dx=T.ghost[0]-b[0], dy=T.ghost[1]-b[1];
 const L=Math.hypot(dx,dy);
 return L>1?[dx/L,dy/L]:null;
}
export function promptText(){
 const st=step();
 if(!st)return {tool:"",p:"أداة:",live:""};
 let p=st.p;
 if(st.opts)p+=" ["+Object.keys(st.opts)
  .map(k=>`${st.opts[k].n}(${k.toUpperCase()})`).join("/")+"]";
 const b=baseOf();
 let live="";
 if(b&&T.ghost){
  live=`${(lenOf(b,T.ghost)/1000).toFixed(3)} م  `
      +`${angOf(b,T.ghost).toFixed(1)}°`;
  if(T.lock!=null)live+=`  ⟨قفل ${T.lock}°⟩`;
  if(T.lenLock!=null)live+=`  ⟨طول ${(T.lenLock/1000).toFixed(3)}⟩`;
 }
 return {tool:T.def.label||T.id,p:p+":",live};
}
/* ═══ الدخول والخروج ═══ */
function reset(){
 T.id=null; T.def=null; T.steps=null; T.i=0;
 T.ctx=null; T.snap=null; T.lock=null; T.lenLock=null; T.ghost=null;
}
/* الوسيط: سطر الإدخال يقتطعه بعد اسم الأداة، والأداة تقرؤه من
   ctx.arg. ومن لا يعرفه يتجاهله — فالتعديل لا يمسّ أداةً قائمة. */
export function begin(id,arg){
 const d=(id&&typeof id==="object")?id:findTool(id);
 if(!d){H.rep("wr",`أداة غير معروفة: ${id}`);return false}
 if(T.def)cancel(true);
 T.def=d; T.id=d.id;
 T.steps=(d.steps||[]).slice();
 T.i=0; T.lock=null; T.lenLock=null; T.ghost=null;
 T.ctx={pts:[],v:{},n:0,made:[],
  arg:(arg==null||arg==="")?null:arg};
 T.snap=snapshot();
 T.last=d.id; T.lastArg=T.ctx.arg;
 jrAdd(d.id);
 if(T.ctx.arg)jrTaint("وسيط أداة");
 if(d.start){
  let ok=true;
  try{ok=d.start(T.ctx)}
  catch(e){H.rep("er",e.message);ok=false}
  if(ok===false){
   /* أمرٌ لحظيّ أو تحديدٌ ناقص: نُثبّت ما صنعه إن صنع */
   const n=T.ctx?T.ctx.made.length:0;
   const sn=T.snap, lbl=d.label||d.id;
   reset();
   if(n){
    touch();
    if(!BATCH.on){pushHistory(sn,lbl); H.refresh(); autosave()}
   }
   H.prompt(); H.draw();
   return true;
  }
 }
 H.prompt(); H.draw();
 return true;
}
export function finish(msg){
 const ctx=T.ctx, sn=T.snap, d=T.def, lbl=(d&&(d.label||d.id))||null;
 reset();
 if(d&&d.done&&ctx){try{d.done(ctx)}catch(e){H.rep("er",e.message)}}
 const n=ctx?ctx.made.length:0;
 if(n){
  touch();
  if(!BATCH.on){pushHistory(sn,lbl); H.refresh(); autosave()}
 }
 if(msg)H.rep(n?"ok":"in",msg);
 H.prompt(); H.draw();
}
export function cancel(quiet){
 const ctx=T.ctx, sn=T.snap, lbl=(T.def&&(T.def.label||T.def.id))||null;
 const n=(ctx&&ctx.made.length)||0;
 reset();
 if(ctx)jrAdd("esc");
 if(n){
  touch();
  if(!BATCH.on){pushHistory(sn,lbl); H.refresh(); autosave()}
 }
 if(!quiet)H.rep("in",n?`أُلغي — بقي ${n} عنصر`:"أُلغي");
 H.prompt(); H.draw();
}
/* restart: الأداة تعيد نفسها من الخطوة الأولى بعد كل عنصر —
   تنفيذاً لقاعدة «الأداة تبقى فعّالة حتى Esc» */
function restart(){
 T.i=0; T.lock=null; T.lenLock=null; T.ghost=null;
 if(T.ctx){T.ctx.pts.length=0; T.ctx.v={}; T.ctx.n=0}
 H.prompt();
}
function advance(){
 const st=step();
 if(st&&st.restart){restart(); return}
 if(st&&st.loop){T.ctx.n++; T.lock=null; T.lenLock=null;
  H.prompt(); return}
 T.i++; T.lock=null; T.lenLock=null;
 if(!step())finish(); else H.prompt();
}
export const nextStep=advance;
/* إدراج خطوات فرعية بعد الجارية — للأدوات المتفرّعة مثل rotate R */
export const pushSteps=arr=>{
 if(T.steps)T.steps.splice(T.i+1,0,...(arr||[]));
};
export const rec=(ctx,e,coll)=>{
 if(e&&ctx)ctx.made.push({id:e.id,coll:coll||"walls"});
 return e;
};
export const dirty=ctx=>{if(ctx)ctx.made.push({id:"~",coll:"__"})};

/* ═══ الإدخال بالكتابة على خطوات الكيانات ═══
   يشترك فيه النقر والكتابة: النقر يعطي hit مباشرة، والكتابة
   تحلّ المعرّف أولاً ثم تمرّ بالمسار نفسه (entVia والرفض). */
const entPt=hit=>{
 const sh=shapeOf(hit);
 if(!sh)return null;
 if(sh.t==="pt")return sh.p.slice();
 if(sh.t==="seg")return [Math.round((sh.a[0]+sh.b[0])/2),
                         Math.round((sh.a[1]+sh.b[1])/2)];
 const P=sh.pts||[];
 if(!P.length)return null;
 return [Math.round(P.reduce((s,p)=>s+p[0],0)/P.length),
         Math.round(P.reduce((s,p)=>s+p[1],0)/P.length)];
};
/* نقطة على مسار الكيان بمسافةٍ من بدايته · السالب من نهايته */
const entAlong=(hit,v)=>{
 const sh=shapeOf(hit);
 if(!sh||sh.t!=="seg")
  throw new Error("المسافة على المسار للجدران والدرج وحدها");
 const dx=sh.b[0]-sh.a[0], dy=sh.b[1]-sh.a[1];
 const L=Math.hypot(dx,dy);
 if(L<1)throw new Error("مسارٌ صفري");
 const t=clamp((v<0)?(L+v):v,0,L);
 return [Math.round(sh.a[0]+dx/L*t), Math.round(sh.a[1]+dy/L*t)];
};
function useEnt(st,hit,p){
 if(st.ent!==1&&hit.k!==st.ent){
  /* entVia: بديلٌ مصرَّح به — النقر على فتحةٍ يعني جدارها */
  const via=st.entVia&&st.entVia[hit.k];
  const alt=via?via(hit):null;
  if(!alt)throw new Error(`انقر على ${st.entName||st.ent}`
   +" — أو اكتب معرّفه");
  hit=alt;
 }
 T.ctx.v[st.k||"ent"]=hit;
 if(st.each)st.each(T.ctx,hit,p||entPt(hit));
 advance();
}

/* ═══ التغذية ═══ */
export function feedPoint(p){
 const st=step();
 if(!st)return false;
 if(st.confirm){
  H.rep("wr","هذه الخطوة تحتاج Enter للتأكيد لا نقرة");
  return false;
 }
 try{
  if(st.ent){
   const hit=H.hit(p[0],p[1]);
   if(!hit)throw new Error("لا عنصر هنا");
   useEnt(st,hit,p);
  }else if(st.ang){
   const b=baseOf();
   if(!b)throw new Error("لا نقطة أساس للزاوية");
   const a=angOf(b,p);
   T.ctx.v[st.k||"a"]=a;
   if(st.each)st.each(T.ctx,a);
   advance();
  }else{
   T.ctx.pts.push(p);
   if(st.k)T.ctx.v[st.k]=p;
   if(st.each)st.each(T.ctx,p);
   advance();
  }
 }catch(e){H.rep("er",e.message); H.prompt(); H.draw(); return false}
  jrAdd(`${(p[0]/1000).toFixed(3)},${(p[1]/1000).toFixed(3)}`);
  H.draw(); return true;
}
/* ═══ الضربة الحرّة ═══
   الخطوة تبقى فعّالة: الرسم يتراكم حتى Enter، ولا يُنشأ شيء منه. */
export function feedStroke(pts){
 const st=step();
 if(!st||!st.freehand)return false;
 const P=(pts||[])
  .filter(p=>p&&isFinite(p[0])&&isFinite(p[1]))
  .map(p=>[Math.round(p[0]),Math.round(p[1])]);
 if(P.length<2)return false;
 try{
  if(st.each)st.each(T.ctx,P);
  T.ctx.n++;
 }catch(e){H.rep("er",e.message)}
 H.prompt(); H.draw();
 return true;
}
export function feedText(s){
 const st=step();
 if(!st)return false;
 s=String(s==null?"":s).trim();
 if(!s)return enter();
 if(st.opts){
  const k=norm(s);
  if(k.length===1&&st.opts[k]){
   try{st.opts[k].run(T.ctx)}
   catch(e){H.rep("er",e.message)}
   H.prompt(); H.draw(); return true;
  }
 }
 try{
  /* خطوة نصٍّ حرّ — تُستعملها أداة التحديد وأدوات التأشير */
  if(st.text){
   if(st.each)st.each(T.ctx,s);
   jrAdd(s);
   advance();
   H.draw(); return true;
  }
  /* ضبط خيار الأداة من سطر الإدخال: t=0.2 · type=ext · chain=0 */
  const oa=/^([A-Za-z][A-Za-z0-9]*)=(.*)$/.exec(s);
  if(oa&&T.def){
   const f=(T.def.opts||[]).find(x=>
    x.k.toLowerCase()===oa[1].toLowerCase());
   if(f){
    let v=oa[2].trim();
    if(f.type==="chk")v=/^(1|on|y|yes|نعم)$/i.test(v)?1:0;
    else if(f.type==="sel"){
     if(!(f.items||[]).some(([iv])=>String(iv)===v))
      throw new Error(`«${v}» ليس من: `
       +(f.items||[]).map(x=>x[0]).join(" · "));
    }else if(f.type==="len"){
     if(v!==""&&Mx(v)==null)throw new Error(`«${v}» ليس طولاً`);
    }else if(f.type==="num"){
     if(v!==""&&Nx(v)==null)throw new Error(`«${v}» ليس رقماً`);
    }
    setOpt(T.def.id,f.k,v);
    jrAdd(`${f.k}=${v}`);
    H.rep("in",`${f.label}: `
     +((f.type==="chk")?(v?"مُشغّل":"مُوقف"):v));
    H.prompt(); H.draw(); return true;
   }
  }
  /* خطوة الزاوية تقبل رقماً مباشراً */
  if(st.ang){
   const v=parseFloat(s);
   if(!isFinite(v))throw new Error(`«${s}» ليست زاوية`);
   T.ctx.v[st.k||"a"]=v;
   jrAdd(String(v));
   if(st.each)st.each(T.ctx,v);
   advance();
   H.draw(); return true;
  }
  /* خطوة كيان: W7 يعني منتصفه · W7@2.4 مسافةٌ على مساره */
  if(st.ent){
   const m=/^([A-Za-z]+\d+)(?:@(-?\d*\.?\d+))?$/.exec(s);
   const hit=m?findById(m[1]):null;
   if(!hit)throw new Error("هذه الخطوة تحتاج نقرة على عنصر — أو "
    +"اكتب معرّفه مثل W7 أو W7@2.4");
   jrAdd(s);
   useEnt(st,hit,(m[2]!=null)?entAlong(hit,M(m[2])):null);
   H.draw(); return true;
  }
  const q=parsePt(s,baseOf(),dirOf());
  if(!q){
   /* اسم أداةٍ صريح وسط أداة ⇒ تبديل لا رفض.
      begin يلغي الحالية ويُثبّت ما صنعتَه قبلها. */
   const d=(s.length>1)?findTool(s):null;
   if(d){histAdd(s); begin(d); return true}
   throw new Error(`«${s}» ليست إحداثياً ولا اسم أداة`);
  }
  if(q.k==="err")throw new Error(q.m);
  if(q.k==="ang"){
   T.lock=q.a;
   H.rep("in",`زاوية مقفلة ${q.a}°`);
   H.prompt(); H.draw(); return true;
  }
  if(q.k==="len")throw new Error("حدّد نقطة الأساس أولاً");
  if(q.k==="dim"){
   const b=baseOf();
   if(!b)throw new Error("المقاس يحتاج نقطة أساس");
   const sx=(T.ghost&&T.ghost[0]<b[0])?-1:1;
   const sy=(T.ghost&&T.ghost[1]<b[1])?-1:1;
   return feedPoint([b[0]+q.w*sx,b[1]+q.d*sy]);
  }
  return feedPoint(q.p);
 }catch(e){H.rep("er",e.message); H.prompt(); H.draw(); return false}
}
export function enter(){
 const st=step();
 if(!st){
  /* الإعادة تحمل الوسيط: بعد «ELEV S» تعيد الجنوبية لا الخيار
     الافتراضي */
  if(T.last){begin(T.last,T.lastArg); return true}
  return false;
 }
 if(st.confirm){
  if(st.each){
   try{st.each(T.ctx)}
   catch(e){H.rep("er",e.message); return false}
  }
  T.i++; jrAdd("."); T.lock=null; T.lenLock=null;
  if(!step())finish(); else {H.prompt(); H.draw()}
  return true;
 }
 if(st.loop){
  if(st.min&&T.ctx.n<st.min){
   H.rep("wr",`تحتاج ${st.min} على الأقل`);
   return false;
  }
  T.i++; jrAdd("."); T.lock=null; T.lenLock=null;
  if(!step())finish(); else {H.prompt(); H.draw()}
  return true;
 }
 H.rep("wr",`${T.def.label}: ${st.p} مطلوب`);
 return false;
}
export function undoStep(){
 jrTaint("تراجع خطوة داخل أداة");
 const ctx=T.ctx;
 if(!ctx)return false;
 if(ctx.made.length){
  const m=ctx.made.pop();
  if(m.coll!=="__"&&Array.isArray(S[m.coll]))
   S[m.coll]=S[m.coll].filter(e=>e.id!==m.id);
  if(ctx.pts.length>1)ctx.pts.pop();
  if(ctx.n>0)ctx.n--;
  touch(); H.rep("in","تراجع خطوة"); H.prompt(); H.draw();
  return true;
 }
 if(ctx.pts.length){ctx.pts.pop(); H.prompt(); H.draw(); return true}
 return false;
}
export function preview(){
 const d=T.def;
 if(!d||!d.prev||!T.ctx)return [];
 try{return d.prev(T.ctx,T.ghost)||[]}
 catch(e){return []}
}
export const pvLine=(a,b,c)=>({t:"l",a,b,c});
export const pvRect=(a,b,c)=>({t:"r",a,b,c});
export const pvBand=(a,b,t,c)=>({t:"b",a,b,w:t,c});
export const pvPoly=(pts,c,cl)=>({t:"pg",pts,c,cl:cl===0?0:1});
export const pvText=(p,s,c)=>({t:"tx",p,s,c});

export function histAdd(s){
 s=String(s||"").trim();
 if(!s)return;
 if(T.hist[T.hist.length-1]===s)return;
 T.hist.push(s);
 if(T.hist.length>60)T.hist.shift();
}

/* ═══ القفل من الإدخال الحركي ═══
   قيمةٌ يكتبها المستخدم فتُقيّد المؤشّر حتى ينقر — مساعدةُ إدخال
   خالصة، لا تعدّل شيئاً بعد وقوع النقطة. */
export function lockLen(mm){
 T.lenLock=(mm==null)?null:Math.max(1,Math.round(mm));
 H.prompt(); H.draw();
 return T.lenLock;
}
export function lockAng(a){
 T.lock=(a==null)?null:a;
 H.prompt(); H.draw();
 return T.lock;
}
```

<a id="f-js-tools-section-js"></a>

---

## `js/tools/section.js`

```javascript
/* ═══ أداة المقطع ═══
   نقطتان لخطّ القطع، ومعاينةٌ حيّة: الخطّ ونطاقُه وعددُ ما سيُقطَع.
   والأداة تبقى فعّالة بعد كل مقطعٍ حتى Esc (restart) كبقيّة الأدوات.

   ولا تعدّل شيئاً: لا rec ولا dirty ولا edit — فctx.made يبقى
   فارغاً، ولا تُدفَع خطوةُ تاريخ. والتصدير خيارٌ لا أثرٌ لازم:
   fmt="none" افتراضاً. */
import {S} from "../core/state.js";
import {sectCmd,sectSay,sectWalls,sectRunSay,MINCUT,CUT}
 from "../core/section.js";
import {sectFile} from "../io/sect.js";
import {dl} from "../io/project.js";
import {m2} from "../core/units.js";
import {defTool,H,ov,ovLen,ovOn,pvLine,pvBand,pvText}
 from "./registry.js";

const tolOf=()=>Math.max(0,ovLen("section","tol")||CUT);

/* كائنٌ واحد يقرؤه اثنان، كلٌّ يأخذ مفاتيحه: section() تقرأ
   tol/back/unfold وتتجاهل mark، وsectFile تقرأ mark وتتجاهل
   الثلاثة. فلا تعارض، ولا قارئَ خيارٍ ثانٍ يتخلّف عن الأول. */
const optsOf=()=>({tol:tolOf(),
 back:ovOn("section","back")?1:0,
 unfold:ovOn("section","unfold")?1:0,
 mark:String(ov("section","mark")||"").trim()});

export function runSect(a,b){
 const o=optsOf();
 const e=sectCmd(a,b,o);                 /* يرمي إن لا جدار يعبره */
 H.rep("ok",sectSay(e));
 e.runs.forEach(r=>H.rep("in",sectRunSay(r)));
 e.warn.forEach(w=>H.rep("wr",w.msg));

 const fmt=String(ov("section","fmt")||"none");
 if(fmt==="none")return e;
 const F=sectFile(fmt,e,o);              /* mark يُقرأ هنا */
 F.notes.forEach(s=>H.rep("wr",s));
 if(F.bad)H.rep("wr",`${F.bad} محرفاً تعذّر ترميزه`);
 if(ovOn("section","save")){
  const size=dl(F.name,(F.txt!=null)?F.txt:F.bytes,F.mime);
  H.rep("ok",`نُزّل ${F.name}`+(size?` · ${size} بايت`:""));
 }else{
  H.rep("in",`${F.name} جاهز — شغّل «نزّل الملفّ» ليُكتَب`);
 }
 return e;
}
/* الرمي يُلتقَط هنا لا في مصيدة الخطوة: لو صعد لتوقّف advance فبقيت
   نقطتان في الخطوة، وصارت الثالثة تُضاف إليهما. */
function safeRun(a,b){
 try{return runSect(a,b)}
 catch(x){H.rep("er",x.message); return null}
}
defTool({
 id:"section", alias:"مقطع قطاع sect",
 label:"مقطع",
 hint:"نقطتان لخطّ القطع · يُقطَع ما يعبره الخطّ بنطاقه — "
  +"ولا يتحدّث تلقائياً بعدها",
 opts:[
  {k:"tol",    label:"نطاق القطع", type:"len", def:"0.30"},
  {k:"back",   label:"النظر من الجهة الأخرى", type:"chk", def:0},
  {k:"unfold", label:"فرد تراكميّ بدل الموضع الحقيقي",
   type:"chk", def:0},
  /* بلا type: يسقط على الفرع الافتراضي في ctlOf فيُرسَم حقلاً
     نصّياً، وfeedText لا يُصدِّق عليه شيئاً — فيقبل «A-A» كما هو.
     وsafeName في io/sect.js يُطهّره من محارف المسار قبل الكتابة،
     فـ«A/A» تصير «A_A» ولا تُخرِج اسماً معطوباً. */
  {k:"mark",   label:"رمز المقطع (A-A)", def:""},
  {k:"fmt",    label:"التصدير", type:"sel", def:"none", items:[
   ["none","لا شيء"],["svg","SVG"],["dxf","DXF"],["pdf","PDF"]]},
  {k:"save",   label:"نزّل الملفّ", type:"chk", def:1}],
 steps:[
  {p:"نقطة خطّ القطع الأولى", k:"a"},
  {p:"النقطة الثانية", k:"b", restart:1,
   each(ctx,p){safeRun(ctx.v.a,p)}}],
 /* ═══ المعاينة ═══ */
 prev(ctx,g){
  const a=ctx.v.a||ctx.pts[0];
  if(!a||!g)return [];
  const tol=tolOf();
  const out=[pvLine(a,g), pvBand(a,g,Math.max(2,tol*2))];
  const L=Math.hypot(g[0]-a[0],g[1]-a[1]);
  let say=`${m2(L)} م`;
  if(L<MINCUT)say+=" · أقصر من الحدّ";
  else{
   let n=0, m=0;
   try{
    const q=sectWalls(a,g,{tol,back:ovOn("section","back")?1:0});
    n=q.list.length; m=q.miss.length;
   }catch(e){}
   say+=` · ${n} جدار`+(m?` · ${m} قارب`:"");
  }
  out.push(pvText([Math.round((a[0]+g[0])/2),
   Math.round((a[1]+g[1])/2)], say));
  return out;
 }});
```

<a id="f-js-tools-sketch-js"></a>

---

## `js/tools/sketch.js`

```javascript
/* ═══ أداة الخربشة ═══
   اسحب لترسم بحرّية كما على ورقة. Enter ينهي الرسم فتُعرَض الخطّة:
   مساراتٌ بأطوالها وانحرافها. Enter الثانية تُنشئ الجدران عبر
   addWall نفسها — فتنالها قيوده كلّها، وما رُفض يُذكَر بسببه.
   Esc قبل التأكيد لا يترك أثراً، والتأكيد خطوةُ تراجعٍ واحدة.

   لا شيء يُنشأ من ضربةٍ وحدها: الخربشة اقتراح، والجدار أمرك. */
import {S} from "../core/state.js";
import {m2,m3,M,clamp,arrow} from "../core/units.js";
import {addWall,ALIGN,MINW} from "../core/walls.js";
import {trace,scalePlan,snapPts,planSay,cornerSay} from "../core/trace.js";
import {defTool,H,T,rec,step,ov,ovLen,ovNum,ovOn,
        pvLine,pvBand,pvPoly,pvText} from "./registry.js";

const GRN="#5cd98e", YEL="#ffd06b", RED="#ff6f6f", DIM="#3d4a58";
const TY=[["int","داخلي"],["ext","خارجي"],["low","سترة"]];
const AL=[["c","مركزي"],["l","الوجه الأيسر"],["r","الوجه الأيمن"]];

/* مفتاح الكاش: الضربات + كل خيارٍ يؤثّر في الاستنباط. تغييرُ حقلٍ
   في الشريط يُعيد الاستنباط فوراً وتراه قبل أن تؤكّد. */
const keyOf=ctx=>[
 (ctx.v.st||[]).length,
 (ctx.v.st||[]).reduce((s,p)=>s+p.length,0),
 ov("sketch","eps"), ov("sketch","angTol"),
 ov("sketch","minSeg"), ov("sketch","nodeTol"),
 ov("sketch","cal"), ov("sketch","grid")].join("|");

function planOf(ctx){
 const k=keyOf(ctx);
 if(ctx.v.plan&&ctx.v.pk===k)return ctx.v.plan;
 const O={};
 const e=ovLen("sketch","eps");     if(e)O.eps=e;
 const m=ovLen("sketch","minSeg");  if(m)O.minSeg=m;
 const n=ovLen("sketch","nodeTol"); if(n)O.nodeTol=n;
 const a=ovNum("sketch","angTol");  if(a)O.angTol=clamp(a,1,44);
 const P=trace(ctx.v.st||[],O);
 /* المعايرة: طولٌ تعرفه لأطول مسار — تحويلٌ متشابه واحد */
 const cal=ovLen("sketch","cal");
 P.cal=null;
 if(cal>0&&P.segs.length){
  const was=P.segs[0].L;
  if(was>1){
   const kk=cal/was;
   scalePlan(P.segs,kk);
   P.cal={k:kk,was,now:cal};
  }
 }
 P.shift=ovOn("sketch","grid")
  ? snapPts(P.segs,Math.max(1,S.meta.snap)) : 0;
 ctx.v.plan=P; ctx.v.pk=k;
 return P;
}
defTool({
 id:"sketch", alias:"sk خربشه مسوده ارسم_بيدك", label:"خربشة",
 hint:"اسحب لترسم · Enter يعرض الخطّة · Enter يؤكّد · U يمسح آخر ضربة",
 opts:[
  {k:"t",      label:"السماكة م", type:"len", def:"0.2"},
  {k:"type",   label:"النوع",     type:"sel", items:TY, def:"ext"},
  {k:"align",  label:"المسار على",type:"sel", items:AL, def:"c"},
  {k:"cal",    label:"طول أطول ضلع م", type:"len", def:"",
   hint:"فارغ = بمقياس خربشتك · اكتب طولاً تعرفه لتُعايَر الخطّة"},
  {k:"angTol", label:"أقصى قصٍّ زاويّ °", type:"num", def:18,
   hint:"ما زاد انحرافه يبقى حرّاً ويُبلَّغ"},
  {k:"eps",    label:"تفاوت التبسيط م", type:"len", def:"",
   hint:"فارغ = يُشتقّ من حجم خربشتك"},
  {k:"minSeg", label:"أقصر مسار م", type:"len", def:""},
  {k:"nodeTol",label:"تقارب الأطراف م", type:"len", def:""},
  {k:"grid",   label:"قرّب إلى خطوة الالتقاط", type:"chk", def:0,
   hint:"يُبلَّغ أكبر إزاحة"}],
 steps:[
  {p:"اخربش المخطّط ثم Enter (U يمسح آخر ضربة)",
   freehand:1, loop:1, min:1,
   opts:{
    u:{n:"امسح آخر ضربة",run(ctx){
     if(!(ctx.v.st||[]).length)throw new Error("لا ضربة تُمسَح");
     ctx.v.st.pop(); ctx.v.plan=null; ctx.v.n2=0;
     if(ctx.n>0)ctx.n--;
     H.rep("in",`بقيت ${ctx.v.st.length} ضربة`);
    }},
    c:{n:"امسح الكلّ",run(ctx){
     ctx.v.st=[]; ctx.v.plan=null; ctx.n=0;
     H.rep("in","مُسحت الخربشة");
    }}},
   each(ctx,P){
    ctx.v.st=(ctx.v.st||[]).concat([P]);
    ctx.v.plan=null;
   }},
  {p:"Enter يؤكّد تحويل الخطّة إلى جدران · S يُكمل الخربشة · Esc يلغي",
   confirm:1,
   opts:{s:{n:"أكمل الخربشة",run(){T.i=0; H.rep("in","أكمل الرسم")}}},
   each(ctx){
    const P=planOf(ctx);
    if(!P.segs.length)
     throw new Error("لا مسار في الخطّة — اخربش أطول، أو صغّر "
      +"«أقصر مسار»");
    const t=ovLen("sketch","t"), ty=ov("sketch","type"),
          al=ov("sketch","align");
    let n=0;
    const ref=[];
    P.segs.forEach(s=>{
     try{
      rec(ctx,addWall(s.a,s.b,t,ty,al),"walls");
      n++;
     }catch(e){ref.push(`${m3(s.L)} م: ${e.message}`)}
    });
    if(!n)throw new Error("لم يُقبَل مسارٌ واحد — "
     +(ref[0]||"راجع الخيارات"));
    H.rep("ok",`أُنشئ ${n} جدار من ${P.segs.length} مساراً · `
     +`${m2(t)} م ${ALIGN[al]}`);
    H.rep("in",cornerSay(P)
     +` · الخربشة لم تُحفَظ: الجدران وحدها صارت بيانات`);
    ref.slice(0,6).forEach(m=>H.rep("wr","  رُفض "+m));
    if(ref.length>6)H.rep("in",`  … و ${ref.length-6} رفضاً آخر`);
    if(P.stat.free)H.rep("wr",`${P.stat.free} مساراً بقي حرّاً — `
     +`زاويته كما رسمتها. استعمل «لحم» أو «دوران» إن شئت.`);
   }}],
 prev(ctx,g){
  const out=[];
  (ctx.v.st||[]).forEach(P=>{
   if(P.length>1)out.push(pvPoly(P,DIM,0));
  });
  if(!(ctx.v.st||[]).length)return out;
  const P=planOf(ctx);
  const st=step();
  /* تقريرٌ مرّةً واحدة عند دخول خطوة التأكيد أو تغيّر الخطّة */
  if(st&&st.confirm&&ctx.v.said!==ctx.v.pk){
   ctx.v.said=ctx.v.pk;
   H.rep(P.segs.length?"wr":"er","الخطّة: "+planSay(P));
   H.rep("in","  "+cornerSay(P));
   if(P.cal)H.rep("in",`  معايرة ×${P.cal.k.toFixed(5)} · `
    +`${arrow(m3(P.cal.was),m3(P.cal.now))} م`);
   if(P.shift)H.rep("in",`  التقريب إلى الشبكة أزاح `
    +`${m3(P.shift)} م على الأكثر`);
   P.segs.slice(0,14).forEach((s,i)=>H.rep("in",
    `  ${i+1}· ${m3(s.L)} م · ${s.ang.toFixed(1)}°`
    +(s.snapped?"":` · حرّ (انحراف ${s.dev.toFixed(1)}°)`)
    +(s.merged>1?` · دُمج ${s.merged}`:"")));
   if(P.segs.length>14)
    H.rep("in",`  … و ${P.segs.length-14} مساراً آخر`);
   H.rep("wr","  Enter يؤكّد · Esc يلغي بلا أثر");
  }
  const t=ovLen("sketch","t")||150;
  P.segs.forEach(s=>{
   out.push(pvBand(s.a,s.b,t,s.snapped?GRN:RED));
   out.push(pvLine(s.a,s.b,s.snapped?YEL:RED));
  });
  P.segs.slice(0,24).forEach(s=>{
   if(s.L<t*3)return;
   out.push(pvText([(s.a[0]+s.b[0])/2,(s.a[1]+s.b[1])/2],
    m3(s.L)+(s.snapped?"":" ✗"), s.snapped?GRN:RED));
  });
  return out;
 }});
```

---

# الإدخال والإخراج (`js/io/`)

<a id="f-js-io-boq-js"></a>

---

