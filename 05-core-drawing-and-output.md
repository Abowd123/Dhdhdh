# CivilDraft — Core: Drawing, Rendering & Output

> الرسم، المقاطع، الواجهات، الإسقاط ثلاثي الأبعاد، الأسعار، جداول الكميات.

**عدد الملفات:** 11

---

## `js/core/render.js`

```javascript
/* ═══ المشهد ═══
   قائمةُ أوّلياتٍ واحدة يقرأها الرسم والتصدير والفاحص، فلا يفترق
   ما يُرى عمّا يُصدَّر. ولا رسمَ هنا: هذا الملفّ لا يعرف قماشاً ولا
   سياقاً ولا لوناً — الهيئة في io/style.js.

   ═══ ولا بنّاءَ أوّلياتٍ هنا كذلك ═══
   الرمزُ يسكن مع كيانه: openPrims في opens.js وcolPrims في cols.js
   وstPrims في stairs.js وgridPrims في dims.js. وبناءُ نسخةٍ ثانية
   هنا كان يُنتِج أربع خسائر: عمودٌ يُرسَم حدُّه فوق صمته المدمَج،
   وبابٌ مزدوجٌ بمصراعٍ واحد، وثلاثةُ أنواعِ فتحاتٍ على طبقةٍ تخالف
   طبقةَ كيانها (فتُخفى ولا تختفي)، وعلاماتُ العطب لا تُرسَم أصلاً.

   ═══ الترتيب جدولٌ مُعلَن ═══
   كان ترتيبُ الطلاء ضمنياً في تسلسل الأسطر داخل scene: لا يُقرأ
   ولا يُختبَر ولا يُكاش جزءاً جزءاً. وصيرورتُه بياناتٍ (BANDS) هي
   ما يجعل التجزيء ممكناً بلا تغييرٍ في ما يُطلى — فالمناطق تحت
   الجدران وهي «عرض» والجدران «هندسة»، والتجزيء بالنوع وحده
   يقلبهما.

   ═══ ولكلّ نطاقٍ مفتاحه ═══
   سحبُ بُعدٍ كان يعيد بناء أوراق الأبواب ونقوش الهاشور ونتوءات
   الدرج وستّين ألف أوّليةٍ مرجعية — في كل إطار. وبعد اليوم يُعاد
   بناء ما تغيّر مفتاحُه وحده.

   والمفاتيح آمنةٌ لأن touch يُقدّم VER.g (الدفعة ٨أ): من نسي أن
   يُعلن نوع تعديله يخسر أداءً لا صحّة. ومع ذلك أُدرِجت خياراتُ
   العرض في المفتاح صريحاً (optKey) فلا يتّكل النطاق على افتراض. */
import {S,VER,txtH,refVersion} from "./state.js";
import {bboxOf,bboxUnion,bandPoly,rectPoly,circPoly,polyBool,
        cleanRing,PERF as GP} from "./geom.js";
import {band,centerLine,dir,isLow,lowH,wallLen,isArc} from "./walls.js";
import {span,depOf,badOpens,badPrims,openPrims,
        okOf} from "./opens.js";
import {areaPrims,staleCount} from "./areas.js";
import {dimPrims,chainPrims,annoPrims,gridPrims,looseDims,
        isOverridden} from "./dims.js";
import {fixPrims} from "./fixt.js";
import {stPrims} from "./stairs.js";
import {colPrims} from "./cols.js";
import {refPrims} from "./ref.js";
import {sheetPrims} from "./sheet.js";
import {explode} from "./blocks.js";
import {vis,plots,hiddenCount,layVer} from "./layers.js";
import {bump} from "./perf.js";

const R=v=>Math.round(v);

/* ═══ صندوق الأوّليات ═══
   النصّ يُقدَّر عرضاً: قياسُه الحقيقيّ يحتاج قماشاً، والقماش خارج
   هذا الملفّ. والتقدير يزيد ولا ينقص، فلا يُقصّ نصٌّ في التصدير. */
export function primsBBox(list){
 const P=[];
 (list||[]).forEach(g=>{
  if(!g)return;
  if(g.t==="line"){P.push(g.a,g.b); return}
  if(g.t==="poly"){(g.pts||[]).forEach(p=>P.push(p)); return}
  if(g.t==="fill"){(g.ring||[]).forEach(p=>P.push(p)); return}
  if(g.t==="hatch"){
   (g.loops||[]).forEach(l=>(l||[]).forEach(p=>P.push(p)));
   return;
  }
  if(g.t==="arc"){
   const r=Math.abs(g.r)||0;
   P.push([g.cx-r,g.cy-r],[g.cx+r,g.cy+r]);
   return;
  }
  if(g.t==="text"){
   const w=Math.max(1,String(g.s==null?"":g.s).length)*g.h*0.62;
   P.push([g.x-w,g.y-g.h],[g.x+w,g.y+g.h]);
  }
 });
 return bboxOf(P);
}
/* ═══ التصفية ═══
   المخفيّ ليس في المشهد: لا يُرسَم ولا يُصدَّر ولا يدخل الصندوق.
   وما لا يُطبَع يبقى — يُرى على الشاشة ويُستثنى عند الهيئة. */
export const filterPrims=list=>
 (list||[]).filter(g=>g&&vis(g.L||"0"));

/* ═══ المحاور المدمجة ═══
   جداران متّصلان على استقامةٍ واحدة بسماكةٍ واحدة يصيران محوراً
   واحداً، فلا يظهر خطُّ الوصل بينهما. والدمج عرضٌ لا تعديل: البيانات
   تبقى جدارَين، ويُلغى بخيار S.opt.joins.

   ويعيد lo/hi على المحور لا نقطتين وحدهما: طرحُ الفتحات يقع على
   المحور المدمج، فيحتاج إحداثياً واحداً يُقاس عليه. */
const centerPt=(g,s)=>[R(g.u[0]*s+g.n[0]*g.off), R(g.u[1]*s+g.n[1]*g.off)];
/* نقطتا جسم المحور عند طرفٍ ما — على بُعد نصف السماكة يميناً ويساراً
   من الطرف، على المحور الحقيقي (بعد المحاذاة). */
const endCorners=(g,end)=>{
 const c=centerPt(g,(end==="lo")?g.lo:g.hi), h=g.t/2;
 return [[c[0]+g.n[0]*h,c[1]+g.n[1]*h],[c[0]-g.n[0]*h,c[1]-g.n[1]*h]];
};
/* ═══ لحمُ الأركان ═══
   محوران يلتقيان بزاويةٍ (لا استقامة) عند طرفَي مسارهما الأصليَّين
   يترك اتحادُ جسميهما ثلمةً في الركن الخارجي: كلُّ جسمٍ يقف عند
   الطرف، فلا يغطّي أحدُهما نتوء الركن الذي وراء الآخر. وحجمُ الثلمة
   يتغيَّر بالمحاذاة (مركزية أو على وجه)، فلا يكفيها رقمٌ ثابت.

   والحلّ مدُّ محور كلّ جدارٍ عند طرفه الملتقي بآخر حتى يبلغ أبعدَ
   نقطةٍ من جسم ذلك الآخر على امتداد محوره هو — فيغطّي جسمه الثلمة
   مهما كانت المحاذاة، والاتحاد بعدها يمتصّ أيّ تراكبٍ زائد. ولمّا
   كان المدُّ يزيد لا ينقص، فتكرارُه من الطرفين معاً بلا ضرر. */
function weldCorners(out){
 const key=p=>R(p[0])+","+R(p[1]);
 const M=new Map();
 out.forEach(g=>{
  [["lo",g.rawLo],["hi",g.rawHi]].forEach(([end,p])=>{
   if(!p)return;
   const k=key(p);
   let a=M.get(k);
   if(!a){a=[]; M.set(k,a)}
   a.push({g,end});
  });
 });
 const ext=new Map();
 M.forEach(list=>{
  if(new Set(list.map(e=>e.g)).size<2)return;
  list.forEach(({g,end})=>{
   list.forEach(o=>{
    if(o.g===g)return;
    endCorners(o.g,o.end).forEach(c=>{
     const proj=c[0]*g.u[0]+c[1]*g.u[1];
     const cur=ext.get(g)||{};
     if(end==="lo")cur.lo=Math.min(cur.lo==null?g.lo:cur.lo,proj);
     else cur.hi=Math.max(cur.hi==null?g.hi:cur.hi,proj);
     ext.set(g,cur);
    });
   });
  });
 });
 ext.forEach((v,g)=>{
  if(v.lo!=null)g.lo=Math.min(g.lo,v.lo);
  if(v.hi!=null)g.hi=Math.max(g.hi,v.hi);
 });
 out.forEach(g=>{ g.a=centerPt(g,g.lo); g.b=centerPt(g,g.hi); });
}
export function centers(walls,joins){
 const out=[];
 const mk=w=>{
  const c=centerLine(w), d=dir(w);
  if(!c||!d)return null;
  let ang=Math.atan2(d.uy,d.ux);
  if(ang<0)ang+=Math.PI;
  const ux=Math.cos(ang), uy=Math.sin(ang);
  const nx=-uy, ny=ux;
  const s0=c.a[0]*ux+c.a[1]*uy, s1=c.b[0]*ux+c.b[1]*uy;
  const lo0=(s0<=s1), rawLo=lo0?w.a:w.b, rawHi=lo0?w.b:w.a;
  return {ang,u:[ux,uy],n:[nx,ny],
   off:c.a[0]*nx+c.a[1]*ny,
   lo:Math.min(s0,s1), hi:Math.max(s0,s1), t:w.t, ws:[w],
   rawLo,rawHi};
 };
 const pt=centerPt;
 if(!joins){
  (walls||[]).forEach(w=>{
   const g=mk(w);
   if(g){g.a=pt(g,g.lo); g.b=pt(g,g.hi); out.push(g)}
  });
  weldCorners(out);
  return out;
 }
 const G=new Map();
 (walls||[]).forEach(w=>{
  const g=mk(w);
  if(!g)return;
  const k=`${Math.round(g.ang*1e4)}|${Math.round(g.off)}|${g.t}`;
  let a=G.get(k);
  if(!a){a=[]; G.set(k,a)}
  a.push(g);
 });
 G.forEach(list=>{
  list.sort((p,q)=>p.lo-q.lo);
  let cur=null;
  const flush=()=>{
   if(!cur)return;
   cur.a=pt(cur,cur.lo); cur.b=pt(cur,cur.hi);
   out.push(cur); cur=null;
  };
  list.forEach(g=>{
   if(cur&&g.lo<=cur.hi+1){
    if(g.hi>cur.hi)cur.rawHi=g.rawHi;
    cur.hi=Math.max(cur.hi,g.hi);
    cur.ws.push(g.ws[0]);
    return;
   }
   flush();
   cur=g;
  });
  flush();
 });
 weldCorners(out);
 return out;
}
/* الفتحات العابرة تُطرَح من الأجسام — والكوّة لا تعبر فلا تُطرَح.
   وخريطةٌ واحدة لكل نداء: المسحُ لكل جدارٍ يجعلها جدرانٌ × فتحات. */
function openMap(){
 const M=new Map();
 S.opens.forEach(o=>{
  if(o.kind==="niche")return;
  let a=M.get(o.wall);
  if(!a){a=[]; M.set(o.wall,a)}
  a.push(o);
 });
 return M;
}
function solidRuns(g,OM){
 const cuts=[];
 g.ws.forEach(w=>{
  const list=OM.get(w.id);
  if(!list||!list.length)return;
  const c=centerLine(w);
  if(!c)return;
  const s0=c.a[0]*g.u[0]+c.a[1]*g.u[1];
  const s1=c.b[0]*g.u[0]+c.b[1]*g.u[1];
  const sg=(s1>=s0)?1:-1;
  list.forEach(o=>{
   const [a,b]=span(o);
   const p=s0+sg*a, q=s0+sg*b;
   cuts.push([Math.min(p,q),Math.max(p,q)]);
  });
 });
 if(!cuts.length)return [[g.lo,g.hi]];
 cuts.sort((a,b)=>a[0]-b[0]);
 const runs=[];
 let at=g.lo;
 cuts.forEach(([a,b])=>{
  if(b<=at)return;
  if(a>at+1)runs.push([at,Math.min(a,g.hi)]);
  at=Math.max(at,b);
 });
 if(at<g.hi-1)runs.push([at,g.hi]);
 return runs.filter(([a,b])=>b-a>1);
}
/* ═══ الأجسام ═══
   forLoops=1: بلا طرح الفتحات — الباب لا يوسّع الغرفة، ولو طُرح
   لتسرّبت الحلقة من الفتحة. */
export function bodyOf(walls,cols,forLoops){
 const polys=[];
 const OM=forLoops?null:openMap();
 /* القوسية تُبنى مضلّعاً مباشرةً (band) فتدخل الاتحاد كسائر
    الأجسام؛ والمستقيمة تمرّ بالمحاور فتنال الدمج وطرح الفتحات. */
 const straight=[], arcs=[];
 (walls||[]).forEach(w=>{(isArc(w)?arcs:straight).push(w)});
 arcs.forEach(w=>{
  const p=band(w);
  if(p&&p.length>2)polys.push(p);
 });
 centers(straight,!!+S.opt.joins).forEach(g=>{
  const runs=forLoops?[[g.lo,g.hi]]:solidRuns(g,OM);
  runs.forEach(([a,b])=>{
   const p1=[g.u[0]*a+g.n[0]*g.off, g.u[1]*a+g.n[1]*g.off];
   const p2=[g.u[0]*b+g.n[0]*g.off, g.u[1]*b+g.n[1]*g.off];
   const bp=bandPoly(p1[0],p1[1],p2[0],p2[1],g.t);
   if(bp)polys.push(bp);
  });
 });
 (cols||[]).forEach(c=>{
  const p=(c.kind==="circ")
   ? circPoly(c.x,c.y,c.w/2,32)
   : rectPoly(c.x,c.y,c.w,c.h,c.rot);
  if(p)polys.push(p);
 });
 if(!polys.length)return [];
 return polyBool(polys,{eps:1,minArea:400});
}
/* ═══ حلقات المناطق ═══
   تتجاهل الإخفاء تماماً — الإخفاء عرضٌ لا حذف. والأعمدة تدخلها
   ولو عُرضت مستقلّة: العمود مانعٌ فعليّ.
   وما لم يُخَط يُقرأ فرقاً في عدّاد geom لا من المخرَج: bodyOf
   يُرشِّح الحلقات فتُفقَد خاصّية open عليها. */
const optKey=()=>`${S.opt.fill}|${+S.opt.joins}|${+S.opt.colSolo}`;
const gKey =()=>`${VER.g}|${optKey()}`;
const goKey=()=>`${gKey()}|${VER.o}`;
const nKey =()=>String(VER.n);

let RL=null, RLK="", RLO=0, RLW=0, RLAt=null;
export function regionLoops(){
 const k=`${VER.g}|${+S.opt.joins}`;
 if(RLK===k&&RL)return RL;
 const o0=GP.open, w0=GP.weld;
 RL=bodyOf(S.walls, S.cols, true);
 RLK=k;
 RLO=GP.open-o0; RLW=GP.weld-w0;
 /* الموضعُ يُقرأ بعد الفرق: openAt آخرُ ما وقع، فإن لم يقع في
    ندائنا هذا فهو من نداءٍ سابقٍ ويدلّ على غير مكانه. */
 RLAt=RLO?GP.openAt:null;
 bump("loops");
 return RL;
}
export const loopOpen  =()=>RLO;
export const loopOpenAt=()=>RLAt;
export const loopWeld  =()=>RLW;

let BC=null, BCK="", BCO=0, BCW=0, BCAt=null;
function bodies(){
 const k=goKey();
 if(BC&&BCK===k)return BC;
 const solo=!!+S.opt.colSolo;
 const cut=S.walls.filter(w=>!isLow(w));
 const low=S.walls.filter(isLow);
 const o0=GP.open, w0=GP.weld;
 BC={solid:bodyOf(cut, solo?null:S.cols, false),
     lows :bodyOf(low, null, false)};
 BCK=k;
 BCO=GP.open-o0; BCW=GP.weld-w0;
 BCAt=BCO?GP.openAt:null;
 bump("bodies");
 return BC;
}
export const bodyOpen  =()=>BCO;
export const bodyWeld  =()=>BCW;
export const bodyStats=()=>({key:BCK,loops:RLK,
 open:BCO+RLO, weld:BCW+RLW, at:BCAt||RLAt});

/* ═══ بنّاؤو النطاقات ═══
   كلٌّ يعيد أوّلياتٍ خامّاً بلا تصفية: التصفية طبقةٌ فوقها بمفتاحٍ
   آخر (نسخة الطبقات)، فإخفاءُ طبقةٍ لا يعيد بناء ستّين ألف أوّلية. */
let RE_last=null, RE_ep=0;
const refKey=()=>{
 if(S.ref.ents!==RE_last){RE_last=S.ref.ents; RE_ep++}
 const t=S.ref.tr||{};
 return `${RE_ep}|${refVersion()}|${t.k},${t.rot},${t.dx},${t.dy}`
  +`|${Object.keys(S.ref.off||{}).sort().join(",")}`;
};
const refBand=()=>refPrims();

const areaBand=()=>{
 const out=[], h=txtH();
 S.areas.forEach(a=>areaPrims(a,h).forEach(g=>out.push(g)));
 return out;
};
const HP={hatch:"ANSI31", solid:"SOLID"};
const hatchBand=()=>{
 if(S.opt.fill==="none")return [];
 const pat=HP[S.opt.fill]||"ANSI31";
 const sc=Math.max(8,txtH()*1.1);
 const B2=bodies(), out=[];
 if(B2.solid.length)
  out.push({t:"hatch",L:"A-WALL-PATT",loops:B2.solid,pat,sc});
 if(B2.lows.length)
  out.push({t:"hatch",L:"A-WALL-LOW",loops:B2.lows,pat,sc});
 /* ولا هاشورَ للعمود المستقلّ هنا: colPrims يُخرِجه بمادّته —
    ANSI31 للحديد وSOLID للخرسانة — وعلى طبقته A-COLS. */
 return out;
};
const bodyBand=()=>{
 const B2=bodies(), out=[];
 B2.solid.forEach(r=>out.push({t:"poly",L:"A-WALL",pts:r,cl:1}));
 B2.lows.forEach(r=>out.push({t:"poly",L:"A-WALL-LOW",pts:r,cl:1}));
 return out;
};
/* ═══ الفتحات ═══
   الرمزُ من opens.js: البابُ المزدوج مصراعان، والشبّاكُ قوائمُه
   بعدد مصاريعه، وجانبا الفتحة العابرة من حدود الجسم نفسه (الطرحُ
   يقطع الشريط فتظهر أوجهُه) — فلا خطٌّ مزدوج.
   والكوّةُ وحدها تُرسَم هنا: openMap يتخطّاها فلا تُطرَح من الجسم،
   وحدُّها يُرسَم على طبقتها هي لا على A-WALL — فإخفاءُ طبقتها
   يُخفيها، وذلك عقد «المخفيّ ليس في المشهد». */
const openBand=()=>{
 const out=[];
 const WM=new Map(S.walls.map(w=>[w.id,w]));
 S.opens.forEach(o=>{
  const w=WM.get(o.wall);
  if(!w)return;
  if(o.kind==="niche"){
   const c=centerLine(w), d=dir(w);
   if(!c||!d)return;
   const [a,b]=span(o);
   const hw=w.t/2, dp=depOf(o,w.t);
   const sg=(o.face==="r")?-1:1;
   const at=(s,off)=>[R(c.a[0]+d.ux*s+d.nx*off),
                      R(c.a[1]+d.uy*s+d.ny*off)];
   out.push({t:"poly",L:okOf(o.kind).lay,cl:1,oid:o.id,pts:[
    at(a,hw*sg),at(b,hw*sg),
    at(b,hw*sg-dp*sg),at(a,hw*sg-dp*sg)]});
   return;
  }
  openPrims(o).forEach(g=>{g.oid=o.id; out.push(g)});
 });
 return out;
};
/* علاماتُ العطب: تُرسَم على __BAD فتُرى ولو أُخفيت طبقةُ الفتحة —
   تقريرٌ عن حالتك لا زينة. وplots("__BAD") كاذبةٌ فلا تُصدَّر. */
const badBand=()=>{
 const out=[];
 badOpens().forEach(o=>badPrims(o).forEach(g=>out.push(g)));
 return out;
};
/* العمودُ المدمَج لا حدَّ خاصّ له — حدُّه من الاتحاد نفسه، ويبقى
   صليبُ مركزه ووسمُه. والمستقلُّ يُرسَم محيطاً وهاشوراً، والدائريُّ
   قوساً حقيقياً فيُصدَّر CIRCLE لا مضلّعاً بـ٣٢ ضلعاً. */
const colBand=()=>{
 const solo=!!+S.opt.colSolo;
 const out=[];
 S.cols.forEach(c=>colPrims(c,solo).forEach(g=>out.push(g)));
 return out;
};
const fixtBand=()=>{
 const out=[];
 S.fixt.forEach(f=>fixPrims(f).forEach(g=>out.push(g)));
 return out;
};
const stairBand=()=>{
 const out=[];
 S.stairs.forEach(s=>stPrims(s).forEach(g=>out.push(g)));
 return out;
};
/* المحاورُ تمتدّ على صندوق الهندسة، فتقرؤه من السياق. وليست geo
   بقصد: لو دخلت الصندوق لنمت الورقةُ به فنمت المحاورُ معها. */
const axisBand=cx=>gridPrims(cx.B);
const dimBand=()=>{
 const out=[];
 S.dims.forEach(d=>dimPrims(d).forEach(g=>out.push(g)));
 S.chains.forEach(c=>chainPrims(c).forEach(g=>out.push(g)));
 return out;
};
const annoBand=()=>{
 const out=[];
 S.anno.forEach(a=>annoPrims(a).forEach(g=>out.push(g)));
 return out;
};
const blockBand=()=>{
 const out=[];
 (S.blocks||[]).forEach(b=>{
  explode(b).forEach(g=>{
   if(g.t==="line")
    out.push({t:"line",L:g.layer||"0",a:g.a,b:g.b,bid:b.id});
   else if(g.t==="pline")
    out.push({t:"poly",L:g.layer||"0",pts:g.pts,cl:g.closed?1:0,bid:b.id});
  });
 });
 return out;
};
/* الورقة تستند إلى صندوق الهندسة، فتُبنى آخراً ويُمرَّر إليها.
   وتُوسَم sheet:1 فيُعرَف حبرُها من ورقها في القياس والتقارير. */
const sheetBand=cx=>{
 if(!+S.sheet.on)return [];
 const out=sheetPrims(cx.B)||[];
 out.forEach(g=>{g.sheet=1});
 return out;
};
/* ═══ جدول النطاقات ═══
   الترتيب ترتيبُ الطلاء: الأوّل تحت، والأخير فوق.
   geo=1   يدخل صندوق الهندسة (عليه تُبنى الورقة)
   sheet=1 يُستثنى من صندوق الحبر
   diag=1  تشخيصٌ لا يُطبَع ولا يدخل صندوق الحبر — فلا يُبلَّغ
           بتجاوزٍ للورقة سببُه علامةُ تحذير */
const BANDS=[
 {n:"ref",   key:refKey, f:refBand},
 {n:"area",  key:nKey,   f:areaBand,  geo:1},
 {n:"hatch", key:goKey,  f:hatchBand, geo:1},
 {n:"body",  key:goKey,  f:bodyBand,  geo:1},
 {n:"open",  key:goKey,  f:openBand,  geo:1},
 {n:"col",   key:gKey,   f:colBand,   geo:1},
 {n:"fixt",  key:nKey,   f:fixtBand,  geo:1},
 {n:"stair", key:nKey,   f:stairBand, geo:1},
 {n:"axis",  key:nKey,   f:axisBand},
 {n:"dim",   key:nKey,   f:dimBand},
 {n:"anno",  key:nKey,   f:annoBand},
 {n:"block", key:nKey,   f:blockBand, geo:1},
 {n:"bad",   key:goKey,  f:badBand,   diag:1},
 {n:"sheet", key:nKey,   f:sheetBand, sheet:1}
];
export const bandNames=()=>BANDS.map(b=>b.n);

const CB=new Map();          /* اسم النطاق → {k,raw,lv,out,box} */
function bandOf(b,cx){
 let e=CB.get(b.n);
 const k=b.key();
 if(!e||e.k!==k){
  const raw=b.f(cx)||[];
  e={k,raw,lv:-1,out:null,box:null};
  CB.set(b.n,e);
  bump("band"); bump("prims",raw.length);
 }
 /* التصفية على نسخة الطبقات وحدها: كلُّ كاتبٍ في الجدول يُنادي
    layers.invalidate، وهي تُقدّم العدّاد. ولو صفّينا على VER.g
    لأُعيدت تصفيةُ المرجع في كل إطارٍ من سحب جدار. */
 const lv=layVer();
 if(e.lv!==lv){
  e.out=filterPrims(e.raw);
  e.box=primsBBox(e.out);
  e.lv=lv;
  bump("filter",e.raw.length);
 }
 return e;
}
/* ═══ الكاش ═══ */
let CACHE=null, CVER=-1;
let FLAT=null, LASTO=null;
export const invalidate=()=>{
 CVER=-1;
 CB.clear(); FLAT=null; LASTO=null;
 /* والأجسامُ والحلقاتُ لا تُمسَح هنا: مفتاحاهما (goKey/gKey) يكفيان
    داخل الجلسة — bodies() وregionLoops() يقارنان المفتاح ذاتيّاً
    فيُعيدان الاتحادَ نفسَه إن لم تتغيّر الهندسة. مسحُهما هنا كان
    يُجبر إعادة بناء الاتحاد على كلّ نصٍّ أو بُعدٍ يُضاف، رغم أنّ
    نطاقَه (dim/anno) لا صلة له بالجدران. */
};
export function scene(){
 if(CVER===VER.n&&CACHE)return CACHE;
 bump("scene");
 const cx={B:null};
 const outs=[];
 let Bg=null, Bink=null, Ball=null;
 for(let i=0;i<BANDS.length;i++){
  const b=BANDS[i];
  const e=bandOf(b,cx);
  outs.push(e.out);
  if(b.geo)Bg=bboxUnion(Bg,e.box);
  if(!b.sheet&&!b.diag)Bink=bboxUnion(Bink,e.box);
  Ball=bboxUnion(Ball,e.box);
  if(b.geo)cx.B=Bg;          /* الورقة والمحاور تقرآنه */
 }
 /* النسخُ يبقى: تسلسلُ الطلاء واحدٌ فلا سبيل إلى تجزيئه بلا تغيير
    ما يُطلى. وكلفتُه نسخُ مؤشّرات لا بناءُ أوّليات. وحين لا يتبدّل
    نطاقٌ واحد (تبديلُ لاقطٍ · خيارُ شريط) يُعاد المسطَّح كما هو. */
 let same=!!(FLAT&&LASTO&&LASTO.length===outs.length);
 if(same)for(let i=0;i<outs.length;i++)
  if(LASTO[i]!==outs[i]){same=false; break}
 if(!same){
  FLAT=[].concat.apply([],outs);
  LASTO=outs;
 }
 CACHE={P:FLAT, B:Bg, Bink, Ball,
  solid:bodies().solid, lows:bodies().lows,
  bad:badOpens().length,
  stale:staleCount(),
  loose:looseDims(30).length,
  over:S.dims.filter(isOverridden).length,
  hidden:hiddenCount(),
  /* شظاياً لم تُخَط: الجدار ينفتح على الشاشة والحلقة تغيب —
     تقريرٌ لا إصلاح. وبعد اللحم لا تقع إلّا في مدخلٍ معطوبٍ
     فعلاً، فصار العدُّ ذا معنى. */
  open:BCO, openAt:BCAt, weld:BCW};
 CVER=VER.n;
 return CACHE;
}
/* ═══ الصناديق ═══
   ثلاثةٌ محسوبةٌ بعد التصفية، فإخفاءُ طبقةٍ يُصغّرها فعلاً — وكانت
   تُحسَب قبلها: تُخفي مرجعاً مستورداً ثم تُصدِّر، فتخرج ورقةٌ
   بهوامش خالية وfit يُصغِّر إلى لا شيء.
     B     الهندسة المبنيّة — عليها تُبنى الورقة
     Bink  كلُّ ما يُطلى إلّا الورقة والتشخيص — به يُقاس تجاوزها
     Ball  الكلّ — عليه تُلائم الشاشة */
export const sceneBBox   =()=>scene().B;
export const sceneBBoxInk=()=>scene().Bink||scene().B;
export const sceneBBoxAll=()=>scene().Ball||scene().B;
/* ═══ صندوق ما يُطبَع ═══
   طبقةٌ تراها ولا تُطبَع لا يجوز أن تُوسِّع الورقة: العقد «ما تراه
   هو ما يُصدَّر» يعمل في الاتجاهين. ويُحسَب بطلب التصدير لا في كل
   إطار — نقرةٌ لا ستّون في الثانية. */
let PB=null, PBP=null, PBL=-1;
export function sceneBBoxPlot(){
 const c=scene(), lv=layVer();
 if(PB&&PBP===c.P&&PBL===lv)return PB;
 PB=primsBBox(c.P.filter(g=>plots(g.L||"0")))||c.B;
 PBP=c.P; PBL=lv;
 return PB;
}
export const sceneStats=()=>{
 const o={};
 CB.forEach((e,n)=>{o[n]={n:e.out?e.out.length:0,key:e.k}});
 return o;
};
```

<a id="f-js-core-section-js"></a>

---

## `js/core/section.js`

```javascript
/* ═══ المقاطع (Section) ═══
   وحدةٌ هندسية بحتة كأختها الواجهة: تُسأل فتجيب، ولا تُكتَب في S
   حرفاً، ولا يستدعيها مشهدٌ ولا إطار. المقطع يُولَّد بأمر SECTION
   صريحاً وحده، وإن تبدّل المخطّط بعده بقي كما وُلد ويُبلَّغ أنه
   أقدم من الحالة (sectStale) — تقريرٌ لا إصلاح.

   ═══ الفرق عن الواجهة ═══
   الإسقاط واحد (projectWalls في elevation.js)، والفرق في ثلاثة:

   ١ · التصفية: الواجهة تختار بالناظم واتجاهٍ ثابت، والمقطع يختار
       بالمسافة العمودية عن خطّ قطعٍ حرٍّ ترسمه بنقطتين.

   ٢ · الموضع الأفقي: الواجهة تُفرَد تراكمياً، والمقطع يقع في موضعه
       الحقيقي على خطّ القطع (x = المسافة من نقطته الأولى). ومن أراد
       الفرد: {unfold:1}.

   ٣ · العرض: عرضُ الجدار في الواجهة طولُه، وفي المقطع وترُ خطّ
       القطع في جسمه — سماكتُه إن كان عمودياً على الخطّ، وأطولُ منها
       إن كان مائلاً. يُحسَب بتقاطع خطّ القطع مع وجهَي الجدار.

   المخرَج مسطّح: {kind:"rect", x,y,w,h, layer} بالمليمتر، y‑up. */
import {S,VER} from "./state.js";
import {clamp,deg,m2,dm2,R2D} from "./units.js";
import {dir,band,faces,centerLine,wallsIn,TMAX,WTYPE}
 from "./walls.js";
import {opensOf,isPart,openState,depOf,span} from "./opens.js";
import {distSeg,distPoly,segSeg,pip,lineX,bboxOf,bboxPad} from "./geom.js";
import {viewFrame,projectWalls,wallHeight,angDiff} from "./elevation.js";

const R=v=>Math.round(v);

export const SLAY="A-SECT";
export const CUT=300;            /* نطاق القبول العمودي — مم */
export const MINCUT=200;         /* أقصر خطّ قطعٍ مقبول */
export const PAR=1e-6;           /* حدُّ التوازي */

/* ═══ إطار المقطع ═══ */
export function cutFrame(a,b,back){
 const A=[R(a[0]),R(a[1])], B=[R(b[0]),R(b[1])];
 const dx=B[0]-A[0], dy=B[1]-A[1], L=Math.hypot(dx,dy);
 if(L<MINCUT)
  throw new Error(`خطّ القطع ${m2(L)} م — الأقلّ ${m2(MINCUT)} م`);
 const cut=deg(Math.atan2(dy,dx)*R2D);
 const F=viewFrame(deg(cut+(back?90:-90)));
 return {A,B,L,cut,back:back?1:0,
  v:F.v, rt:F.rt, view:F.ang, org:back?B:A};
}
export const uOf=(F,p)=>
 (p[0]-F.org[0])*F.rt.x+(p[1]-F.org[1])*F.rt.y;

export function sectName(ang){
 const A=deg(ang);
 if(Math.min(Math.abs(angDiff(A,0)),Math.abs(angDiff(A,180)))<0.5)
  return "مقطع أفقي";
 if(Math.min(Math.abs(angDiff(A,90)),Math.abs(angDiff(A,270)))<0.5)
  return "مقطع رأسي";
 return `مقطع بزاوية ${R(A)}°`;
}

export function cutDist(w,A,B){
 const p=band(w);
 if(!p)return 1/0;
 const n=p.length;
 for(let i=0;i<n;i++)
  if(segSeg(A,B,p[i],p[(i+1)%n]))return 0;
 if(pip(p,A[0],A[1])||pip(p,B[0],B[1]))return 0;
 let m=1/0;
 for(let i=0;i<n;i++){
  const d=distSeg(A,B,p[i][0],p[i][1]);
  if(d<m)m=d;
 }
 return Math.min(m, distPoly(p,A[0],A[1]), distPoly(p,B[0],B[1]));
}

export function cutFoot(w,F){
 const d=dir(w), f=faces(w);
 if(!d||!f)return null;
 const cr=d.ux*F.rt.y-d.uy*F.rt.x;
 if(Math.abs(cr)<PAR)return {par:1};
 const Pl=lineX(F.A,F.B,f.l[0],f.l[1]);
 const Pr=lineX(F.A,F.B,f.r[0],f.r[1]);
 if(!Pl||!Pr)return {par:1};
 const ul=uOf(F,Pl), ur=uOf(F,Pr);
 return {par:0, ul, ur,
  u0:Math.min(ul,ur), u1:Math.max(ul,ur),
  skew:Math.abs(ul-ur)};
}

export function sOfCut(w,F){
 const d=dir(w), c=centerLine(w);
 if(!d||!c)return null;
 const P=lineX(F.A,F.B,c.a,c.b);
 if(!P)return null;
 return (P[0]-w.a[0])*d.ux+(P[1]-w.a[1])*d.uy;
}

export function sectWalls(a,b,opt){
 const o=opt||{};
 const F=cutFrame(a,b,!!o.back);
 const tol=clamp((o.tol==null)?CUT:+o.tol,0,5000);
 const box=bboxPad(bboxOf([F.A,F.B]),tol+TMAX);
 const P=projectWalls(wallsIn(box),F,{ref:null});
 const list=[], miss=[];
 P.forEach(p=>{
  const w=p.w;
  const dd=cutDist(w,F.A,F.B);
  if(dd>tol)return;
  const f=cutFoot(w,F);
  if(!f||f.par){
   miss.push({code:"par",id:w.id,d:R(dd),
    msg:`${w.id} موازٍ لخطّ القطع — لا وترَ له فلا مقطع`});
   return;
  }
  const s=sOfCut(w,F);
  if(s==null||s<-tol||s>p.L+tol){
   miss.push({code:"end",id:w.id,d:R(dd),
    msg:`${w.id} يقارب الخطّ عند طرفه ولا يعبره`});
   return;
  }
  if(f.u1<=-1||f.u0>=F.L+1){
   miss.push({code:"span",id:w.id,d:R(dd),
    msg:`${w.id} خارج مدى خطّ القطع — مدِّده إن أردته`});
   return;
  }
  list.push(Object.assign({},p,{d:dd,f,s}));
 });
 list.sort((p,q)=>(p.f.u0-q.f.u0)||(p.d-q.d)||(p.i-q.i));
 return {F,tol,L:F.L,list,miss};
}

export function section(a,b,opt){
 const o=opt||{};
 const Q=sectWalls(a,b,opt);
 const F=Q.F, Lc=F.L;
 const unfold=!!o.unfold;
 const gap=Math.max(0,R(+o.gap||0));
 const shapes=[], runs=[], warn=[];
 Q.miss.forEach(m=>warn.push(m));
 let top=0, cur=0;
 Q.list.forEach(p=>{
  const w=p.w, f=p.f, H=wallHeight(w);
  let x0=f.u0, x1=f.u1, clip=0;
  if(unfold){x0=cur; x1=cur+f.skew}
  else{
   if(x0<0){x0=0; clip=1}
   if(x1>Lc){x1=Lc; clip=1}
  }
  const wid=Math.max(1,R(x1-x0));
  const X=R(x0);
  if(clip)warn.push({code:"clip",id:w.id,
   msg:`${w.id} قُصَّ عند حدّ خطّ القطع — لا قصَّ صامتاً`});
  shapes.push({kind:"rect",x:X,y:0,w:wid,h:H,layer:SLAY,
   role:"wall",id:w.id,wall:w.id});
  const run={id:w.id,x0:X,x1:X+wid,w:wid,h:H,
   t:w.t,type:w.type,d:R(p.d),s:R(p.s),
   skew:R(f.skew),clip,flip:p.flip?1:0,opens:0};
  const kx=(f.skew/Math.max(1,w.t));
  opensOf(w.id).forEach(op=>{
   const [a0,a1]=span(op);
   if(p.s<a0-0.5||p.s>a1+0.5)return;
   const oy=Math.max(0,R(op.sill||0)), oh=Math.max(1,R(op.h));
   let ox=X, ow=wid;
   if(isPart(op)){
    const dep=depOf(op,w.t);
    const dw=Math.max(1,dep*kx);
    const at=(op.face==="r")?f.ur:f.ul;
    const to=(op.face==="r")?f.ul:f.ur;
    const s1=(to>=at)?1:-1;
    let n0=Math.min(at,at+s1*dw), n1=Math.max(at,at+s1*dw);
    if(unfold){const sh=X-f.u0; n0+=sh; n1+=sh}
    n0=Math.max(n0,X); n1=Math.min(n1,X+wid);
    if(n1-n0<1)return;
    ox=R(n0); ow=Math.max(1,R(n1-n0));
   }
   shapes.push({kind:"rect",x:ox,y:oy,w:ow,h:oh,layer:SLAY,
    role:isPart(op)?"niche":"open",kindOf:op.kind,
    id:op.id,wall:w.id});
   run.opens++;
   const st=openState(op);
   if(st!=="ok")warn.push({code:st,id:op.id,
    msg:`${op.id} على ${w.id}: `
     +(st==="over"?"تخرج عن مدى جدارها"
      :st==="clash"?"تتراكب مع فتحةٍ أخرى":"يتيمة")});
   if(oy+oh>H)warn.push({code:"tall",id:op.id,
    msg:`${op.id} تعلو جدارها — ${m2(oy+oh)} م فوق ${m2(H)} م`});
   top=Math.max(top,oy+oh);
  });
  top=Math.max(top,H);
  runs.push(run);
  cur=x1+gap;
 });
 const wid=unfold
  ? Math.max(0,R(cur-(runs.length?gap:0)))
  : R(Lc);
 const hgt=R(top);
 return {a:F.A, b:F.B, ang:R(F.cut), view:F.view, back:F.back,
  name:sectName(F.cut), layer:SLAY, tol:Q.tol, L:R(Lc),
  unfold:unfold?1:0, gap,
  shapes, runs, warn, miss:Q.miss,
  w:wid, h:hgt,
  bbox:runs.length?{x0:0,y0:0,x1:wid,y1:hgt}:null,
  n:{walls:runs.length, opens:shapes.length-runs.length,
     shapes:shapes.length, miss:Q.miss.length},
  ver:{n:VER.n,g:VER.g,o:VER.o}, at:Date.now()};
}

let LAST=null;
export function sectPrims(e,dx,dy){
 const t=e||LAST;
 if(!t)return [];
 const X=R(dx||0), Y=R(dy||0);
 return t.shapes.map(s=>({t:"poly",L:s.layer,cl:1,
  pts:[[X+s.x, Y+s.y], [X+s.x+s.w, Y+s.y],
       [X+s.x+s.w, Y+s.y+s.h], [X+s.x, Y+s.y+s.h]]}));
}
export const lastSect=()=>LAST;
export const clearSect=()=>{LAST=null};
export function buildSect(a,b,opt){
 LAST=section(a,b,opt);
 return LAST;
}
export const sectStale=e=>{
 const t=e||LAST;
 return t?(t.ver.g!==VER.g||t.ver.o!==VER.o):null;
};
export function sectSay(e){
 const t=e||LAST;
 if(!t)return "لا مقطع — نفّذ SECTION";
 return `${t.name}: ${t.n.walls} جدار · ${t.n.opens} فتحة · `
  +dm2(t.w,t.h,"م")
  +(t.n.miss?` · ${t.n.miss} قارب ولم يُقطَع`:"")
  +(t.warn.length?` · ${t.warn.length} ملاحظة`:"")
  +(sectStale(t)?" · المخطّط تبدّل بعد توليده":"");
}
export function sectCmd(a,b,opt){
 const e=buildSect(a,b,opt);
 if(!e.n.walls)throw new Error(
  `لا جدار يعبره خطّ القطع بنطاق ${m2(e.tol)} م`
  +(e.n.miss?` — ${e.n.miss} قاربه: ${e.miss[0].msg}`:""));
 return e;
}
export const sectRunSay=r=>`${r.id} (${WTYPE[r.type].n}): `
 +`${m2(r.x0)} → ${m2(r.x1)} م · سماكة ${m2(r.t)} م`
 +(r.skew>r.t+1?` · وتر ${m2(r.skew)} م بالميل`:"")
 +` · ${r.opens} فتحة`+(r.clip?" · مقصوص":"");
```

<a id="f-js-core-sheet-js"></a>

---

## `js/core/elevation.js`

```javascript
/* ═══ الواجهات (Elevation) ═══
   وحدةٌ هندسية بحتة: لا Canvas ولا DOM ولا مؤقّتات ولا كاشَ نسخة —
   تُسأل فتجيب. ولا يستدعيها مشهدٌ ولا إطار: الواجهة تُولَّد بأمر
   ELEV صريحاً وحده، فلا يتحرّك شيء إلا بأمرك. وإن تبدّل المخطّط
   بعدها بقيت الواجهة كما وُلدت، ويُبلَّغ أنها أقدم من الحالة
   (elevStale) — تقريرٌ لا إصلاح، كحال openState وبصمة المناطق.

   ولا يُكتَب في S حرفٌ واحد: التوليد قراءةٌ محضة، فلا يمرّ بـedit()
   ولا يدخل التاريخ ولا يستدعي حفظاً.

   المخرَج مسطّح: {kind:"rect", x,y,w,h, layer} بالمليمتر.
     x من يسار الواجهة إلى يمينها كما يراها الناظر
     y من منسوب الأرض صاعداً (y‑up كالـDXF)
   فالمصدِّر يقلب المحور إن احتاج، ولا يُقلَب هنا مرّتين. */
import {S,VER} from "./state.js";
import {clamp,deg,m2,dm2} from "./units.js";
import {dir,wallLen,isLow,lowH,isArc} from "./walls.js";
import {opensOf,isPart,openState} from "./opens.js";

const R=v=>Math.round(v);
const D2R=Math.PI/180;

export const ELAY="A-ELEV";
export const TOL=45;                 /* نصف نطاق القبول بالدرجات */
export const VIEWS={E:0,N:90,W:180,S:270};
export const VNAME={E:"الواجهة الشرقية",N:"الواجهة الشمالية",
 W:"الواجهة الغربية",S:"الواجهة الجنوبية"};

export const angDiff=(a,b)=>((a-b)%360+540)%360-180;

export function viewAngle(v){
 if(typeof v==="number"&&isFinite(v))return deg(v);
 const k=String(v==null?"":v).trim().toUpperCase();
 if(VIEWS[k]!=null)return deg(VIEWS[k]);
 const n=parseFloat(k);
 if(isFinite(n))return deg(n);
 throw new Error(`اتجاه نظر غير مفهوم: «${v}» — `
  +`استعمل N أو S أو E أو W أو زاويةً بالدرجات`);
}
export function viewName(a){
 const A=deg(a);
 for(const k of Object.keys(VIEWS))
  if(Math.abs(angDiff(VIEWS[k],A))<0.5)return VNAME[k];
 return `واجهة بزاوية ${R(A)}°`;
}

/* ═══ جهة الخارج ═══
   تُستنتَج من مركز ثقل أوساط الجدران الخارجية موزوناً بأطوالها؛
   الناظم الخارجي هو المبتعد عن هذا المركز. استنتاجُ عرضٍ لا تعديلَ
   بيانات — لا يُكتَب في S. */
export function extRef(list){
 let sx=0, sy=0, sl=0;
 (list||S.walls).forEach(w=>{
  if(w.type!=="ext")return;
  const L=wallLen(w);
  if(L<1e-6)return;
  sx+=((w.a[0]+w.b[0])/2)*L; sy+=((w.a[1]+w.b[1])/2)*L; sl+=L;
 });
 return sl?[sx/sl,sy/sl]:null;
}
export function outNormal(w,ref){
 const d=dir(w);
 if(!d)return null;
 const l={x:d.nx,y:d.ny,ang:deg(d.ang+90)};
 const r={x:-d.nx,y:-d.ny,ang:deg(d.ang-90)};
 if(!ref)return {n:l,alt:r,sure:0};
 const mx=(w.a[0]+w.b[0])/2-ref[0], my=(w.a[1]+w.b[1])/2-ref[1];
 const p=mx*l.x+my*l.y;
 if(Math.abs(p)<Math.max(1,w.t/2))return {n:l,alt:r,sure:0};
 return (p>0)?{n:l,alt:r,sure:1}:{n:r,alt:l,sure:1};
}

/* ═══ إطار النظر ═══
   v اتجاه النظر · rt يمين الناظر (محور التوزيع الأفقي).
   ويُبنى من زاويةٍ لا من نقطتين: المقطع يشتقّ زاويته من خطّ قطعه
   ثم ينادي هذه — فالإطار واحدٌ للاثنين. */
export function viewFrame(view){
 const a=viewAngle(view);
 const vx=Math.cos(a*D2R), vy=Math.sin(a*D2R);
 return {ang:a, v:{x:vx,y:vy}, rt:{x:-vy,y:vx}};
}

/* ═══ الإسقاط ═══ مشتركٌ بين الواجهة والمقطع ═══
   يُسقط جداراً واحداً في إطار النظر ولا يقرّر شيئاً: لا يصفّي بنوعٍ
   ولا بزاويةٍ ولا بمسافة — القرار عند المستدعي. */
export function projectWall(w,F,ref,i){
 /* الجدار القوسي لا يُسقَط بعد — إسقاطه كوترٍ مستقيمٍ يعطي شكلاً
    وطولاً خاطئين بلا تحذير. يُستبعَد هنا صراحةً (تقريرٌ في
    inspect.js لا إسقاطٌ خاطئ صامت) إلى أن تُبنى واجهات/مقاطع
    القوس في مرحلةٍ قادمة. */
 if(isArc(w))return null;
 const d=dir(w);
 if(!d)return null;
 const L=wallLen(w);
 if(L<1e-6)return null;
 const N=outNormal(w,ref);
 if(!N)return null;
 const mx=(w.a[0]+w.b[0])/2, my=(w.a[1]+w.b[1])/2;
 return {w, i:(i==null?-1:i), d, L:R(L),
  n:N.n, alt:N.alt, sure:N.sure,
  lat:mx*F.rt.x+my*F.rt.y, dep:mx*F.v.x+my*F.v.y,
  flip:(d.ux*F.rt.x+d.uy*F.rt.y)<0};
}

/* keep مرشِّحٌ اختياريّ يُنفَّذ قبل الإسقاط. ref===null يعني
   «لا تحسب مرجعاً» — المقطع لا يحتاج جهةَ الخارج. */
export function projectWalls(list,F,opt){
 const o=opt||{};
 const src=list||S.walls;
 const ref=(o.ref===null)?null
  :((o.ref&&isFinite(o.ref[0])&&isFinite(o.ref[1]))
    ? o.ref : extRef(src));
 const keep=(typeof o.keep==="function")?o.keep:null;
 const out=[];
 src.forEach((w,i)=>{
  if(keep&&!keep(w))return;
  const p=projectWall(w,F,ref,i);
  if(p)out.push(p);
 });
 return out;
}

/* ارتفاع الجدار: w.h إن وُجد (السترة)، وإلّا meta.wallH.
   مُصدَّرٌ لأن المقطع يقرأ الارتفاع نفسه. */
export const wallHeight=w=>{
 if(w.h!=null&&isFinite(w.h))return Math.max(200,R(+w.h));
 return isLow(w)?lowH(w):R(S.meta.wallH);
};

/* ═══ اختيار جدران الواجهة وترتيبها ═══ */
export function elevWalls(view,opt){
 const o=opt||{}, F=viewFrame(view), a=F.ang;
 const tol=clamp(+o.tol||TOL,1,90);
 const ref=(o.ref&&isFinite(o.ref[0])&&isFinite(o.ref[1]))
  ? o.ref : extRef();
 const P=projectWalls(S.walls,F,{ref,keep:w=>w.type==="ext"});
 const out=[];
 P.forEach(p=>{
  let n=p.n, off=Math.abs(angDiff(p.n.ang,a));
  if(o.both||!p.sure){
   const o2=Math.abs(angDiff(p.alt.ang,a));
   if(o2<off){n=p.alt; off=o2}
  }
  if(off>tol+1e-9)return;
  out.push({w:p.w, i:p.i, L:p.L, n, off, sure:p.sure,
   lat:p.lat, dep:p.dep, flip:p.flip});
 });
 out.sort((p,q)=>(p.lat-q.lat)||(q.dep-p.dep)||(p.i-q.i));
 return {view:a, v:F.v, rt:F.rt, ref, tol, list:out};
}

/* ═══ التوليد ═══ */
export function elevation(view,opt){
 const o=opt||{}, P=elevWalls(view,opt);
 const gap=Math.max(0,R(+o.gap||0));
 const shapes=[], runs=[], warn=[];
 let x=0, top=0;
 P.list.forEach(p=>{
  const w=p.w, L=p.L, H=wallHeight(w);
  shapes.push({kind:"rect",x,y:0,w:L,h:H,layer:ELAY,
   role:"wall",id:w.id,wall:w.id});
  const run={id:w.id,x0:x,x1:x+L,L,h:H,flip:p.flip?1:0,
   off:R(p.off),dep:R(p.dep),sure:p.sure,opens:0};
  if(!p.sure)warn.push({code:"side",id:w.id,
   msg:`${w.id} يمرّ بمركز المسقط — جهة خارجه غير محسومة`});
  opensOf(w.id)
   .map(op=>({op, c:p.flip?(L-op.s):op.s}))
   .sort((m,n)=>(m.c-n.c)||(m.op.id<n.op.id?-1:1))
   .forEach(({op,c})=>{
    const ow=Math.max(1,R(op.w)), oh=Math.max(1,R(op.h));
    const oy=Math.max(0,R(op.sill||0));
    const ox=R(x+c-ow/2);
    shapes.push({kind:"rect",x:ox,y:oy,w:ow,h:oh,layer:ELAY,
     role:isPart(op)?"niche":"open",kindOf:op.kind,
     id:op.id,wall:w.id});
    run.opens++;
    const st=openState(op);
    if(st!=="ok")warn.push({code:st,id:op.id,
     msg:`${op.id} على ${w.id}: `
      +(st==="over"?"تخرج عن مدى جدارها"
       :st==="clash"?"تتراكب مع فتحةٍ أخرى":"يتيمة")});
    if(oy+oh>H)warn.push({code:"tall",id:op.id,
     msg:`${op.id} تعلو جدارها — ${m2(oy+oh)} م فوق ${m2(H)} م`});
    if(ox<x-1||ox+ow>x+L+1)warn.push({code:"out",id:op.id,
     msg:`${op.id} تخرج عن فرد ${w.id}`});
    top=Math.max(top,oy+oh);
   });
  top=Math.max(top,H);
  runs.push(run);
  x+=L+gap;
 });
 const wid=Math.max(0,R(x-(runs.length?gap:0))), hgt=R(top);
 return {view:P.view, name:viewName(P.view), layer:ELAY,
  shapes, runs, warn, gap, ref:P.ref, tol:P.tol,
  w:wid, h:hgt,
  bbox:runs.length?{x0:0,y0:0,x1:wid,y1:hgt}:null,
  n:{walls:runs.length, opens:shapes.length-runs.length,
     shapes:shapes.length},
  ver:{n:VER.n,g:VER.g,o:VER.o}, at:Date.now()};
}

/* ═══ إلى أوّليات المشروع ═══ */
let LAST=null;
export function elevPrims(e,dx,dy){
 const t=e||LAST;
 if(!t)return [];
 const X=R(dx||0), Y=R(dy||0);
 return t.shapes.map(s=>({t:"poly",L:s.layer,cl:1,
  pts:[[X+s.x, Y+s.y], [X+s.x+s.w, Y+s.y],
       [X+s.x+s.w, Y+s.y+s.h], [X+s.x, Y+s.y+s.h]]}));
}

/* ═══ الأمر ELEV ═══
   آخر واجهةٍ وُلّدت تُحفَظ في الوحدة لا في S — تقريرٌ مشتقّ لا
   بيانات مشروع. */
export const lastElev=()=>LAST;
export const clearElev=()=>{LAST=null};
export function buildElev(view,opt){
 LAST=elevation(view,opt);
 return LAST;
}
export const elevStale=e=>{
 const t=e||LAST;
 return t?(t.ver.g!==VER.g||t.ver.o!==VER.o):null;
};
export function elevSay(e){
 const t=e||LAST;
 if(!t)return "لا واجهة — نفّذ ELEV";
 return `${t.name}: ${t.n.walls} جدار · ${t.n.opens} فتحة · `
  +dm2(t.w,t.h,"م")
  +(t.warn.length?` · ${t.warn.length} ملاحظة`:"")
  +(elevStale(t)?" · المخطّط تبدّل بعد توليدها":"");
}
export function elevCmd(arg,opt){
 const e=buildElev(arg==null?"S":arg,opt);
 if(!e.n.walls)throw new Error(
  `لا جدار خارجيّ يواجه ${e.name} بتفاوت ${e.tol}° — `
  +`الواجهة تُبنى من type="ext" وحدها`);
 return e;
}
```

<a id="f-js-core-entreg-js"></a>

---

## `js/core/proj3d.js`

```javascript
/* ═══ الإسقاط الأكسونومتري ═══
   رياضيّاتٌ خالصة بلا قماشٍ ولا DOM: تُسقِط نقطةً ثلاثيةً (بالمليمتر)
   على مستوى الشاشة بدورانٍ حول Z (yaw) ثم إمالةٍ (pitch) وإسقاطٍ
   متوازٍ (orthographic). تُختبَر في Node، وتقرؤها ui/view3d.js.

   yaw   دوران أفقيّ حول المحور الرأسي Z
   pitch إمالة الكاميرا (0 = مسقطٌ علويّ · π/2 = واجهةٌ جانبية)
   depth عمقٌ لترتيب الرسّام (الأبعد يُرسَم أوّلاً) */

export function project(p, cam){
 const x=+p[0]||0, y=+p[1]||0, z=+p[2]||0;
 const cy=Math.cos(cam.yaw||0), sy=Math.sin(cam.yaw||0);
 const cp=Math.cos(cam.pitch==null?Math.PI/3:cam.pitch);
 const sp=Math.sin(cam.pitch==null?Math.PI/3:cam.pitch);
 /* دوران حول Z */
 const rx= x*cy - y*sy;
 const ry= x*sy + y*cy;
 /* pitch زاويةُ الارتفاع: 90° مسقطٌ علويّ (الأرض تملأ الشاشة) ·
    0° واجهةٌ أماميّة (الارتفاع يملأ الشاشة). فالأرض ry تُقاس
    بـsin والارتفاع z بـcos — وزيادةُ z ترفع النقطة دائماً. */
 const sx= rx;
 const svy= ry*sp + z*cp;              /* أعلى الشاشة موجب */
 const depth= ry*cp - z*sp;            /* بُعدٌ عن الكاميرا */
 return {sx, sy:svy, depth};
}
/* إلى إحداثيات القماش (y للأسفل) — داخليّ */
function toScreen(pr, cam){
 const k=cam.scale||1;
 return [ (cam.ox||0) + pr.sx*k, (cam.oy||0) - pr.sy*k ];
}
export const projScreen=(p,cam)=>toScreen(project(p,cam),cam);

/* صندوق الإسقاط لمجموعة نقاط ثلاثية — لملاءمة العرض */
export function projBBox(pts3, cam){
 let x0=1/0,y0=1/0,x1=-1/0,y1=-1/0;
 for(const p of (pts3||[])){
  const pr=project(p,cam);
  if(pr.sx<x0)x0=pr.sx; if(pr.sx>x1)x1=pr.sx;
  if(pr.sy<y0)y0=pr.sy; if(pr.sy>y1)y1=pr.sy;
 }
 return (x0>x1)?null:{x0,y0,x1,y1};
}
/* شدّة تظليل وجهٍ من مُتَّجهه العموديّ واتّجاه النور (كلاهما 3D) */
export function shade(normal, light){
 const n=norm3(normal), l=norm3(light||[0.4,-0.5,0.85]);
 const d=n[0]*l[0]+n[1]*l[1]+n[2]*l[2];
 return Math.max(0, Math.min(1, 0.35 + 0.65*Math.abs(d)));
}
function norm3(v){
 const L=Math.hypot(v[0],v[1],v[2])||1;
 return [v[0]/L, v[1]/L, v[2]/L];
}
```

<a id="f-js-core-ref-js"></a>

---

## `js/core/underlay.js`

```javascript
/* ═══ صورة مرجعية للتتبّع والمعايرة ═══ */
const st = { src: null, img: null, x: 0, y: 0, mpp: 0.01, rot: 0, opacity: 0.5, visible: true, locked: false, w: 0, h: 0 };
let notify = () => {};
export const state = () => ({ ...st });
export const onChange = fn => { notify = fn || (() => {}); };
export function setImage(src) {
  st.src = src;
  if (typeof Image === "undefined") { notify(); return; }
  const im = new Image();
  im.onload = () => { st.img = im; st.w = im.naturalWidth; st.h = im.naturalHeight; notify(); };
  im.src = src;
}
export const setOpacity = v => { st.opacity = Math.min(1, Math.max(0, +v || 0)); notify(); };
export const setVisible = v => { st.visible = !!v; notify(); };
export const setLocked = v => { st.locked = !!v; notify(); };
export const move = (x, y) => { if (!st.locked) { st.x = +x || 0; st.y = +y || 0; notify(); } };
export const setRotation = r => { st.rot = +r || 0; notify(); };
export const setScale = mpp => { st.mpp = Math.max(1e-9, +mpp || st.mpp); notify(); };
export function calibrate(p1, p2, realMeters) {
  const cur = Math.hypot(p2.x - p1.x, p2.y - p1.y);
  if (cur <= 0 || realMeters <= 0) return st.mpp;
  st.mpp *= realMeters / cur; notify(); return st.mpp;
}
export function draw(ctx, worldToScreen, pxPerWorld) {
  if (!st.visible || !st.img) return;
  const o = worldToScreen([st.x, st.y]);
  const sw = st.w * st.mpp * pxPerWorld, sh = st.h * st.mpp * pxPerWorld;
  ctx.save(); ctx.globalAlpha = st.opacity; ctx.translate(o[0], o[1]); ctx.rotate(-st.rot);
  ctx.drawImage(st.img, 0, -sh, sw, sh); ctx.restore();
}
export const toJSON = () => ({ src: st.src, x: st.x, y: st.y, mpp: st.mpp, rot: st.rot, opacity: st.opacity, visible: st.visible });
export function fromJSON(d) {
  if (!d) return;
  Object.assign(st, { x: d.x || 0, y: d.y || 0, mpp: d.mpp || 0.01, rot: d.rot || 0, opacity: d.opacity ?? 0.5, visible: d.visible !== false });
  if (d.src) setImage(d.src);
}
```

<a id="f-js-core-units-js"></a>

---

## `js/core/sheet.js`

```javascript
/* ═══ الورقة وبلوك العنوان ═══
   يُبنى كلّه بالمليمتر النموذجي: مقاس الورقة × المقياس. فيمرّ في
   خطّ الأنابيب نفسه، ويُصدَّر مع كل شيء، ولا فضاء ورقة منفصل.
   لا يستورد render.js — الصندوق يُمرَّر وسيطاً، فلا دورة. */
import {S,txtH} from "./state.js";
import {clamp,m2,m3,mnum,deg,D2R,scl,dim2} from "./units.js";

const R=v=>Math.round(v);
/* المقاسات بالمليمتر · أفقياً (عرض × ارتفاع) */
export const SIZES={
 A4:[297,210], A3:[420,297], A2:[594,420],
 A1:[841,594], A0:[1189,841]};
export const SNAMES=Object.keys(SIZES);

export function paperMM(){
 const s=SIZES[S.sheet.size]||SIZES.A3;
 return (S.sheet.orient==="p")?[s[1],s[0]]:[s[0],s[1]];
}
/* مقاس الورقة محوَّلاً إلى مليمتر نموذجي */
export function paperModel(){
 const k=Math.max(1,S.meta.scale);
 const p=paperMM();
 return [p[0]*k, p[1]*k];
}
/* ═══ مستطيل الورقة في إحداثيات النموذج ═══
   المركز من S.sheet.cx/cy إن ضُبط، وإلا مركز الرسم. */
export function sheetRect(bbox){
 const [W,H]=paperModel();
 let cx=S.sheet.cx, cy=S.sheet.cy;
 if(cx==null||cy==null){
  if(bbox){cx=(bbox.x0+bbox.x1)/2; cy=(bbox.y0+bbox.y1)/2}
  else{cx=W/2; cy=H/2}
 }
 return {x0:R(cx-W/2), y0:R(cy-H/2),
         x1:R(cx+W/2), y1:R(cy+H/2), W, H};
}
export const innerRect=r=>{
 const m=Math.max(0,S.sheet.margin)*Math.max(1,S.meta.scale);
 return {x0:r.x0+m, y0:r.y0+m, x1:r.x1-m, y1:r.y1-m};
};
/* هل يقع الرسم كلّه داخل الإطار الداخلي؟ */
export function fitsSheet(bbox){
 if(!bbox)return {ok:1,over:0};
 const i=innerRect(sheetRect(bbox));
 const over=Math.max(0, i.x0-bbox.x0, bbox.x1-i.x1,
                        i.y0-bbox.y0, bbox.y1-i.y1);
 return {ok:over<=0?1:0, over:R(over)};
}
/* ═══ بلوك العنوان ═══
   عمود واحد أسفل يمين الإطار الداخلي · الصفوف بالمليمتر الورقي. */
const TB_W=180;
export function titleRows(){
 const t=S.title||{};
 return [
  {n:"المشروع", v:t.proj||"—", h:11, big:1},
  {n:"المالك",  v:t.owner||"—", h:8},
  {n:"الموقع",  v:t.loc||"—",  h:8},
  {n:"اسم اللوحة", v:S.meta.name||"—", h:11, big:1},
  {n:"المقياس · التاريخ",
   v:`${scl(S.meta.scale)}   ·   ${S.meta.date||""}`, h:9},
  {n:"اللوحة · المراجعة · الرسم",
   v:`${t.sheet||"—"}   ·   ${t.rev||"0"}   ·   ${t.by||"—"}`, h:9}];
}
/* ═══ سهم الشمال ═══
   رمزٌ حقيقي في زاوية الإطار الداخلي، زاويته S.meta.north مقيسةً
   عكس الساعة من الشمال. يُصدَّر مع كل شيء لأنه أوّلياتٌ لا شارةُ
   شاشة — والشارة في الواجهة مؤشّرٌ عليه لا بديلٌ عنه. */
export function northPrims(i,k){
 if(!+S.sheet.north)return [];
 const L="A-SHET", out=[];
 const r=8*k, pad=13*k;
 const cx=R(i.x0+pad), cy=R(i.y0+pad);
 const a=(90+(+S.meta.north||0))*D2R;
 const ux=Math.cos(a), uy=Math.sin(a);
 const nx=-uy, ny=ux;
 const P=(u,v)=>[R(cx+ux*u+nx*v), R(cy+uy*u+ny*v)];
 out.push({t:"arc",L,cx,cy,r:R(r),a0:0,a1:359.9,sheet:1});
 /* رأسٌ مصمَّت وذيلٌ مشروح — يُقرأ اتجاهه بلا لبس */
 out.push({t:"poly",L,sheet:1,cl:1,
  pts:[P(r*1.05,0),P(-r*0.3,r*0.42),P(-r*0.3,-r*0.42)]});
 out.push({t:"line",L,sheet:1,a:P(-r*0.3,0),b:P(-r*1.05,0)});
 out.push({t:"text",L,sheet:1,s:"ش",
  x:P(r*1.75,0)[0], y:R(P(r*1.75,0)[1]-k*1.2),
  h:R(k*3.4), al:"mc"});
 return out;
}
/* ═══ أوّليات الورقة ═══ */
export function sheetPrims(bbox){
 if(!+S.sheet.on)return [];
 const k=Math.max(1,S.meta.scale);
 const r=sheetRect(bbox), i=innerRect(r);
 const L="A-SHET", out=[];
 const RC=q=>[[R(q.x0),R(q.y0)],[R(q.x1),R(q.y0)],
              [R(q.x1),R(q.y1)],[R(q.x0),R(q.y1)]];
 out.push({t:"poly",L,pts:RC(r),cl:1,sheet:1});
 out.push({t:"poly",L,pts:RC(i),cl:1,sheet:1});
 northPrims(i,k).forEach(g=>out.push(g));
 if(!+S.sheet.tb)return out;

 const rows=titleRows();
 const totH=rows.reduce((s,x)=>s+x.h,0);
 const w=TB_W*k, H=totH*k;
 const x0=i.x1-w, x1=i.x1, yb=i.y0;
 out.push({t:"poly",L,sheet:1,cl:1,pts:[
  [R(x0),R(yb)],[R(x1),R(yb)],[R(x1),R(yb+H)],[R(x0),R(yb+H)]]});
 /* الصفوف من الأسفل إلى الأعلى بترتيب معكوس */
 let y=yb;
 const lab=k*2.0, val=k*3.2, valBig=k*4.6;
 for(let n=rows.length-1;n>=0;n--){
  const rw=rows[n], hh=rw.h*k;
  if(n<rows.length-1)
   out.push({t:"line",L,sheet:1,
    a:[R(x0),R(y)], b:[R(x1),R(y)]});
  out.push({t:"text",L,sheet:1,s:rw.n,
   x:R(x1-k*2.5), y:R(y+hh-lab*1.35), h:R(lab), al:"br"});
  out.push({t:"text",L,sheet:1,s:rw.v,
   x:R(x1-k*2.5), y:R(y+k*1.8),
   h:R(rw.big?valBig:val), al:"br"});
  y+=hh;
 }
 return out;
}
```

<a id="f-js-core-sindex-js"></a>

---

## `js/core/ref.js`

```javascript
/* ═══ المرجع المستورد ═══
   جامدٌ بالتصميم: لا يُحدَّد ولا يُحرَّر ولا يدخل الاتحاد ولا الحلقات
   ولا المساحات ولا البصمات. تراه وتقيس عليه وتلتقط نقاطه، ثم ترسم
   جدرانك فوقه بيدك — ولا يُستنتَج منه جدار.

   الإحداثيات تُخزَّن مرّةً بالمليمتر، والتحويل يُخزَّن صريحاً
   ويُطبَّق عند العرض — فالمحاذاة المتكرّرة لا تتراكم. */
import {S,touch,refBump} from "./state.js";
import {clamp,deg,D2R,R2D,m2,m3} from "./units.js";
import {dist,bboxOf,bboxUnion} from "./geom.js";

const R=v=>Math.round(v);
export const RLAY="A-REFR";
export const hasRef=()=>!!(S.ref&&S.ref.ents&&S.ref.ents.length);
export const refCount=()=>hasRef()?S.ref.ents.length:0;
export const KIND={l:"خطّ",p:"مضلّع",a:"قوس",t:"نصّ",x:"نقطة"};

/* ═══ التحويل ═══ */
export const refTr=()=>{
 const t=(S.ref&&S.ref.tr)||{};
 return {k:(+t.k||1), rot:deg(+t.rot||0),
  dx:R(+t.dx||0), dy:R(+t.dy||0)};
};
export const isIdent=()=>{
 const t=refTr();
 return Math.abs(t.k-1)<1e-9&&t.rot===0&&!t.dx&&!t.dy;
};
export function mapper(round){
 const t=refTr(), a=t.rot*D2R;
 const ca=Math.cos(a)*t.k, sa=Math.sin(a)*t.k;
 return (round===false)
  ? p=>[p[0]*ca-p[1]*sa+t.dx, p[0]*sa+p[1]*ca+t.dy]
  : p=>[R(p[0]*ca-p[1]*sa+t.dx), R(p[0]*sa+p[1]*ca+t.dy)];
}
/* تركيب تحويلٍ متشابهٍ جديد على القائم — لا استبدال، فلا نفقد
   المعايرة السابقة عند المحاذاة */
export function applySim(m,adeg,tx,ty){
 const t=refTr(), a=adeg*D2R;
 const ca=Math.cos(a), sa=Math.sin(a);
 S.ref.tr={
  k:clamp(t.k*m,1e-4,1e4),
  rot:deg(t.rot+adeg),
  dx:R(m*(t.dx*ca-t.dy*sa)+tx),
  dy:R(m*(t.dx*sa+t.dy*ca)+ty)};
 touch();
 return S.ref.tr;
}
export function alignRef(p1,p2,q1,q2){
 const d1=dist(p1,p2), d2=dist(q1,q2);
 if(d1<1)throw new Error("نقطتا المرجع متطابقتان");
 if(d2<1)throw new Error("نقطتا الهدف متطابقتان");
 const m=d2/d1;
 const a=deg(Math.atan2(q2[1]-q1[1],q2[0]-q1[0])*R2D
  -Math.atan2(p2[1]-p1[1],p2[0]-p1[0])*R2D);
 const ar=a*D2R, ca=Math.cos(ar), sa=Math.sin(ar);
 applySim(m,a, q1[0]-m*(p1[0]*ca-p1[1]*sa),
               q1[1]-m*(p1[0]*sa+p1[1]*ca));
 return {k:m,rot:a,from:d1,to:d2};
}
export function calRef(p1,p2,real){
 const d=dist(p1,p2);
 if(d<1)throw new Error("النقطتان متطابقتان");
 if(!(real>=1))throw new Error("المسافة الحقيقية غير صالحة");
 const m=real/d;
 applySim(m,0, p1[0]*(1-m), p1[1]*(1-m));
 return {k:m,was:d,now:real};
}
export function moveRef(from,to){
 applySim(1,0, to[0]-from[0], to[1]-from[1]);
 return {dx:to[0]-from[0], dy:to[1]-from[1]};
}
export function resetRef(){
 S.ref.tr={k:1,rot:0,dx:0,dy:0};
 touch();
}
/* ═══ التحميل والإزالة ═══ */
export function setRef(res,name){
 S.ref={name:String(name||"").slice(0,80),
  units:(res.units&&res.units.name)||"",
  uf:(res.units&&res.units.f)||1, enc:res.enc||"",
  guessed:res.guessed?1:0,
  tr:{k:1,rot:0,dx:0,dy:0},
  ents:res.ents||[], src:res.src||{}, off:{},
  skip:res.skip||{}, trunc:res.trunc||0,
  approx:res.approx||{}};
 refBump();          /* نسخةٌ جديدة: اللقطات بعدها تشير إليها */
 touch();
 return S.ref;
}
export function clearRef(){
 const n=refCount();
 S.ref={name:"",units:"",uf:1,enc:"",guessed:0,
  tr:{k:1,rot:0,dx:0,dy:0},ents:[],src:{},off:{},
  skip:{},trunc:0,approx:{}};
 refBump();
 touch();
 return n;
}
/* ═══ طبقات المصدر ═══
   ملفّ DXF يحمل عشرات الطبقات، أكثرها ضجيج. الإخفاء هنا فرديّ
   وداخل المرجع، ولا علاقة له بطبقات مِسطَر. */
export const srcOn=n=>!(S.ref.off&&S.ref.off[n]);
export function srcSet(n,off){
 if(!S.ref.off)S.ref.off={};
 if(off)S.ref.off[n]=1; else delete S.ref.off[n];
 touch();
 return srcOn(n);
}
export const srcList=()=>Object.keys(S.ref.src||{})
 .sort((a,b)=>(S.ref.src[b]-S.ref.src[a])||a.localeCompare(b));
export const srcShown=()=>srcList().filter(srcOn).length;
export const visEnts=()=>hasRef()
 ? S.ref.ents.filter(e=>srcOn(e.sl||"0")) : [];

/* ═══ الأوّليات ═══ */
export function refPrims(){
 if(!hasRef())return [];
 const P=mapper(), t=refTr(), out=[];
 visEnts().forEach(e=>{
  if(e.t==="l")out.push({t:"line",L:RLAY,a:P(e.a),b:P(e.b),
   dash:e.dash||null,ref:1});
  else if(e.t==="p"){
   if(!e.pts||e.pts.length<2)return;
   out.push({t:"poly",L:RLAY,pts:e.pts.map(P),
    cl:e.cl?1:0,dash:e.dash||null,ref:1});
  }
  else if(e.t==="a"){
   const c=P(e.c);
   out.push({t:"arc",L:RLAY,cx:c[0],cy:c[1],
    r:Math.max(1,R(e.r*t.k)),
    a0:deg(e.a0+t.rot),a1:deg(e.a1+t.rot),ref:1});
  }
  else if(e.t==="t"){
   const p=P(e.p);
   out.push({t:"text",L:RLAY,s:e.s,x:p[0],y:p[1],
    h:Math.max(1,R(e.h*t.k)),rot:deg((e.rot||0)+t.rot),
    al:e.al||"bl",ref:1});
  }
  else if(e.t==="x"){
   const p=P(e.p), d=Math.max(30,R(60*t.k));
   out.push({t:"line",L:RLAY,ref:1,
    a:[p[0]-d,p[1]], b:[p[0]+d,p[1]]});
   out.push({t:"line",L:RLAY,ref:1,
    a:[p[0],p[1]-d], b:[p[0],p[1]+d]});
  }
 });
 return out;
}
export function refBBox(){
 if(!hasRef())return null;
 const P=mapper(), t=refTr();
 let B=null;
 visEnts().forEach(e=>{
  if(e.t==="l")B=bboxUnion(B,bboxOf([P(e.a),P(e.b)]));
  else if(e.t==="p")B=bboxUnion(B,bboxOf(e.pts.map(P)));
  else if(e.t==="a"){
   const c=P(e.c), r=e.r*t.k;
   B=bboxUnion(B,{x0:c[0]-r,y0:c[1]-r,x1:c[0]+r,y1:c[1]+r});
  }
  else B=bboxUnion(B,bboxOf([P(e.p)]));
 });
 return B;
}
/* ═══ نقاط الالتقاط ═══
   شبكة خلايا تُبنى مرّةً لكل نسخة حالة — فالتقاطٌ على ملفٍّ كبير
   لا يمسح ستّين ألف كيان في كل حركة مؤشّر. */
const CELL=2000, MAXPT=240000;
let GRID=null, GVER=-1;
const key=(x,y)=>Math.floor(x/CELL)+","+Math.floor(y/CELL);
function addPt(G,p,kind,n){
 if(n.c>=MAXPT)return;
 const k=key(p[0],p[1]);
 let a=G.get(k);
 if(!a){a=[]; G.set(k,a)}
 a.push({p:[R(p[0]),R(p[1])],kind});
 n.c++;
}
const mid=(a,b)=>[(a[0]+b[0])/2,(a[1]+b[1])/2];
function build(){
 const G=new Map(), n={c:0};
 if(!hasRef())return G;
 const P=mapper(false), t=refTr();
 visEnts().forEach(e=>{
  if(e.t==="l"){
   const a=P(e.a), b=P(e.b);
   addPt(G,a,"end",n); addPt(G,b,"end",n);
   addPt(G,mid(a,b),"mid",n);
  }else if(e.t==="p"){
   const q=e.pts.map(P);
   const m=e.cl?q.length:q.length-1;
   q.forEach(p=>addPt(G,p,"end",n));
   for(let i=0;i<m;i++)
    addPt(G,mid(q[i],q[(i+1)%q.length]),"mid",n);
  }else if(e.t==="a"){
   const c=P(e.c), r=e.r*t.k;
   addPt(G,c,"cen",n);
   for(let i=0;i<4;i++){
    const a=(i*90+t.rot)*D2R;
    addPt(G,[c[0]+r*Math.cos(a),c[1]+r*Math.sin(a)],"qua",n);
   }
   const s=(e.a0+t.rot)*D2R, f=(e.a1+t.rot)*D2R;
   addPt(G,[c[0]+r*Math.cos(s),c[1]+r*Math.sin(s)],"end",n);
   addPt(G,[c[0]+r*Math.cos(f),c[1]+r*Math.sin(f)],"end",n);
  }else addPt(G,P(e.p),"ins",n);
 });
 return G;
}
export function refGrid(){
 if(GVER===S.__ver&&GRID)return GRID;
 GRID=build();
 GVER=S.__ver;
 return GRID;
}
/* أقرب نقطة مرجعية داخل نصف قطر — أو null */
export function refSnap(x,y,r){
 if(!hasRef())return null;
 if(!+((S.os&&S.os.ref)!=null?S.os.ref:1))return null;
 const G=refGrid(), rad=Math.max(1,r||150);
 let best=null, bd=rad;
 const cx=Math.floor(x/CELL), cy=Math.floor(y/CELL);
 for(let i=-1;i<=1;i++)for(let j=-1;j<=1;j++){
  const a=G.get((cx+i)+","+(cy+j));
  if(!a)continue;
  for(const q of a){
   const d=Math.hypot(q.p[0]-x,q.p[1]-y);
   if(d<bd){bd=d; best={p:q.p,kind:q.kind,d,src:"ref"}}
  }
 }
 return best;
}
export const refSnapCount=()=>{
 let n=0;
 refGrid().forEach(a=>{n+=a.length});
 return n;
};
/* ═══ التقرير ═══ */
export function refStats(){
 if(!hasRef())return null;
 const B=refBBox(), t=refTr();
 const by={};
 S.ref.ents.forEach(e=>{by[e.t]=(by[e.t]||0)+1});
 return {name:S.ref.name, n:refCount(), shown:visEnts().length,
  units:S.ref.units, guessed:!!S.ref.guessed, enc:S.ref.enc,
  layers:srcList().length, shownLayers:srcShown(),
  k:t.k, rot:t.rot, dx:t.dx, dy:t.dy, by, bbox:B,
  skip:S.ref.skip||{}, trunc:S.ref.trunc||0,
  approx:S.ref.approx||{}};
}
```

<a id="f-js-core-render-js"></a>

---

## `js/core/modify.js`

```javascript
/* ═══ التحويلات وعمليات التعديل ═══
   المبدأ: كل تحويل يُحسب من لقطة الأصل لا من الحالة الجارية، فتصير
   المعاينة الحيّة صحيحة والسحب المتكرّر لا يتراكم.
   ولا عملية هنا تُنفَّذ بلا أمر صريح على تحديد صريح.

   النقل والنسخ يعملان على كل الأنواع. الدوران والمرآة يعملان على
   الجميع، ويرفضان البُعد والسلسلة إن كان التحويل يفسد قياسهما —
   فلا يُعرَض رقمٌ خاطئ بهيئة يقين.
   والإزاحة والقطع والقصّ والتمديد والشدّ واللحم للجدران وحدها. */
import {S,touch} from "./state.js";
import {newId,clamp,D2R,R2D,deg,m2,m3} from "./units.js";
import {dist,lineX,nearOnSeg,mid,bboxOf} from "./geom.js";
import {dir,wallById,wallLen,band,addWall,MINW} from "./walls.js";
import {opensOf,openById,span,delOpen} from "./opens.js";
import {areaById} from "./areas.js";
import {dimById,chainById,annoById} from "./dims.js";
import {colById} from "./cols.js";
import {fixById} from "./fixt.js";
import {stById} from "./stairs.js";
import {grabOf,entOf,moveEnt,shapeOf,outlineOf,NAME} from "./ents.js";
import {pickable} from "./layers.js";
import {ENT} from "./entreg.js";

const R=v=>Math.round(v);
const P2=p=>[R(p[0]),R(p[1])];

/* ═══ اللقطة الجماعية ═══
   الفتحة لا تُلقَط: s نسبيّ فتتبع جدارها مجّاناً. */
export function grab(list){
 const out=[];
 (list||[]).forEach(s=>{
  if(!s||s.k==="open")return;
  if(!pickable(s))return;
  const o=grabOf(s);
  if(o)out.push({s,o});
 });
 return out;
}
export const segsOf=G=>(G||[])
 .filter(g=>g.s.k==="wall"&&g.o.a)
 .map(g=>[g.o.a,g.o.b]);

/* ═══ التكرار ═══
   الجدار حاضنٌ فيسحب فتحاته؛ والفتحة لا تُنسَخ وحدها لأنها لا
   تقوم بلا حاضن. وما عداهما نسخةٌ بمعرّفٍ جديد، وحقولٌ تُطرَح
   بالإعلان (dupDrop) لا بشرطٍ هنا. */
export function dupEnt(s,withOpens){
 const e=entOf(s);
 if(!e)return null;
 const d=ENT[s.k];
 if(!d||d.noDup)return null;
 const cp=JSON.parse(JSON.stringify(e));
 cp.id=newId(d.pre);
 if(s.k==="wall"){
  S.walls.push(cp);
  let no=0;
  if(withOpens!==false){
   opensOf(s.id).forEach(o=>{
    S.opens.push(Object.assign({},o,{id:newId("O"),wall:cp.id}));
    no++;
   });
  }
  touch();
  return {e:cp,opens:no};
 }
 (d.dupDrop||[]).forEach(k=>{delete cp[k]});
 S[d.coll].push(cp);
 touch();
 return {e:cp,opens:0};
}
export const dupWall=(id,withOpens)=>{
 const r=dupEnt({k:"wall",id},withOpens);
 return r?{wall:r.e,opens:r.opens}:null;
};
/* ═══ النقل ═══ */
export function moveAll(G,dx,dy){
 (G||[]).forEach(({s,o})=>moveEnt(s,o,dx,dy));
 touch();
 return (G||[]).length;
}
/* ═══ نسخةٌ واحدةٌ محوَّلة ═══
   التحويلُ من لقطة الأصل لا من النسخة المتحرّكة، وxf يقع على
   المقبض المنسوخ. قلبُ copyAll والمصفوفتين معاً — فالقاعدةُ في
   موضعٍ واحد. */
function copyOne(s,o,withOpens,xf){
 const r=dupEnt(s,withOpens);
 if(!r)return null;
 const ns={k:s.k,id:r.e.id};
 const g=grabOf(ns);
 if(g){
  Object.keys(o).forEach(k=>{if(k!=="e")g[k]=o[k]});
  xf(ns,g);
 }
 return {ns,opens:r.opens};
}
export function copyAll(G,dx,dy,n,withOpens){
 n=clamp(R(n||1),1,200);
 if((G||[]).length*n>500)throw new Error("أكثر من 500 نسخة");
 let nw=0, no=0;
 const made=[];
 for(let i=1;i<=n;i++){
  (G||[]).forEach(({s,o})=>{
   const q=copyOne(s,o,withOpens,(ns,g)=>moveEnt(ns,g,dx*i,dy*i));
   if(!q)return;
   nw++; no+=q.opens; made.push(q.ns);
  });
 }
 touch();
 return {walls:nw,opens:no,made};
}
/* ═══ الدوران ═══ */
export const rotP=(p,c,degv)=>{
 const a=degv*D2R, ca=Math.cos(a), sa=Math.sin(a);
 const dx=p[0]-c[0], dy=p[1]-c[1];
 return [R(c[0]+dx*ca-dy*sa), R(c[1]+dx*sa+dy*ca)];
};
const q90=a=>{
 const d=deg(a);
 return (Math.abs(d%90)<0.01)?Math.round(d/90)%4:-1;
};
/* ═══ من يقبل دوراناً حرّاً ═══
   البُعدُ الأفقيُّ والرأسيُّ والسلسلةُ لا تدور إلّا بمضاعفات ٩٠° —
   فلا يُعرَض رقمٌ خاطئ بهيئة يقين. والقاعدةُ تُقرأ مرّتين: rotEnt
   عند التحويل، وcanRotate للترشيح قبل أن تُنشَأ نسخةٌ تُرفَض. */
const rotOk=(k,kind,degv)=>{
 if(k==="dim")return (kind==="al")||q90(degv)>=0;
 if(k==="chain")return q90(degv)>=0;
 return true;
};
export const canRotate=(s,degv)=>{
 const e=entOf(s);
 return !!e&&rotOk(s&&s.k,e.kind,degv);
};
/* يعيد true إن طُبِّق · false إن رُفض (يُبلَّغ باسمه) */
function rotEnt(s,o,c,a){
 const e=o.e;
 if(s.k==="wall"||s.k==="stair"){
  e.a=rotP(o.a,c,a); e.b=rotP(o.b,c,a);
  return true;
 }
 if(s.k==="area"){
  e.ring=o.ring.map(p=>rotP(p,c,a));
  if(o.lp)e.lp=rotP(o.lp,c,a);
  return true;
 }
 if(s.k==="col"){
  const p=rotP([o.x,o.y],c,a);
  e.x=p[0]; e.y=p[1];
  if(e.kind!=="circ")e.rot=deg(o.rot+a);
  return true;
 }
 if(s.k==="fix"){
  const p=rotP([o.x,o.y],c,a);
  e.x=p[0]; e.y=p[1];
  e.rot=deg(o.rot+a);
  return true;
 }
 if(s.k==="anno"){
  if(o.pts){e.pts=o.pts.map(p=>rotP(p,c,a)); return true}
  const p=rotP([o.x,o.y],c,a);
  e.x=p[0]; e.y=p[1];
  if(e.kind==="text")e.rot=deg((e.rot||0)+a);
  return true;
 }
 if(s.k==="dim"){
  if(!rotOk("dim",o.kind,a))return false;
  if(o.kind==="al"){
   e.a=rotP(o.a,c,a); e.b=rotP(o.b,c,a);
   return true;
  }
  /* نقطةٌ على خطّ البُعد تدور معه، فيُستخرَج pos الجديد منها */
  const ref=(o.kind==="h")?[o.a[0],o.pos]:[o.pos,o.a[1]];
  const M=rotP(ref,c,a);
  e.a=rotP(o.a,c,a); e.b=rotP(o.b,c,a);
  if(q90(a)%2===1)e.kind=(o.kind==="h")?"v":"h";
  e.pos=(e.kind==="h")?M[1]:M[0];
  return true;
 }
 if(s.k==="chain"){
  if(!rotOk("chain",null,a))return false;
  const b=rotP(o.base,c,a);
  const p0=rotP(chainRef(o),c,a);
  e.base=b;
  if(q90(a)%2===1)e.axis=(e.axis==="h")?"v":"h";
  e.pos=(e.axis==="h")?p0[1]:p0[0];
  /* الاتجاه قد ينقلب: القيَم مكتوبة فلا تُمَسّ، والأساس يُصحَّح */
  return true;
 }
 return false;
}
const chainRef=o=>(o.axis==="h")
 ? [o.base[0],o.pos] : [o.pos,o.base[1]];

export function rotateAll(G,c,degv,copy,withOpens){
 let nw=0,no=0,ref=[];
 const work=copy?[]:null;
 if(copy){
  (G||[]).forEach(({s,o})=>{
   const r=dupEnt(s,withOpens);
   if(!r)return;
   const ns={k:s.k,id:r.e.id};
   const g=grabOf(ns);
   if(!g)return;
   Object.keys(o).forEach(k=>{if(k!=="e")g[k]=o[k]});
   work.push({s:ns,o:g});
   no+=r.opens;
  });
 }
 (work||G||[]).forEach(({s,o})=>{
  if(rotEnt(s,o,c,degv))nw++;
  else ref.push(`${s.id} ${NAME[s.k]||s.k}`);
 });
 touch();
 return {walls:nw,opens:no,refused:ref,
  made:work?work.map(x=>x.s):[]};
}
/* ═══ المرآة ═══
   الانعكاس يقلب اتجاه المسار، فالمحاذاة l تصير r وبالعكس ليبقى
   الجسم على الوجه نفسه هندسياً. وجهة فتح الباب تُقلَب لأنها
   تُقاس من عمود المسار. والأداة تُعكَس بعلم mir. */
const flipAlign=a=>(a==="l")?"r":((a==="r")?"l":"c");
function mirEnt(s,o,M,axisKind,rotDelta){
 const e=o.e;
 if(s.k==="wall"){
  e.a=M(o.a); e.b=M(o.b);
  e.align=flipAlign(e.align);
  opensOf(e.id).forEach(op=>{
   op.swing=(op.swing==="left")?"right":"left";
   if(op.face)op.face=(op.face==="l")?"r":"l";
  });
  return true;
 }
 if(s.k==="stair"){e.a=M(o.a); e.b=M(o.b); return true}
 if(s.k==="area"){
  e.ring=o.ring.map(M).reverse();
  if(o.lp)e.lp=M(o.lp);
  return true;
 }
 if(s.k==="col"){
  const p=M([o.x,o.y]);
  e.x=p[0]; e.y=p[1];
  if(e.kind!=="circ")e.rot=deg(rotDelta-o.rot);
  return true;
 }
 if(s.k==="fix"){
  const p=M([o.x,o.y]);
  e.x=p[0]; e.y=p[1];
  e.rot=deg(rotDelta-o.rot);
  if(e.mir)delete e.mir; else e.mir=1;
  return true;
 }
 if(s.k==="anno"){
  if(o.pts){e.pts=o.pts.map(M); return true}
  const p=M([o.x,o.y]);
  e.x=p[0]; e.y=p[1];
  if(e.kind==="text")e.rot=deg(rotDelta-(e.rot||0));
  return true;
 }
 if(s.k==="dim"){
  if(o.kind==="al"){e.a=M(o.a); e.b=M(o.b); return true}
  if(!axisKind)return false;      /* h/v تحتاج محوراً قائماً */
  const A=M(o.a), B=M(o.b);
  const Q=M((o.kind==="h")?[o.a[0],o.pos]:[o.pos,o.a[1]]);
  e.a=A; e.b=B;
  if(axisKind==="d")e.kind=(o.kind==="h")?"v":"h";
  e.pos=(e.kind==="h")?Q[1]:Q[0];
  return true;
 }
 if(s.k==="chain"){
  if(!axisKind)return false;
  const b=M(o.base);
  const q=M(chainRef(o));
  e.base=b;
  if(axisKind==="d")e.axis=(e.axis==="h")?"v":"h";
  e.pos=(e.axis==="h")?q[1]:q[0];
  return true;
 }
 return false;
}
export function mirrorAll(G,a,b,keep,withOpens){
 const dx=b[0]-a[0], dy=b[1]-a[1], L=Math.hypot(dx,dy);
 if(L<1)throw new Error("محور المرآة صفري");
 const ux=dx/L, uy=dy/L;
 const M=p=>{
  const px=p[0]-a[0], py=p[1]-a[1], t=px*ux+py*uy;
  return [R(a[0]+2*ux*t-px), R(a[1]+2*uy*t-py)];
 };
 /* محور قائم؟ h ⇒ أفقي/رأسي يبقى · d ⇒ قطريّ يبدّل */
 const ang=deg(Math.atan2(uy,ux)*R2D);
 /* تفاوتٌ حول المضاعف من الجهتين — 89.9999 محورٌ قائم */
 const at=(v,m)=>{const r=((v%m)+m)%m; return Math.min(r,m-r)<0.01};
 const m90=at(ang,90), m45=at(ang-45,90);
 const axisKind=m90?"s":(m45?"d":null);
 const rotDelta=2*ang;                /* θ' = 2α − θ */
 let nw=0,no=0,ref=[];
 const work=keep?[]:null;
 if(keep){
  (G||[]).forEach(({s,o})=>{
   const r=dupEnt(s,withOpens);
   if(!r)return;
   const ns={k:s.k,id:r.e.id};
   const g=grabOf(ns);
   if(!g)return;
   Object.keys(o).forEach(k=>{if(k!=="e")g[k]=o[k]});
   work.push({s:ns,o:g});
   no+=r.opens;
  });
 }
 (work||G||[]).forEach(({s,o})=>{
  if(mirEnt(s,o,M,axisKind,rotDelta))nw++;
  else ref.push(`${s.id} ${NAME[s.k]||s.k}`);
 });
 touch();
 return {walls:nw,opens:no,refused:ref,
  made:work?work.map(x=>x.s):[]};
}
/* ═══ الإزاحة ═══ على عمود المسار · clear يجعل المسافة صافية ═══ */
export function offsetWall(id,d,side,clear,t,type,withOpens){
 const w=wallById(id);
 if(!w)throw new Error("الجدار غير موجود");
 const u=dir(w);
 if(!u)throw new Error("الجدار صفري");
 const t2=clamp(R(t||w.t),50,1000);
 const D=d+(clear?(w.t+t2)/2:0);
 const sg=(side<0)?-1:1;
 const px=u.nx*sg*D, py=u.ny*sg*D;
 const r=dupWall(id,withOpens);
 if(!r)throw new Error("تعذّر النسخ");
 r.wall.a=[R(w.a[0]+px),R(w.a[1]+py)];
 r.wall.b=[R(w.b[0]+px),R(w.b[1]+py)];
 r.wall.t=t2;
 if(type)r.wall.type=type;
 touch();
 return {wall:r.wall,opens:r.opens,d:D};
}
/* ═══ الفتحات عند تقصير الجدار ═══
   لا زحف أبداً: ما يقع في المقطوع يُحذَف، وما يبقى لا يُمَسّ.
   ولا نقلّم موضع الباقي ولو صار خارج المدى — العلامة الحمراء
   تخبرك، وأنت تقرّر. */
function dropOpensIn(id,lo,hi){
 const kill=opensOf(id).filter(o=>{
  const [a,b]=span(o);
  return a<hi-1&&lo<b-1;
 });
 kill.forEach(o=>delOpen(o));
 return kill.length;
}
function shiftOpensTo(id,newId2,lo,hi,ds){
 let n=0;
 opensOf(id).forEach(o=>{
  const [a,b]=span(o);
  if(a<lo-1||b>hi+1)return;
  o.wall=newId2;
  o.s=R(o.s-ds);
  n++;
 });
 return n;
}
/* ═══ القطع عند نقطة ═══ */
export function breakWall(id,p){
 const w=wallById(id);
 if(!w)throw new Error("الجدار غير موجود");
 const u=dir(w);
 if(!u)throw new Error("الجدار صفري");
 const s=R((p[0]-w.a[0])*u.ux+(p[1]-w.a[1])*u.uy);
 if(s<MINW||s>u.L-MINW)
  throw new Error(`نقطة القطع على ${m2(s)} م — يجب أن تبعد `
   +`${m2(MINW)} م عن الطرفين على الأقل`);
 /* الفتحة التي تعبر نقطة القطع تُحذَف: لا تنتمي إلى أحدهما */
 const lost=dropOpensIn(id,s,s);
 const n=Object.assign({},w,{id:newId("W"),
  a:[R(w.a[0]+u.ux*s),R(w.a[1]+u.uy*s)], b:w.b.slice()});
 S.walls.push(n);
 const moved=shiftOpensTo(id,n.id,s,u.L,s);
 w.b=n.a.slice();
 touch();
 return {nw:n,a:s,b:u.L-s,lost,moved};
}
/* ═══ القصّ ═══ الحدّ صريح دائماً — لا «كل الجدران» ضمنياً ═══ */
export function trimWall(id,cuts,p){
 const w=wallById(id);
 if(!w)throw new Error("الجدار غير موجود");
 const u=dir(w);
 if(!u)throw new Error("الجدار صفري");
 const T=[];
 (cuts||[]).forEach(c=>{
  const o=wallById(c);
  if(!o||o.id===id)return;
  const x=lineX(w.a,w.b,o.a,o.b);
  if(!x)return;
  /* التقاطع يجب أن يقع على جسم الحدّ نفسه لا على امتداده */
  if(nearOnSeg(o.a,o.b,x[0],x[1]).d>2)return;
  const s=R((x[0]-w.a[0])*u.ux+(x[1]-w.a[1])*u.uy);
  if(s>2&&s<u.L-2)T.push(s);
 });
 if(!T.length)
  throw new Error("لا حدّ من المحدَّدة يعبر هذا الجدار");
 T.sort((a,b)=>a-b);
 const sp=clamp((p[0]-w.a[0])*u.ux+(p[1]-w.a[1])*u.uy,0,u.L);
 let lo=null, hi=null;
 T.forEach(v=>{
  if(v<=sp&&(lo==null||v>lo))lo=v;
  if(v>=sp&&(hi==null||v<hi))hi=v;
 });
 const gl=(lo!=null), gh=(hi!=null);
 if(!gl&&!gh)throw new Error("انقر على جزء بين حدَّين أو خارجهما");
 if(gl&&gh){
  if(hi-lo<20)throw new Error("الجزء المحدَّد أرقّ من 2 سم");
  const lost=dropOpensIn(id,lo,hi);
  const n=Object.assign({},w,{id:newId("W"),
   a:[R(w.a[0]+u.ux*hi),R(w.a[1]+u.uy*hi)], b:w.b.slice()});
  S.walls.push(n);
  const moved=shiftOpensTo(id,n.id,hi,u.L,hi);
  w.b=[R(w.a[0]+u.ux*lo),R(w.a[1]+u.uy*lo)];
  touch();
  return {mode:"mid",cut:hi-lo,nw:n,lost,moved};
 }
 if(!gl){
  const lost=dropOpensIn(id,0,hi);
  let moved=0;
  opensOf(id).forEach(o=>{o.s=R(o.s-hi); moved++});
  w.a=[R(w.a[0]+u.ux*hi),R(w.a[1]+u.uy*hi)];
  touch();
  return {mode:"start",cut:hi,lost,moved};
 }
 const lost=dropOpensIn(id,lo,u.L);
 w.b=[R(w.a[0]+u.ux*lo),R(w.a[1]+u.uy*lo)];
 touch();
 return {mode:"end",cut:u.L-lo,lost,moved:0};
}
/* ═══ التمديد ═══ */
export function extendWall(id,bnds,p){
 const w=wallById(id);
 if(!w)throw new Error("الجدار غير موجود");
 const u=dir(w);
 if(!u)throw new Error("الجدار صفري");
 const atEnd=((p[0]-w.a[0])*u.ux+(p[1]-w.a[1])*u.uy)>u.L/2;
 let best=null;
 (bnds||[]).forEach(c=>{
  const o=wallById(c);
  if(!o||o.id===id)return;
  const x=lineX(w.a,w.b,o.a,o.b);
  if(!x)return;
  if(nearOnSeg(o.a,o.b,x[0],x[1]).d>2)return;
  const s=(x[0]-w.a[0])*u.ux+(x[1]-w.a[1])*u.uy;
  if(atEnd){if(s>u.L+10&&(best==null||s<best))best=s}
  else{if(s<-10&&(best==null||s>best))best=s}
 });
 if(best==null)
  throw new Error("لا حدّ من المحدَّدة في هذا الاتجاه");
 if(atEnd){
  w.b=[R(w.a[0]+u.ux*best),R(w.a[1]+u.uy*best)];
  touch();
  return {mode:"end",add:best-u.L};
 }
 const add=-best;
 w.a=[R(w.a[0]+u.ux*best),R(w.a[1]+u.uy*best)];
 /* الفتحات تُقاس من البداية، فتُزاح بمقدار التمديد */
 opensOf(id).forEach(o=>{o.s=R(o.s+add)});
 touch();
 return {mode:"start",add};
}
/* ═══ الشدّ بإطار ═══
   الأطراف داخل الإطار تتحرّك وحدها. الفتحات لا تُمَسّ: ما خرج
   عن المدى يظهر معطوباً وأنت تقرّر. */
export function stretchGrab(r){
 const G=[];
 const inR=p=>p[0]>=r.x0&&p[0]<=r.x1&&p[1]>=r.y0&&p[1]<=r.y1;
 S.walls.forEach(w=>{
  if(!pickable({k:"wall",id:w.id}))return;
  const a=inR(w.a), b=inR(w.b);
  if(a||b)G.push({e:w,a,b,o:{a:w.a.slice(),b:w.b.slice()}});
 });
 return G;
}
export function stretchApply(G,dx,dy){
 (G||[]).forEach(g=>{
  if(g.a)g.e.a=[g.o.a[0]+dx,g.o.a[1]+dy];
  if(g.b)g.e.b=[g.o.b[0]+dx,g.o.b[1]+dy];
 });
 touch();
 return (G||[]).length;
}
export const stretchPrev=(G,dx,dy)=>(G||[]).map(g=>[
 g.a?[g.o.a[0]+dx,g.o.a[1]+dy]:g.o.a,
 g.b?[g.o.b[0]+dx,g.o.b[1]+dy]:g.o.b]);

/* ═══ اللحم — يخطّط ثم يُنفَّذ بعد التأكيد ═══
   weldPlan يعيد قائمة كل طرف سيتحرّك ومقدار حركته، فتراها قبل
   الموافقة. weldApply لا يفعل إلّا ما في الخطة.

   node  طرفان حرّان متقاربان ⇒ يلتقيان في نقطة واحدة
   axis  طرف قريب من محور جدار محدَّد ⇒ يُسقَط على تقاطع المحورين
   tee   طرف قريب من جسم جدار ⇒ يُسقَط عمودياً على مساره       */
export function weldPlan(ids,tol){
 const T=clamp(tol||30,1,2000);
 const W=(ids||[]).map(id=>wallById(id)).filter(Boolean);
 const ends=[];
 W.forEach(w=>{
  ends.push({w,k:"a",p:w.a.slice()});
  ends.push({w,k:"b",p:w.b.slice()});
 });
 const shared=e=>ends.some(x=>x!==e&&x.w!==e.w&&dist(x.p,e.p)<=1);
 const moves=[], done=new Set();
 const key=e=>e.w.id+"/"+e.k;

 /* ١ — أزواج الأطراف المتقاربة: نقطة واحدة في المنتصف */
 for(let i=0;i<ends.length;i++){
  const e=ends[i];
  if(done.has(key(e))||shared(e))continue;
  let best=null,bd=T;
  for(let j=0;j<ends.length;j++){
   const f=ends[j];
   if(f===e||f.w===e.w||done.has(key(f))||shared(f))continue;
   const d=dist(e.p,f.p);
   if(d>0.5&&d<=bd){bd=d;best=f}
  }
  if(!best)continue;
  const m=[R((e.p[0]+best.p[0])/2), R((e.p[1]+best.p[1])/2)];
  [e,best].forEach(x=>{
   const d=dist(x.p,m);
   if(d>0.5)moves.push({id:x.w.id,end:x.k,from:x.p.slice(),
    to:m.slice(),d:R(d),why:"node"});
   done.add(key(x));
  });
 }
 /* ٢ — الطرف على محور جدار محدَّد: تقاطع المحورين */
 ends.forEach(e=>{
  if(done.has(key(e))||shared(e))return;
  let best=null,bd=T;
  W.forEach(o=>{
   if(o===e.w)return;
   const r=nearOnSeg(o.a,o.b,e.p[0],e.p[1]);
   if(r.d>bd||r.d<=0.5)return;
   if(r.t<=0.002||r.t>=0.998)return;
   const x=lineX(e.w.a,e.w.b,o.a,o.b);
   const q=x||r.p;
   const d=dist(e.p,q);
   if(d<=T&&d>0.5&&d<bd){bd=d;best={q,why:"axis"}}
   else if(r.d<bd){bd=r.d;best={q:r.p,why:"tee"}}
  });
  if(!best)return;
  moves.push({id:e.w.id,end:e.k,from:e.p.slice(),
   to:P2(best.q),d:R(dist(e.p,best.q)),why:best.why});
  done.add(key(e));
 });
 return {moves,tol:T};
}
export function weldApply(plan){
 let n=0;
 ((plan&&plan.moves)||[]).forEach(m=>{
  const w=wallById(m.id);
  if(!w)return;
  /* الأمان: لا نُطبّق إلّا إن كان الطرف ما زال حيث خُطِّط له */
  const cur=(m.end==="a")?w.a:w.b;
  if(dist(cur,m.from)>1)return;
  if(m.end==="a")w.a=m.to.slice(); else w.b=m.to.slice();
  n++;
 });
 /* الجدار الذي صار أقصر من الحدّ الأدنى يُبلَّغ ولا يُحذَف */
 const short=S.walls.filter(w=>wallLen(w)<MINW).map(w=>w.id);
 touch();
 return {moved:n,short};
}
export const WHY={node:"طرفان يلتقيان",axis:"إسقاط على محور",
 tee:"إسقاط على جسم"};

/* ═══ المصفوفة المستطيلة ═══
   ما يقبله copy بحرفه: كلُّ نوعٍ يقبله dupEnt. والفتحةُ ليست منه
   (noDup) — تُنسَخ مع جدارها لا وحدها، فتتبعه مجّاناً.
   والأصلُ خليّةٌ في الشبكة لا نسخةٌ زائدة عليها. */
export function arrayRect(G,nx,ny,dx,dy,withOpens){
 const NX=clamp(R(nx||1),1,100), NY=clamp(R(ny||1),1,100);
 const cop=NX*NY-1;
 if(cop<1)
  throw new Error("صفٌّ واحدٌ وعمودٌ واحد — لا نسخةَ تُنشَأ");
 /* تباعدٌ صفرٌ على محورٍ فيه أكثر من خليّة ⇒ نسخٌ متراكبةٌ صامتة */
 if(NX>1&&!dx)
  throw new Error(`تباعد X صفرٌ مع ${NX} أعمدة — النسخُ تتراكب`);
 if(NY>1&&!dy)
  throw new Error(`تباعد Y صفرٌ مع ${NY} صفوف — النسخُ تتراكب`);
 const src=(G||[]);
 if(src.length*cop>500)
  throw new Error(`${src.length*cop} نسخة — الحدّ 500`);
 let nw=0, no=0;
 const made=[];
 for(let j=0;j<NY;j++)for(let i=0;i<NX;i++){
  if(!i&&!j)continue;                /* موضعُ الأصل */
  src.forEach(({s,o})=>{
   const q=copyOne(s,o,withOpens,(ns,g)=>moveEnt(ns,g,dx*i,dy*j));
   if(!q)return;
   nw++; no+=q.opens; made.push(q.ns);
  });
 }
 touch();
 return {walls:nw,opens:no,made,cells:cop,nx:NX,ny:NY};
}
/* نقطةٌ مرجعيةٌ للكيان — مركزُ صندوقه. تخدم التوزيعَ بلا دوران:
   الموضعُ يدور والهيئةُ تبقى، وهو ما يُطلَب لعمودٍ حول قوس. */
function anchorOf(s){
 const bx=r=>{
  const b=bboxOf(r||[]);
  return b?[R((b.x0+b.x1)/2),R((b.y0+b.y1)/2)]:null;
 };
 const o=outlineOf(s);
 if(o&&o.length){
  const p=bx(o);
  if(p)return p;
 }
 const sh=shapeOf(s);
 if(!sh)return null;
 if(sh.t==="pt")return sh.p.slice();
 if(sh.t==="seg")return mid(sh.a,sh.b).map(R);
 return bx(sh.pts);
}
/* ═══ المصفوفة القطبية ═══
   الدورانُ يمرّ بـrotateAll فينال قاعدتَها: ما لا يدور بهذه الزاوية
   يُرفَض ويُسمّى. والترشيحُ على الخطوة قبل النسخ لا مع كل تكرار —
   مصفوفةٌ ناقصةٌ أسوأُ من مصفوفةٍ مرفوضة، ونسخةٌ مرفوضةٌ تبقى في
   موضع الأصل أسوأُ منهما.

   والخطوةُ مُشتقّةٌ ومُعلَنة: دورةٌ كاملةٌ تُقسَم على العدد فلا
   تتراكب الأخيرةُ على الأصل، وما دونها يمتدّ من الأصل إلى آخر
   نسخةٍ فيبلغ الزاويةَ المطلوبة بالضبط. */
export function arrayPolar(G,c,total,n,rot,withOpens){
 const N=clamp(R(n||2),2,200);
 const T=+total||0;
 if(Math.abs(T)<0.01)
  throw new Error("الزاوية الإجمالية صفر — لا توزيعَ يقع");
 const full=Math.abs(Math.abs(T)-360)<0.05;
 const stepA=T/(full?N:(N-1));
 const cop=N-1;
 const src=[], ref=[];
 (G||[]).forEach(g=>{
  if(rot&&!canRotate(g.s,stepA)){
   ref.push(`${g.s.id} ${NAME[g.s.k]||g.s.k}`);
   return;
  }
  src.push(g);
 });
 if(!src.length)
  throw new Error(`لا عنصرَ يقبل التوزيعَ بخطوة `
   +`${stepA.toFixed(1)}° — البُعدُ الأفقيُّ والرأسيُّ والسلسلةُ `
   +`لا تدور إلّا بمضاعفات 90°`);
 if(src.length*cop>500)
  throw new Error(`${src.length*cop} نسخة — الحدّ 500`);
 let nw=0, no=0;
 const made=[];
 for(let k=1;k<=cop;k++){
  const a=stepA*k;
  if(rot){
   const r=rotateAll(src,c,a,1,withOpens);
   nw+=r.walls; no+=r.opens;
   (r.made||[]).forEach(x=>made.push(x));
   continue;
  }
  src.forEach(({s,o})=>{
   const an=anchorOf(s);
   const q=copyOne(s,o,withOpens,(ns,g)=>{
    if(!an)return;
    const p=rotP(an,c,a);
    moveEnt(ns,g,p[0]-an[0],p[1]-an[1]);
   });
   if(!q)return;
   nw++; no+=q.opens; made.push(q.ns);
  });
 }
 touch();
 return {walls:nw,opens:no,made,refused:ref,
  step:Math.round(stepA*100)/100, n:N, full};
}

/* ═══ كسرُ الركن ═══ يخطّط ثم يُنفَّذ بعد التأكيد ═══
   ولِمَ ضلعٌ مستقيمٌ لا قوس؟ الجدارُ في هذا المشروع نقطتان
   وسماكة — لا انحناءَ في بياناته. فقوسٌ يُطلى ولا يُبنى يجعل
   المساحةَ تُحسَب على ركنٍ حادٍّ وتُرى مدوّرةً، وهو نقضُ عقد
   «ما تراه هو ما يُصدَّر». وتقريبُه أضلاعاً يُنتِج عشراتَ جدرانٍ
   دون MINW، كلُّ واحدٍ منها معرّفٌ في الفهرس وتحذيرٌ في الفاحص.

   والجهةُ المُبقاةُ من نقرتك: عند تقاطعٍ في الوسط أربعةُ أركان،
   والنقرةُ تحسم أيَّها — كما في القصّ. والمقاسُ يُقاس من الركن
   لا من الطرف، فحدُّ الرفض مقدارُ ما يبقى فعلاً.

   d1 للأول وd2 للثاني — والخطّةُ تُعرَض بالمليمتر قبل التنفيذ،
   والتنفيذُ لا يتجاوزها. */
export function chamferPlan(id1,id2,d1,d2,p1,p2){
 const w1=wallById(id1), w2=wallById(id2);
 if(!w1||!w2)throw new Error("الجدار غير موجود");
 if(w1===w2)throw new Error("الجداران واحدٌ — انقر جدارين مختلفين");
 const u1=dir(w1), u2=dir(w2);
 if(!u1||!u2)throw new Error("جدارٌ صفريُّ الطول");
 const X=lineX(w1.a,w1.b,w2.a,w2.b);
 if(!X)throw new Error(`${id1} و ${id2} متوازيان — لا ركنَ بينهما`);
 const D1=Math.max(1,R(d1||0)), D2=Math.max(1,R(d2||d1||0));
 const side=(w,u,d,p)=>{
  const s =(X[0]-w.a[0])*u.ux+(X[1]-w.a[1])*u.uy;
  const sc=(p[0]-w.a[0])*u.ux+(p[1]-w.a[1])*u.uy;
  const keepB=sc>s;                 /* النقرةُ بعد الركن ⇒ يُبقى ما بعده */
  const sg=keepB?1:-1;              /* من الركن إلى ما يُبقى */
  return {id:w.id, d, L:R(u.L),
   end:keepB?"a":"b",
   from:(keepB?w.a:w.b).slice(),
   to:[R(X[0]+u.ux*sg*d), R(X[1]+u.uy*sg*d)],
   cut:R(s+sg*d),                   /* المعاملُ الجديد للطرف المتحرّك */
   remain:keepB?(u.L-s-d):(s-d),
   grow:keepB?Math.max(0,-s):Math.max(0,s-u.L),
   dirIn:[u.ux*sg,u.uy*sg]};
 };
 const A=side(w1,u1,D1,p1||mid(w1.a,w1.b));
 const B=side(w2,u2,D2,p2||mid(w2.a,w2.b));
 const cs=A.dirIn[0]*B.dirIn[0]+A.dirIn[1]*B.dirIn[1];
 const ang=Math.acos(clamp(cs,-1,1))*R2D;
 if(ang<5||ang>175)
  throw new Error(`الجداران يلتقيان بزاوية ${ang.toFixed(1)}° — `
   +`لا ركنَ يُكسَر`);
 [A,B].forEach(x=>{
  if(x.remain<MINW)
   throw new Error(`${x.id}: يبقى منه ${m3(x.remain)} م بعد `
    +`الكسر — الأدنى ${m3(MINW)} م. صغّر المسافة، أو انقره في `
    +`الجهة التي تريد إبقاءها`);
 });
 const len=dist(A.to,B.to);
 if(len<MINW)
  throw new Error(`ضلعُ الكسر ${m3(len)} م — الأدنى ${m3(MINW)} م`);
 return {a:A,b:B,X:P2(X),len:R(len),
  ang:Math.round(ang*10)/10, t:w1.t, type:w1.type};
}
export function chamferApply(plan){
 const P=plan||{}, A=P.a, B=P.b;
 if(!A||!B)throw new Error("لا خطّةَ كسر");
 const W1=wallById(A.id), W2=wallById(B.id);
 if(!W1||!W2)throw new Error("الجدارُ لم يبقَ — أعِد الأداة");
 /* الأمان: لا يُنفَّذ إلّا إن كان الطرفان حيث خُطِّط لهما —
    والفحصُ قبل أيِّ كتابة، فلا حالةَ نصفَ مكسورة. */
 [[W1,A],[W2,B]].forEach(([w,x])=>{
  if(dist((x.end==="a")?w.a:w.b,x.from)>1)
   throw new Error(`${x.id} تحرّك بعد التخطيط — أعِد الأداة`);
 });
 let lost=0;
 [[W1,A],[W2,B]].forEach(([w,x])=>{
  if(x.end==="a"){
   /* البدايةُ تتحرّك: ما قبلها يُطرَح، والباقي يُقاس من موضعها
      الجديد — عقدُ trimWall نفسه. */
   lost+=dropOpensIn(w.id,0,x.cut);
   opensOf(w.id).forEach(o=>{o.s=R(o.s-x.cut)});
   w.a=x.to.slice();
  }else{
   lost+=dropOpensIn(w.id,x.cut,x.L);
   w.b=x.to.slice();
  }
 });
 const nw=addWall(A.to,B.to,P.t,P.type,"c");
 touch();
 return {wall:nw,lost,len:P.len};
}
```

<a id="f-js-core-opens-js"></a>

---

## `js/core/boq.js`

```javascript
/* ═══ جدول الكميات ═══
   تقريرٌ يُجمَع عند الطلب: لا حقل يُخزَّن، ولا شيء يُصلَح.
   كجدول المساحات في areas.js وجدول الفتحات في opens.js —
   الحالةُ هي الأصل، والجدولُ قراءةٌ لها في لحظة.

   ولا DOM هنا ولا تنزيل ولا تنسيق: المليمتر يخرج كما هو،
   والقسمةُ على ألفٍ أو مليون شأنُ من يعرض. فمن يختبر يقارن
   أعداداً صحيحة لا نصوصاً تُقرَّب — والتقريب في مكانٍ واحد
   (io/boq.js) لا في اثنين يفترقان.

   القديمة تدخل: حذفُها يجعل المجموع كذبة، وإدخالُها بلا علامة
   يجعله كذبةً أخرى. فتدخل بعلامة — كما يُبلِّغ arearef ولا يُصلح. */
import {S} from "./state.js";
import {netArea,netPerim,isStale} from "./areas.js";
import {okName,okOf,panOf} from "./opens.js";
import {wallLen,WTYPE,isLow,lowH} from "./walls.js";

/* ═══ الأشيع ═══
   السماكةُ الأكثر تكراراً في النوع. وعند التعادل تُختار الأكبر:
   قرارٌ صريح لا ترتيبُ مصفوفة — فجدارٌ يُحذَف ويُعاد لا يقلب
   الجواب. وn عددُ السماكات المختلفة: واحدةٌ تعني نوعاً متجانساً،
   وأكثرُ تعني أن «الأشيع» يخفي تنوّعاً — فيُقال. */
export function modeOf(vals){
 const m=new Map();
 (vals||[]).forEach(v=>{
  const k=Math.round(+v||0);
  m.set(k,(m.get(k)||0)+1);
 });
 let best=0, bn=-1;
 m.forEach((c,k)=>{
  if(c>bn||(c===bn&&k>best)){bn=c; best=k}
 });
 return {v:m.size?best:0, n:m.size};
}
/* ═══ المناطق ═══
   المساحةُ صافيةٌ بين الوجوه الداخلية — netArea هي هي، لا حساب
   ثانٍ يفترق عنها. والمحيطُ معها لأنه يُقاس من الحلقة نفسها. */
export function areaRows(){
 const rows=S.areas.map(a=>({
  id:a.id,
  name:a.name||"(بلا اسم)",
  area:netArea(a),
  perim:netPerim(a),
  stale:isStale(a)?1:0}));
 rows.sort((x,y)=>(y.area-x.area)||x.id.localeCompare(y.id));
 return {rows,
  total:rows.reduce((s,r)=>s+r.area,0),
  stale:rows.reduce((s,r)=>s+r.stale,0),
  n:rows.length};
}
/* ═══ الفتحات ═══
   تُجمَع بالنوع وحده — لا بالمقاس. فجدول الكميات يسأل «كم باباً
   مفرداً؟» لا «كم باباً بعرض ٩٠٠؟»؛ ذاك سؤالُ openSchedule وله
   جوابُه هناك بمقاسه ورمزه.

   والمساحة الإجمالية تُذكَر لأنها الكميّةُ التي تُشترى: زجاجٌ
   بالمتر المربّع، وحشوةُ بابٍ كذلك. وأصغرُ وأكبرُ عرضٍ يقولان
   إن كان النوع متجانساً. */
export function openRows(){
 const G=new Map();
 S.opens.forEach(o=>{
  const k=o.kind;
  let r=G.get(k);
  if(!r){
   r={kind:k, name:okName(k), n:0, ar:0,
    wMin:1/0, wMax:0, hMin:1/0, hMax:0, pan:0};
   G.set(k,r);
  }
  r.n++;
  r.ar+=(+o.w||0)*(+o.h||0);
  if(o.w<r.wMin)r.wMin=o.w;
  if(o.w>r.wMax)r.wMax=o.w;
  if(o.h<r.hMin)r.hMin=o.h;
  if(o.h>r.hMax)r.hMax=o.h;
  if(okOf(k).pan)r.pan+=panOf(o);
 });
 const rows=[...G.values()].map(r=>{
  if(!isFinite(r.wMin))r.wMin=0;
  if(!isFinite(r.hMin))r.hMin=0;
  return r;
 });
 /* الترتيب بالعدد نازلاً ثم بالمفتاح — لا بترتيب S.opens، فلا
    يتبدّل الجدولُ بإضافةِ فتحةٍ لا تغيّر شيئاً في العدّ */
 rows.sort((a,b)=>(b.n-a.n)||a.kind.localeCompare(b.kind));
 return {rows,
  total:S.opens.length,
  ar:rows.reduce((s,r)=>s+r.ar,0),
  n:rows.length};
}
/* ═══ الجدران ═══
   الطولُ مجموعُ wallLen على المسار المرسوم — لا على محور الجسم
   ولا على الوجه. فالمسارُ هو ما رُسم، وما عداه اشتقاقٌ يختلف
   بالمحاذاة.

   والفتحاتُ لا تُطرَح: طرحُها يحتاج ارتفاعَ الجدار وارتفاعَ كل
   فتحةٍ وجلستَها، وذلك حسابُ حجومٍ لا أطوال. فيُذكَر عددُ فتحات
   النوع تنبيهاً، ويُترَك الطرحُ لمن يريده صريحاً.

   وارتفاعُ السترة من w.h لا من meta.wallH — والباقي من
   meta.wallH، فهو ارتفاعُ الجدار المعلَن. */
export function wallRows(){
 const G=new Map();
 const byId=new Map();
 S.walls.forEach(w=>byId.set(w.id,w));
 const opn=new Map();
 S.opens.forEach(o=>{
  const w=byId.get(o.wall);
  if(!w)return;                      /* اليتيمة لا تُحسَب */
  const t=w.type;
  opn.set(t,(opn.get(t)||0)+1);
 });
 S.walls.forEach(w=>{
  const t=w.type;
  let r=G.get(t);
  if(!r){
   r={type:t, name:(WTYPE[t]||{}).n||t, n:0, len:0,
    ts:[], hs:[], t:0, tn:0, h:0, opens:0};
   G.set(t,r);
  }
  r.n++;
  r.len+=wallLen(w);
  r.ts.push(w.t);
  r.hs.push(isLow(w)?lowH(w):(+S.meta.wallH||3000));
 });
 const rows=[...G.values()].map(r=>{
  const M=modeOf(r.ts);
  r.t=M.v; r.tn=M.n;
  r.h=modeOf(r.hs).v;
  r.len=Math.round(r.len);
  /* المساحة السطحية بالوجه الواحد · بالسماكة الأشيع لا بكلٍّ
     على حدة — فمن أراد الحجم بالضبط قرأ الجدران واحداً واحداً */
  r.face=Math.round(r.len*r.h);
  r.vol=Math.round(r.len*r.h*r.t);
  r.opens=opn.get(r.type)||0;
  delete r.ts; delete r.hs;
  return r;
 });
 const ORD={ext:0,int:1,low:2};
 rows.sort((a,b)=>((ORD[a.type]==null?9:ORD[a.type])
  -(ORD[b.type]==null?9:ORD[b.type]))||a.type.localeCompare(b.type));
 return {rows,
  len:rows.reduce((s,r)=>s+r.len,0),
  face:rows.reduce((s,r)=>s+r.face,0),
  vol:rows.reduce((s,r)=>s+r.vol,0),
  n:S.walls.length};
}
/* ═══ الجدول كاملاً ═══
   ثلاثة أقسام وترويسة. والترويسة من meta لا من التاريخ الحيّ:
   جدولان يُبنيان من الحالة نفسها يتطابقان — فلو حملا وقتَ البناء
   لاختلفا في حرفٍ لا معنى له. وdate حقلُ مشروعٍ قائم في meta. */
export function boq(){
 return {
  name:String(S.meta.name||"PLAN"),
  scale:+S.meta.scale||100,
  date:String(S.meta.date||""),
  wallH:+S.meta.wallH||3000,
  areas:areaRows(),
  opens:openRows(),
  walls:wallRows()};
}
/* سطرُ حصيلةٍ للوحة الحالة — نصٌّ واحد لا كائن */
export const boqLine=B=>`${B.walls.n} جداراً · `
 +`${B.opens.total} فتحة · ${B.areas.n} منطقة`
 +(B.areas.stale?` · ${B.areas.stale} قديمة`:"");
```

<a id="f-js-core-code-js"></a>

---

## `js/core/pricing.js`

```javascript
/* ═══ تسعير حصر الكميات ═══ */
const CFG = { currency: "ر.س", taxRate: 0.15, key: "mistar:pricing" };
const DEFAULTS = {
  wall: { label: "جدران", unit: "م²", rate: 120 },
  floor: { label: "أرضيات", unit: "م²", rate: 90 },
  area: { label: "مساحات", unit: "م²", rate: 0 },
  door: { label: "أبواب", unit: "عدد", rate: 350 },
  window: { label: "نوافذ", unit: "عدد", rate: 420 },
  column: { label: "أعمدة", unit: "عدد", rate: 250 },
  dim: { label: "أبعاد", unit: "عدد", rate: 0 }
};
let RATES = load();
function load() {
  try { return { ...DEFAULTS, ...(JSON.parse(localStorage.getItem(CFG.key)) || {}) }; }
  catch (_) { return { ...DEFAULTS }; }
}
function persist() { try { localStorage.setItem(CFG.key, JSON.stringify(RATES)); } catch (_) {} }
export const getRate = key => RATES[key]?.rate ?? 0;
export const allRates = () => JSON.parse(JSON.stringify(RATES));
export function setRate(key, rate, meta = {}) { RATES[key] = { ...(RATES[key] || {}), ...meta, rate: +rate || 0 }; persist(); }
export function resetRates() { RATES = { ...DEFAULTS }; try { localStorage.removeItem(CFG.key); } catch (_) {} }
export const currency = () => CFG.currency;
export const setCurrency = c => { if (c) CFG.currency = c; };
export const taxRate = () => CFG.taxRate;
export const setTaxRate = r => { CFG.taxRate = Math.max(0, +r || 0); };
export const round2 = n => Math.round((+n + Number.EPSILON) * 100) / 100;
export const formatMoney = n => `${round2(n).toLocaleString("ar-EG", { minimumFractionDigits: 2, maximumFractionDigits: 2 })} ${CFG.currency}`;
export function price(items) {
  const rows = (items || []).map(it => {
    const def = RATES[it.key] || {};
    const rate = it.rate ?? def.rate ?? 0;
    const qty = +it.qty || 0;
    return { key: it.key, label: it.label || def.label || it.key, unit: it.unit || def.unit || "", qty, rate, amount: round2(qty * rate) };
  });
  const subtotal = round2(rows.reduce((s, r) => s + r.amount, 0));
  const tax = round2(subtotal * CFG.taxRate);
  return { rows, subtotal, tax, total: round2(subtotal + tax), currency: CFG.currency, taxRate: CFG.taxRate };
}
export const toJSON = () => ({ currency: CFG.currency, taxRate: CFG.taxRate, rates: RATES });
export function fromJSON(d) {
  if (!d) return;
  if (d.currency) CFG.currency = d.currency;
  if (typeof d.taxRate === "number") CFG.taxRate = Math.max(0, d.taxRate);
  if (d.rates) RATES = { ...DEFAULTS, ...d.rates };
}
```

<a id="f-js-core-proj3d-js"></a>

---

## `js/core/osnap.js`

```javascript
/* ═══ التقاط الكائنات ═══
   طبقة إدخال خالصة: تعينك على إصابة نقطة موجودة، ولا تحرّك شيئاً.
   المرجع المستورد يدخل المرشّحين آخراً وبتحيّزٍ مقصود ضدّه — نقطةُ
   جدارٍ رسمتَه تفوز على نقطة مرجعٍ استوردتَه عند التساوي. */
import {S} from "./state.js";
import {clamp} from "./units.js";
import {nearOnSeg,lineX,dist,bboxHit,bboxOf} from "./geom.js";
import {dir,centerLine,band,faces,isLow} from "./walls.js";
import {opensOf,span,openPt} from "./opens.js";
import {colPoly,colById} from "./cols.js";
import {stPoly} from "./stairs.js";
import {vis} from "./layers.js";
import {refSnap} from "./ref.js";
import * as SI from "./sindex.js";

export const MODES=[
 {k:"end", n:"نهاية", mk:"sq"},
 {k:"mid", n:"منتصف", mk:"tri"},
 {k:"int", n:"تقاطع", mk:"x"},
 {k:"nod", n:"عقدة",  mk:"plus"},
 {k:"per", n:"عمودي", mk:"per"},
 {k:"near",n:"أقرب",  mk:"near"},
 {k:"ref", n:"مرجع",  mk:"ref"}];
export const MNAME={};
MODES.forEach(m=>{MNAME[m.k]=m.n});
export const osOn=()=>MODES.some(m=>+S.os[m.k]);
export const osSummary=()=>{
 const on=MODES.filter(m=>+S.os[m.k]);
 return on.length?on.map(m=>m.n).join(" · "):"لا أنماط";
};
/* الالتقاط على ما تراه: الطبقة المخفيّة لا تُلتقَط نقاطها */
const visW=w=>vis(isLow(w)?"A-WALL-LOW":"A-WALL");
const wallsVis=list=>(list||S.walls).filter(visW);

export function osnap(x,y,tol,from){
 if(!osOn())return null;
 const best={};
 const T=(m,p,ex)=>{
  if(!+S.os[m])return;
  const d=Math.hypot(p[0]-x,p[1]-y);
  if(d>tol)return;
  if(!best[m]||d<best[m].d)
   best[m]=Object.assign(
    {m,d,p:[Math.round(p[0]),Math.round(p[1])]},ex||{});
 };
 /* المرشَّحون من الفهرس: نقطةٌ داخل تفاوتٍ تعني صندوقاً يقع فيه
    الكيان — فما خرج عنه لا يمكن أن يُلتقَط، والباقي يُفحَص كما كان */
 const box=SI.boxAt(x,y,tol);
 const WV=wallsVis(SI.entsIn(box,"wall"));
 /* الالتقاط على المسار المرسوم وعلى الوجهَين المحسوبَين */
 WV.forEach(w=>{
  T("end",w.a); T("end",w.b);
  T("mid",[(w.a[0]+w.b[0])/2,(w.a[1]+w.b[1])/2]);
  const F=faces(w);
  if(F){
   [F.l,F.r].forEach(f=>{
    T("end",f[0]); T("end",f[1]);
    T("mid",[(f[0][0]+f[1][0])/2,(f[0][1]+f[1][1])/2]);
   });
  }
  const d=dir(w);
  if(!d)return;
  const s=(x-w.a[0])*d.ux+(y-w.a[1])*d.uy;
  if(s>=0&&s<=d.L)T("near",[w.a[0]+d.ux*s,w.a[1]+d.uy*s]);
  if(from){
   const t=clamp((from[0]-w.a[0])*d.ux+(from[1]-w.a[1])*d.uy,0,d.L);
   T("per",[w.a[0]+d.ux*t,w.a[1]+d.uy*t],{from});
  }
  /* حدود الفتحات: مواضع مفيدة للقياس والرسم */
  opensOf(w.id).forEach(o=>{
   const [a,b]=span(o);
   T("end",openPt(w,a)); T("end",openPt(w,b));
   T("mid",openPt(w,o.s));
  });
 });
 /* أركان الأعمدة ومراكزها */
 if(vis("A-COLS"))SI.entsIn(box,"col").forEach(c=>{
  const p=colPoly(c);
  if(!p)return;
  T("nod",[c.x,c.y]);
  if(c.kind!=="circ")p.forEach(q=>T("end",q));
 });
 /* أركان الدرج */
 if(vis("A-STRS"))SI.entsIn(box,"stair").forEach(s=>{
  const p=stPoly(s);
  if(!p)return;
  p.forEach(q=>T("end",q));
  T("end",s.a); T("end",s.b);
 });
 /* رؤوس المناطق */
 if(vis("A-AREA"))SI.entsIn(box,"area").forEach(a=>{
  a.ring.forEach(q=>T("end",q));
 });
 /* تقاطع محاور الجدران — على المسارات لا الوجوه */
 if(+S.os.int){
  const near=wallsVis(SI.entsIn(SI.boxAt(x,y,tol*4),"wall"))
   .filter(w=>nearOnSeg(w.a,w.b,x,y).d<tol*4)
   .slice(0,50);
  for(let i=0;i<near.length;i++)for(let j=i+1;j<near.length;j++){
   const p=lineX(near[i].a,near[i].b,near[j].a,near[j].b);
   if(!p)continue;
   const on=q=>nearOnSeg(q.a,q.b,p[0],p[1]).d<2;
   if(on(near[i])&&on(near[j]))T("int",p);
  }
 }
 /* عقد شبكة المحاور — الترشيح على المحور قبل التقاطع، فلا يُضرَب
    عددُ الحروف في عدد الأرقام */
 if(vis("A-GRID")){
  const X=S.grid.xs.filter(v=>Math.abs(v-x)<=tol);
  const Y=S.grid.ys.filter(v=>Math.abs(v-y)<=tol);
  X.forEach(gx=>Y.forEach(gy=>T("nod",[gx,gy])));
 }

 /* المرجع آخر المرشّحين وبتحيّزٍ ضدّه ×1.05 */
 if(+S.os.ref&&vis("A-REFR")){
  const rf=refSnap(x,y,tol);
  if(rf&&(!best.ref||rf.d*1.05<best.ref.d))
   best.ref={m:"ref",d:rf.d*1.05,p:rf.p,kind:rf.kind};
 }
 for(const m of MODES)if(best[m.k])return best[m.k];
 return null;
}
```

<a id="f-js-core-perf-js"></a>

---

