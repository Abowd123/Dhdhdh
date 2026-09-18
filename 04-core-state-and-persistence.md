# CivilDraft — Core: State & Persistence

> إدارة الحالة، الطبقات، السجل التاريخي، الحفظ، الفهرسة، الأداء.

**عدد الملفات:** 12

---

## `js/core/state.js`

```javascript
/* ═══ الحالة · التاريخ ═══
   لا كاش استنتاج هنا: VER عدّاد نسخة يُبطِل كاش العرض وحده.
   كل تعديل يمرّ بـ edit() فيصير ذرّياً وله خطوة تراجع واحدة.
   الطبقات جدولٌ حيّ في S.layers (انظر core/layers.js) — بياناتُ
   مشروعٍ لا تفضيلَ نافذة، لأن إخفاء طبقةٍ يغيّر ما يُصدَّر. */
import {clamp,deg,setIdc,idc,bumpIdc,idNum} from "./units.js";
import * as Store from "../io/store.js";
import {DEFLAYS} from "./laydef.js";
export {LAYERS} from "./laydef.js";   /* توافقٌ لمن كان يستورده هنا */
/* دورةٌ ظاهرية (layers.js يستورد S وVER وtouch من هنا) مقبولةٌ في
   وحدات ES: normLays لا تُنادى وقت التحميل بل من ensureShape —
   أي بعد اكتمال الوحدتين. */
import * as LY from "./layers.js";

export const LSK=Store.LSK;   /* أُبقي للتوافق مع من يستورده */
export const saveMode=()=>Store.mode();

export const COLLS=["walls","opens","areas","dims","chains","anno",
 "cols","fixt","stairs"];
const KEYS=["meta"].concat(COLLS,
 ["grid","opt","rb","os","pol","sheet","title","layers","layst","ref",
  "blocks"]);

export const DEF=()=>({
 meta:{name:"PLAN",scale:100,txtMM:2.2,
  tExt:250,tInt:150,tLow:200,lowH:1000,wallH:3000,
  snap:50,dimDec:2,dimTick:"slash",north:0,
  date:new Date().toISOString().slice(0,10)},
 walls:[],opens:[],areas:[],dims:[],chains:[],anno:[],
  cols:[],fixt:[],stairs:[],blocks:[],
 grid:{xs:[],ys:[]},
 opt:{joins:1,fill:"none",colSolo:0},
 rb:{ortho:1,snap:1,polar:0,grips:1,ends:1,grid:1,gsnap:1,paths:1,
  dyn:1},
 os:{end:1,mid:1,int:1,per:1,near:0,nod:1,ref:1},
 pol:{inc:15,extra:[]},
 sheet:{on:0,size:"A3",orient:"l",margin:12,tb:1,north:1,
  cx:null,cy:null},
 title:{proj:"",owner:"",loc:"",sheet:"A-101",rev:"0",by:""},
 /* الطبقات بياناتُ مشروع: تُحفَظ وتدخل التاريخ لأنها تغيّر ما
    يُصدَّر — لا تفضيلَ نافذة. وS.lay القديم يُطوى فيها بالهجرة. */
 layers:DEFLAYS(),
 layst:{},
 ref:{name:"",units:"",uf:1,enc:"",guessed:0,
  tr:{k:1,rot:0,dx:0,dy:0},ents:[],src:{},off:{},
  skip:{},trunc:0,approx:{}}
});
export const S=DEF();
let blockSeq=1;

/* ═══ ثلاث نسخ ═══
   n  عامّة: تتقدّم بكل تعديل. يقرأها كاش المشهد والفهرس المكاني —
      وهما يتبعان كل شيء يُرسَم أو يُصاب.
   g  هندسية: الجدران والأعمدة وخيار الدمج. وهي وحدها ما يُبطِل
      polyBool وحلقات المناطق وشبكة الأطراف وشبكة المراسي
      وبصمات المناطق وجدول الطبقات.
   o  الفتحات: تُطرَح من الأجسام ولا تُبطِل الحلقات.

   والسبب: سحب مقبض بُعدٍ كان يعيد بناء اتحاد ألف مضلّعٍ في كل
   إطار، وstampOf لكل منطقة، وشبكتَي الأطراف والمراسي، وكاش
   الألوان. والفصل يجعل الكلفة تتبع ما تغيّر فعلاً.

   وtouch يُقدّم n وg معاً بقصد — لا n وحده كما يبدو أوّل النظر:
   سبعةُ مواضع تُعدّل هندسةً بـtouch (applyField · اللوحة المفردة ·
   stretchApply · edit · finish · ops · trace)، فلو كان الافتراض
   سريعاً لأخرج أحدُها مشهداً قديماً. والافتراض الآمن يجعل النسيان
   يُكلِّف أداءً لا صحّة، والإعلان في المواضع الحارّة وحدها:
   سحبُ المقابض وبنّاؤو المجموعات. */
export const VER={n:0,g:0,o:0};
S.__ver=0;
export const touch    =()=>{VER.n++; VER.g++; S.__ver=VER.n};
export const touchGeom=()=>{VER.n++; VER.g++; S.__ver=VER.n};
export const touchOpen=()=>{VER.n++; VER.o++; S.__ver=VER.n};
export const touchView=()=>{VER.n++;          S.__ver=VER.n};
export const txtH=()=>Math.max(1,S.meta.txtMM*Math.max(1,S.meta.scale));
const PT=p=>[Math.round((p&&+p[0])||0),Math.round((p&&+p[1])||0)];

/* ═══ التطبيع الدفاعي ═══
   يُصلح ملفّاً محرَّراً يدوياً، ولا يمسّ هندسةً رسمها المستخدم. */
export function ensureShape(){
 /* الطبقات أوّلاً: يقرأها العدّ والتصفية وresolve، فلا يجوز أن
    يسبقها شيء */
 LY.normLays();
 const d=DEF();
 S.meta=Object.assign(d.meta,S.meta||{});
 S.meta.scale=clamp(parseInt(S.meta.scale,10)||100,1,5000);
 S.meta.txtMM=clamp(+S.meta.txtMM||2.2,0.5,20);
 S.meta.tExt=Math.max(50,+S.meta.tExt||250);
 S.meta.tInt=Math.max(50,+S.meta.tInt||150);
 S.meta.tLow=Math.max(50,+S.meta.tLow||200);
 S.meta.wallH=clamp(+S.meta.wallH||3000,1500,8000);
 S.meta.lowH=clamp(Math.round(+S.meta.lowH||1000),200,S.meta.wallH-200);
 S.meta.snap=clamp(+S.meta.snap||50,1,5000);
 S.meta.dimDec=clamp(parseInt(S.meta.dimDec,10),0,3);
 if(!isFinite(S.meta.dimDec))S.meta.dimDec=2;
 if(!/^(slash|arrow)$/.test(S.meta.dimTick))S.meta.dimTick="slash";
 /* زاوية الشمال: بياناتُ مشروعٍ لا تفضيلَ عرض — تُصدَّر مع اللوحة */
 S.meta.north=deg(+S.meta.north||0);

 COLLS.forEach(k=>{if(!Array.isArray(S[k]))S[k]=[]});
  /* ═══ مثيلات العناصر ═══
     الكتلة تعريفٌ ثابت خارج الحالة، والمثيل بيانات مشروع صريحة. */
  if(!Array.isArray(S.blocks))S.blocks=[];
  S.blocks=S.blocks.filter(b=>b&&typeof b.block==="string"
   &&isFinite(+b.x)&&isFinite(+b.y));
  S.blocks.forEach(b=>{
   b.x=+b.x||0; b.y=+b.y||0;
   b.rot=isFinite(+b.rot)?+b.rot:0;
   b.scale=(isFinite(+b.scale)&&+b.scale>0)?+b.scale:1;
   b.mirror=b.mirror?1:0;
   b.layer=String(b.layer==null?"0":b.layer).slice(0,80)||"0";
   if(b.id==null)b.id="b"+(++blockSeq);
  });
 S.grid=Object.assign({xs:[],ys:[]},S.grid||{});
 ["xs","ys"].forEach(k=>{
  if(!Array.isArray(S.grid[k]))S.grid[k]=[];
  S.grid[k]=[...new Set(S.grid[k].filter(v=>isFinite(v))
   .map(v=>Math.round(v)))].sort((a,b)=>a-b);
 });
 S.opt=Object.assign(d.opt,S.opt||{});
 if(!/^(none|hatch|solid)$/.test(S.opt.fill))S.opt.fill="none";
 S.opt.joins=S.opt.joins?1:0;
 S.opt.colSolo=S.opt.colSolo?1:0;
 S.rb=Object.assign(d.rb,S.rb||{});
 /* المفاتيح الجديدة تأخذ افتراضها من d.rb — فالملفّ القديم
    يبقى على سلوكه: شبكةٌ تُرسَم وتُلتقَط ومساراتٌ تُرى */
 ["grid","gsnap","paths","dyn"].forEach(k=>{S.rb[k]=S.rb[k]?1:0});
 S.os=Object.assign(d.os,S.os||{});
 S.os.ref=(S.os.ref==null)?1:(S.os.ref?1:0);
 S.pol=Object.assign(d.pol,S.pol||{});
 S.pol.inc=clamp(parseInt(S.pol.inc,10)||15,1,90);
 if(!Array.isArray(S.pol.extra))S.pol.extra=[];
 if(S.rb.polar&&S.rb.ortho)S.rb.ortho=0;

 /* ═══ الجدران ═══ */
 const WT={ext:1,int:1,low:1}, AL={c:1,l:1,r:1};
 S.walls=S.walls.filter(w=>w&&Array.isArray(w.a)&&Array.isArray(w.b)
  &&isFinite(w.a[0])&&isFinite(w.b[1]));
 S.walls.forEach(w=>{
  if(!WT[w.type])w.type="int";
  if(!AL[w.align])w.align="c";
  const df=(w.type==="ext")?S.meta.tExt
   :((w.type==="low")?S.meta.tLow:S.meta.tInt);
  w.t=clamp(Math.round(+w.t||df),50,1000);
  w.a=PT(w.a); w.b=PT(w.b);
  if(w.type==="low")w.h=Math.max(200,Math.round(+w.h||S.meta.lowH));
  else delete w.h;
  /* القوس: bulge اختياريّ · خارج الحدّ يُقسَر · الصفر يُطرَح
     فيبقى الجدار مستقيماً بلا حقلٍ زائد */
  if(isFinite(+w.bulge)&&Math.abs(+w.bulge)>1e-4)
   w.bulge=clamp(+w.bulge,-8,8);
  else delete w.bulge;
 });
 /* ═══ الفتحات ═══
    تُحذف اليتيمة — حاضنها زال. ولا يُقلَّم موضعها:
    الخارجة عن مدى جدارها تُبلَّغ ولا تُصلَح.
    والحدود العليا هي حدود المُثبِّتات نفسها (batch.js): قيدٌ
    بمنفذَين يُخرج قيمةً لا تُرى ثم تمنع تعديل جدارها. */
 const OKV={door:1,double:1,sliding:1,window:1,fixed:1,
  opening:1,arch:1,niche:1};
 const WMAP=new Map(S.walls.map(w=>[w.id,w]));
 S.opens=S.opens.filter(o=>o&&WMAP.has(o.wall));
 S.opens.forEach(o=>{
  if(!OKV[o.kind])o.kind="door";
  o.s=Math.round(+o.s||0);
  o.w=Math.max(100,Math.round(+o.w||900));
  o.h=clamp(Math.round(+o.h||2100),100,6000);
  o.sill=clamp(Math.round(+o.sill||0),0,6000);
  o.hinge=(o.hinge==="end")?"end":"start";
  o.swing=(o.swing==="right")?"right":"left";
  if(o.pan!=null)o.pan=clamp(Math.round(o.pan),1,6);
  if(o.dep!=null){
   /* الجدار موجودٌ يقيناً: اليتيمة حُذفت قبل هذا السطر */
   const W2=WMAP.get(o.wall);
   const mx=Math.max(20,((W2&&W2.t)||150)-40);
   o.dep=clamp(Math.round(o.dep),20,mx);
  }
  if(o.face)o.face=(o.face==="r")?"r":"l";
 });
 /* ═══ المناطق ═══
    حلقات مخزَّنة. لا تُعاد حساباً ولا تُقلَّم — الاتّساق
    يُبلَّغ عنه بالبصمة في areas.js، ولا يُصلَح خلسة. */
 S.areas=S.areas.filter(a=>a&&Array.isArray(a.ring));
 S.areas.forEach(a=>{
  a.ring=a.ring
   .filter(p=>Array.isArray(p)&&isFinite(p[0])&&isFinite(p[1]))
   .map(PT);
  a.name=String(a.name==null?"":a.name).slice(0,40);
  a.stamp=String(a.stamp||"");
  a.showArea=a.showArea?1:0;
  if(!/^(none|tint|hatch)$/.test(a.fill))a.fill="tint";
  if(a.lp&&isFinite(a.lp[0])&&isFinite(a.lp[1]))a.lp=PT(a.lp);
  else delete a.lp;
 });
 S.areas=S.areas.filter(a=>a.ring.length>2);

 /* ═══ الأبعاد ═══
    نقطتان صريحتان وموضعُ خطٍّ صريح. لا ترتبط بجدار،
    فلا تُحذَف ولا تُزحَف — «المعلَّق» يُبلَّغ عنه في dims.js. */
 S.dims=S.dims.filter(x=>x&&Array.isArray(x.a)&&Array.isArray(x.b)
  &&isFinite(x.a[0])&&isFinite(x.b[1]));
 S.dims.forEach(x=>{
  if(!/^(h|v|al)$/.test(x.kind))x.kind="h";
  x.a=PT(x.a); x.b=PT(x.b);
  x.pos=Math.round(+x.pos||0);
  if(x.txt!=null){
   x.txt=String(x.txt).slice(0,24);
   if(!x.txt)delete x.txt;
  }
 });
 /* ═══ السلاسل ═══ قيَم مكتوبة — لا تُطابَق ولا تُصحَّح */
 S.chains=S.chains.filter(c=>c&&Array.isArray(c.base)
  &&Array.isArray(c.vals));
 S.chains.forEach(c=>{
  c.axis=(c.axis==="v")?"v":"h";
  c.base=PT(c.base);
  c.pos=Math.round(+c.pos||0);
  c.vals=c.vals.map(v=>Math.max(10,Math.round(+v||0))).slice(0,60);
  c.total=c.total?1:0;
 });
 S.chains=S.chains.filter(c=>c.vals.length);

 /* ═══ النصوص والقوائد والمناسيب ═══ */
 const AKV={text:1,lead:1,level:1};
 S.anno=S.anno.filter(a=>a&&AKV[a.kind]);
 S.anno.forEach(a=>{
  if(a.kind==="lead"){
   a.pts=(Array.isArray(a.pts)?a.pts:[])
    .filter(p=>Array.isArray(p)&&isFinite(p[0])&&isFinite(p[1]))
    .map(PT);
  }else{
   a.x=Math.round(+a.x||0);
   a.y=Math.round(+a.y||0);
  }
  if(a.kind==="level"){
   a.z=Math.round(+a.z||0);
   a.pre=String(a.pre==null?"":a.pre).slice(0,8);
  }else{
   a.s=String(a.s==null?"":a.s).slice(0,120);
   a.hm=clamp(+a.hm||1,0.4,6);
  }
  if(a.kind==="text"){
   a.rot=deg(+a.rot||0);
   if(!/^(bl|bc|ml|mc)$/.test(a.al))a.al="bc";
  }
 });
 S.anno=S.anno.filter(a=>(a.kind==="lead")
  ? (a.pts.length>1&&a.s)
  : (a.kind==="level"?true:!!a.s));

 /* ═══ الأعمدة ═══ الدمج عرضٌ لا تعديل، فلا يُخزَّن منه شيء */
 S.cols=S.cols.filter(c=>c&&isFinite(c.x)&&isFinite(c.y));
 S.cols.forEach(c=>{
  c.kind=(c.kind==="circ")?"circ":"rect";
  c.x=Math.round(c.x); c.y=Math.round(c.y);
  c.w=clamp(Math.round(+c.w||300),100,4000);
  c.h=(c.kind==="circ")?c.w:clamp(Math.round(+c.h||c.w),100,4000);
  c.rot=(c.kind==="circ")?0:deg(+c.rot||0);
  if(!/^(conc|steel|stone)$/.test(c.type))c.type="conc";
  if(c.tag!=null){
   c.tag=String(c.tag).slice(0,10);
   if(!c.tag)delete c.tag;
  }
 });
 /* ═══ الأدوات الصحية ═══ إحداثيات صريحة — لا رابطة تُحفَظ */
 const FKV={wc:1,bidet:1,ur:1,lav:1,sink:1,shower:1,tub:1,wm:1,fd:1};
 S.fixt=S.fixt.filter(f=>f&&FKV[f.kind]
  &&isFinite(f.x)&&isFinite(f.y));
 S.fixt.forEach(f=>{
  f.x=Math.round(f.x); f.y=Math.round(f.y);
  f.rot=deg(+f.rot||0);
  f.w=clamp(Math.round(+f.w||400),80,4000);
  f.d=clamp(Math.round(+f.d||400),80,4000);
  if(f.mir)f.mir=1; else delete f.mir;
 });
 /* ═══ الدرج ═══ a و b و n هي الأصل، وما عداها مشتقّ */
 S.stairs=S.stairs.filter(s=>s&&Array.isArray(s.a)&&Array.isArray(s.b)
  &&isFinite(s.a[0])&&isFinite(s.b[1]));
 S.stairs.forEach(s=>{
  s.a=PT(s.a); s.b=PT(s.b);
  s.w=clamp(Math.round(+s.w||1000),600,6000);
  s.n=clamp(Math.round(+s.n||12),2,80);
  s.up=(s.up==="dn")?"dn":"up";
  s.cut=clamp(+s.cut||0,0,0.95);
  if(s.h!=null){
   s.h=clamp(Math.round(s.h),200,8000);
   if(!s.h)delete s.h;
  }
 });
 /* ═══ الورقة والعنوان ═══ */
 S.sheet=Object.assign(d.sheet,S.sheet||{});
 if(!/^A[0-4]$/.test(S.sheet.size))S.sheet.size="A3";
 S.sheet.orient=(S.sheet.orient==="p")?"p":"l";
 S.sheet.margin=clamp(+S.sheet.margin||12,0,60);
 S.sheet.on=S.sheet.on?1:0;
 S.sheet.tb=S.sheet.tb?1:0;
 S.sheet.north=S.sheet.north?1:0;
 ["cx","cy"].forEach(k=>{
  S.sheet[k]=(S.sheet[k]!=null&&isFinite(S.sheet[k]))
   ? Math.round(S.sheet[k]) : null;
 });
 S.title=Object.assign(d.title,S.title||{});
 ["proj","owner","loc","sheet","rev","by"].forEach(k=>{
  S.title[k]=String(S.title[k]==null?"":S.title[k]).slice(0,60);
 });
 /* ═══ المرجع ═══ جامد: يُطبَّع شكلاً ولا يُصلَح هندسةً ═══
    والحدود هي حدود القارئ نفسها (io/dxfin.js): ملفُّ مشروعٍ
    محرَّرٌ يدوياً منفذٌ ثانٍ إلى الحالة، فلا يُترَك بلا سقف —
    مضلّعٌ بعشرة ملايين رأسٍ يمرّ من هنا كما يمرّ من هناك.
    وهويّة مصفوفة الكيانات تُحفَظ إن لم يُنبَذ منها شيء: مخزنُ
    نسخ المرجع (REFS) يوازن بالهويّة، فإعادةُ بناءٍ بلا سببٍ
    تُنشئ نسخةً زائدة مع كل تراجع، وثمانيةُ تراجعاتٍ تُزحِم
    الحدَّ فتُفقَد نسخةٌ يشير إليها تاريخٌ قائم. */
 const CO=1e9, RPTS=20000;
 const okp=p=>Array.isArray(p)&&isFinite(p[0])&&isFinite(p[1])
  &&Math.abs(p[0])<=CO&&Math.abs(p[1])<=CO;
 S.ref=Object.assign(d.ref,S.ref||{});
 S.ref.tr=Object.assign({k:1,rot:0,dx:0,dy:0},S.ref.tr||{});
 S.ref.tr.k=clamp(+S.ref.tr.k||1,1e-4,1e4);
 S.ref.tr.rot=deg(+S.ref.tr.rot||0);
 S.ref.tr.dx=Math.round(+S.ref.tr.dx||0);
 S.ref.tr.dy=Math.round(+S.ref.tr.dy||0);
 const RT={l:1,p:1,a:1,t:1,x:1};
 const RE=Array.isArray(S.ref.ents)?S.ref.ents:[];
 let RK=RE.filter(e=>e&&RT[e.t]);
 if(RK.length>60000)RK=RK.slice(0,60000);
 RK.forEach(e=>{
  e.sl=String(e.sl==null?"0":e.sl).slice(0,80)||"0";
  if(e.t==="l"){e.a=PT(e.a); e.b=PT(e.b)}
  else if(e.t==="p")e.pts=(Array.isArray(e.pts)?e.pts:[])
   .filter(p=>Array.isArray(p)&&isFinite(p[0])&&isFinite(p[1]))
   .slice(0,RPTS)          /* السقف نفسه: MAXPTS في القارئ */
   .map(PT);
  else if(e.t==="a"){
   e.c=PT(e.c);
   /* قيمةٌ خارج المدى تُقسَر: صندوقٌ هائل يصفّر التكبير وتخلو
      الشاشة بلا رسالة */
   e.r=clamp(Math.round(+e.r||1),1,CO);
   e.a0=deg(+e.a0||0); e.a1=deg(+e.a1||0);
  }
  else if(e.t==="t"){
   e.p=PT(e.p);
   e.s=String(e.s==null?"":e.s).slice(0,200);
   e.h=clamp(Math.round(+e.h||100),1,1e7);
   e.rot=deg(+e.rot||0);
  }else e.p=PT(e.p);
 });
 RK=RK.filter(e=>e.t!=="p"||e.pts.length>1)
      .filter(e=>e.t!=="t"||e.s)
      /* الطبقة الثالثة: PT يعيد [0,0] لما لا يُفهَم، فالنقطة
         المعطوبة تصير أصلاً ولا تُنبَذ — والفلترة هنا تنبذها */
      .filter(e=>{
       if(e.t==="l")return okp(e.a)&&okp(e.b);
       if(e.t==="p")return true;         /* رؤوسه مُقصَّاة أعلاه */
       if(e.t==="a")return okp(e.c);
       return okp(e.p);
      });
 S.ref.ents=(RK.length===RE.length)?RE:RK;
 S.ref.src={};
 S.ref.ents.forEach(e=>{
  S.ref.src[e.sl]=(S.ref.src[e.sl]||0)+1});
 const OF={};
 Object.keys(S.ref.off||{}).forEach(k=>{
  if(S.ref.src[k])OF[k]=1});
 S.ref.off=OF;

 COLLS.forEach(k=>S[k].forEach(e=>bumpIdc(idNum(e.id))));
 return S;
}
/* ═══ المرجع خارج اللقطة ═══
   كيانات المرجع جامدةٌ بالتصميم: لا يتغيّر منها إلّا tr و off.
   فتسلسلُها في كل خطوةِ تراجعٍ كان يضاعف كلفة كل نقرةٍ بحجم
   الملفّ المستورد — ستّون ألف كيانٍ في مئة خطوة، وثلاث مئة
   ميغابايت في الذاكرة، ومقارنةُ نصٍّ بحجم ميغابايت في pushHistory.
   والكيانات تتبدّل بحدثَين صريحَين فقط: setRef و clearRef.

   الحلّ مشاركةٌ بنيوية: الخطوات تحمل رقم نسخةٍ، والمحتوى في مخزنٍ
   واحد. وأمّا tr و off فيدخلان اللقطة كاملَين — فمحاذاةٌ واحدةٌ
   تُتراجَع عنها.

   عدّادان لا واحد: ver تصاعديٌّ لا يعود، وcur نسخةُ الحالة الآن.
   والفصل لازم — التراجع يعيد cur إلى نسخةٍ سابقة، ولو أعاد
   الترقيم معها لكتب استيرادٌ جديدٌ فوق نسخةٍ يشير إليها تاريخٌ
   قائم. */
const RCAP=8;
let refVer=0, refCur=0;
const REFS=new Map();
let ONREF=null;
export const setRefLost=f=>{ONREF=(typeof f==="function")?f:null};
export const refVersion=()=>refCur;
export const refStore=()=>({n:REFS.size,ver:refVer,cur:refCur});

function refTrim(){
 /* لا تنمو بلا حدّ: أقدمُ نسخةٍ لا تشير إليها الحالة تُنسى.
    والمرجع الواحد نسختان في العادة (قبل الاستيراد وبعده). */
 while(REFS.size>RCAP){
  let old=null;
  for(const k of REFS.keys()){if(k!==refCur){old=k; break}}
  if(old==null)break;
  REFS.delete(old);
 }
}
export function refBump(){
 refVer++; refCur=refVer;
 REFS.set(refCur,(S.ref&&S.ref.ents)||[]);
 refTrim();
 return refCur;
}
/* ═══ التاريخ ═══ */
const MAX=100, HIST={u:[],r:[]};
/* تسمياتٌ موازية لسجلّ التراجع — اختياريّة، لا تُغيّر توقيع من
   ينادي pushHistory(snap) بلا تسمية. لوحة السجل المرئية (انظر
   ui/historypanel.js) تقرأ منها. */
const HLBL={u:[],r:[]};
const DEF_LBL="تعديل";
export function pack(){
 const o={};
 KEYS.forEach(k=>{o[k]=S[k]});
 o._idc=idc();
 return o;
}
export function snapshot(){
 const o=pack();
 const ents=(o.ref&&o.ref.ents)||[];
 if(ents.length){
  /* الهويّة هي الميزان: مصفوفةٌ جديدة تعني محتوىً جديداً — وقد
     تأتي من استيرادٍ لم يُعلن نسخته، أو من فتح ملفّ. */
  if(REFS.get(refCur)!==ents)refBump();
  o.ref=Object.assign({},o.ref,{ents:[],__rv:refCur});
 }
 return JSON.stringify(o);
}
function apply(d){
 if(!d)return;
 KEYS.forEach(k=>{if(d[k]!==undefined)S[k]=d[k]});
 /* لقطةٌ كاملة: كل شيء تبدّل يقيناً — الجدران والفتحات والطبقات.
    والكاشات المفتاحيّة (بصمة المناطق) تتّكل على هذا. */
 VER.g++; VER.o++;
 /* استرجاع الكيانات من مخزن النسخ. وإن ضاعت النسخة (تجاوزت
    الحدّ) فالمرجع يزول ويُبلَّغ — ولا تُخترَع كياناتٌ.
    والمصفوفة مشتركةٌ مع المخزن: لا مسارَ يُدخل فيها أو يُخرج،
    فsetRef وclearRef يستبدلانها استبدالاً. */
 if(d.ref&&d.ref.__rv!=null){
  const e=REFS.get(d.ref.__rv);
  S.ref=Object.assign({},d.ref,{ents:e||[]});
  delete S.ref.__rv;
  refCur=d.ref.__rv;
  if(!e&&ONREF)ONREF(d.ref.__rv);
 }
 setIdc(d._idc||0);
 ensureShape();
}
export function pushHistory(snap,label){
 if(!snap)return;
 if(HIST.u[HIST.u.length-1]===snap)return;
 HIST.u.push(snap); HLBL.u.push(label||DEF_LBL);
 if(HIST.u.length>MAX){HIST.u.shift(); HLBL.u.shift()}
 HIST.r.length=0; HLBL.r.length=0;
}
export const canUndo=()=>HIST.u.length>0;
export const canRedo=()=>HIST.r.length>0;
export const clearHistory=()=>{
 HIST.u.length=0; HIST.r.length=0; HLBL.u.length=0; HLBL.r.length=0;
};
/* ═══ الخط الزمني — للوحة السجل المرئية ═══
   past: من الأقدم إلى الأحدث · future: ما أُعيد التراجع عنه، من
   الأقرب إلى الأبعد. current = عدد خطوات past (موضع المؤشّر). */
export function historyTimeline(){
 return {past:HLBL.u.slice(), future:HLBL.r.slice().reverse(),
  current:HLBL.u.length};
}
/* ينقل المؤشّر إلى الخطوة n (0=البداية). يُنادي undo()/redo()
   الحقيقيّتين خطوةً خطوة فيبقى التخزين والحفظ التلقائي سليمَين. */
export function historyJumpTo(n){
 const total=HIST.u.length+HIST.r.length;
 n=Math.max(0,Math.min(n,total));
 while(HIST.u.length>n){if(!undo())break}
 while(HIST.u.length<n){if(!redo())break}
}

let AFTER=()=>{}, ONERR=()=>{};
export const setAfterEdit=f=>{AFTER=(typeof f==="function")?f:(()=>{})};
export const setEditError=f=>{ONERR=(typeof f==="function")?f:(()=>{})};

export function undo(){
 if(!HIST.u.length)return false;
 const cur=snapshot();
 apply(JSON.parse(HIST.u.pop()));
 const lbl=HLBL.u.pop()||DEF_LBL;
 HIST.r.push(cur); HLBL.r.push(lbl);
 if(HIST.r.length>MAX){HIST.r.shift(); HLBL.r.shift()}
 touch(); AFTER(true); autosave();
 return true;
}
export function redo(){
 if(!HIST.r.length)return false;
 const cur=snapshot();
 apply(JSON.parse(HIST.r.pop()));
 const lbl=HLBL.r.pop()||DEF_LBL;
 HIST.u.push(cur); HLBL.u.push(lbl);
 if(HIST.u.length>MAX){HIST.u.shift(); HLBL.u.shift()}
 touch(); AFTER(true); autosave();
 return true;
}
/* تعديل ذرّي: الفشل يُرجَع إلى اللقطة ويُبلَّغ — لا حالة نصف معدَّلة */
let FAILED=false;
/* هل أخفق آخر edit()؟ — يُسأل مباشرةً بعده وحده.
   لم نُعِد قيمةً مميّزة لأن كل مستدعٍ يقرأ العائد قيمةً شرعية،
   فأيُّ رمزٍ نعيده يصير صادقاً في شرطٍ قائم. */
export const editFailed=()=>FAILED;

export function edit(fn,label){
 const sn=snapshot();
 FAILED=false;
 let out=null;
 try{out=fn()}
 catch(e){
  apply(JSON.parse(sn));
  touch(); AFTER(true);
  FAILED=true;
  ONERR((e&&e.message)?e.message:String(e));
  return undefined;
 }
 pushHistory(sn,label); touch(); AFTER(false); autosave();
 return out;
}
/* ═══ الحفظ التلقائي ═══
   يُنادى متزامناً من كل تعديل، ويكتب لاحقاً. والكتابةُ الجارية لا
   تُقاطَع: ما يقع أثناءها يُعلَّم ويُكتَب بعدها — فلا تُفقَد آخر
   لمسة، ولا تتزاحم معاملتان على السجلّ نفسه. */
let AT=null, BUSY=false, DIRTY=false, SEALED=false;
let ONSAVE=()=>{};
export const setSaveError=f=>{
 ONSAVE=(typeof f==="function")?f:(()=>{});
};
async function flush(){
 AT=null;
 if(SEALED)return;              /* saveNow كتبت الأخيرة — لا تُسبَق */
 if(BUSY){DIRTY=true; return}
 if(!DIRTY)return;
 DIRTY=false; BUSY=true;
 let r;
 try{r=await Store.save(pack())}
 catch(e){r={ok:0,via:"?",err:(e&&e.message)||String(e)}}
 BUSY=false;
 if(SEALED)return;              /* أُغلق الباب أثناء الكتابة */
 if(!r.ok)ONSAVE(r);
 if(DIRTY&&!AT)AT=setTimeout(flush,700);
}
export function autosave(){
 DIRTY=true;
 SEALED=false;              /* تعديلٌ جديد: الصفحة عادت */
 if(AT)clearTimeout(AT);
 AT=setTimeout(flush,700);
}
/* ═══ الكتابة الأخيرة ═══
   تُنادى عند الإغلاق والإخفاء. تكتب متزامناً وتُغلق الباب: كتابةٌ
   آجلة كانت قيد التنفيذ تحمل لقطةً أقدم، فلو أكملت بعدها لكتبت
   فوق الأحدث — وهو الفقد نفسه الذي بُنيت هذه الدالّة لمنعه. */
export function saveNow(){
 if(AT){clearTimeout(AT); AT=null}
 if(!DIRTY&&!BUSY){SEALED=true; return {ok:1,via:"skip"}}
 DIRTY=false; SEALED=true;
 return Store.flushSync(pack());
}
/* الصفحة عادت (visibilitychange ⇒ visible): يُفتَح الباب */
export function saveResume(){
 if(!SEALED)return false;
 SEALED=false;
 if(DIRTY)autosave();
 return true;
}
export function loadState(d,resetHist){
 apply(d||DEF());
 if(resetHist)clearHistory();
 touch();
 return S;
}
export function newState(){
 setIdc(0);
 loadState(DEF(),true);
 DIRTY=false; SEALED=false;
 if(AT){clearTimeout(AT); AT=null}
 Store.del();
 AFTER(true);
 return S;
}
/* ═══ الاستعادة ═══
   تعيد اسم المصدر ("idb" · "ls" · "migrate") أو كائناً
   {via:"healed",refs} أو false. والسلسلة صادقة، فمن كان يفحص
   صحّتها يبقى على حاله — والمستدعي يقرأ had.via أو had. */
export async function restore(){
 let r=null;
 try{r=await Store.load()}
 catch(e){return false}
 if(!r||!r.data||!Array.isArray(r.data.walls))return false;
 loadState(r.data,true);
 /* الترميم والهجرة يُثبَّتان في IndexedDB فوراً، فلا يُعاد
    الترميم في كل إقلاع */
 if(r.via==="migrate"||r.via==="healed")autosave();
 return (r.via==="healed")
  ? {via:"healed",refs:r.refs||0} : r.via;
}
```

<a id="f-js-core-templates-js"></a>

---

## `js/core/layers.js`

```javascript
/* ═══ الطبقات: الجدول الحيّ والسياسة ═══
   الجدول بياناتُ مشروع: يُحفَظ في الملفّ ويدخل التاريخ، لأن إخفاء
   طبقةٍ أو وزنَ خطّها يغيّر ما يُصدَّر — فهو حالةُ رسمٍ لا حالة
   نافذة.

   وresolve هي المكسب الأكبر: مصدرٌ واحد للون والوزن والنوع
   والشفافية يقرأه القماش والمصدِّرون الأربعة. كان لكلٍّ نسخته
   (LAYERS.css · theme.PRINT · LAYERS.c)، وكانت تنجرف.

   والهندسة لا تُخفى: regionLoops (في render.js) لا يقرأ هذا
   الملفّ أصلاً. */
import {S,VER,touch} from "./state.js";
import {LAYERS as BASE,PRN,LT,LWS,ORDER,DESC,AUX,
        DEFLAYS,layRow,ltOf,lwOk} from "./laydef.js";
import {ENT,ORD} from "./entreg.js";

export {LT,LWS,AUX,ltOf,lwOk,DESC};
export const HEX=/^#[0-9a-fA-F]{6}$/;
export const isInternal=L=>/^__/.test(String(L||""));

/* ═══ الوصول ═══ فهرسٌ بالاسم مُكاشٌ على النسخة الهندسية ═══
   جدول الطبقات بياناتُ مشروع: كلُّ كاتبٍ فيه ينادي invalidate
   صريحاً (setLay · showAll · isolate · stateApply · resetLays)،
   ولقطةُ التاريخ تُقدّم النسخة الهندسية. فالمفتاح g لا n —
   وكان n يُفرِغ كاش الألوان في كل إطارٍ أثناء سحب بُعد، وهو
   يُنادى لكل أوّلية. */
let IX=null, IXV=-1;
function index(){
 if(IXV===VER.g&&IX)return IX;
 IX=new Map();
 (S.layers||[]).forEach(l=>IX.set(l.n,l));
 IXV=VER.g;
 return IX;
}
export const LAYS=()=>S.layers||[];
export const layNames=()=>LAYS().map(l=>l.n);
export const layOf=n=>index().get(n)||null;
export const hasLay=n=>index().has(n);
export const layLabel=n=>{
 const l=layOf(n);
 return l?(l.d||n):(DESC[n]||n);
};
export const LNAME=layLabel;   /* توافقٌ لمن كان يستوردها بهذا الاسم */

/* ═══ السياسة ═══
   الطبقة المجهولة تُرى ولا تُقفَل: كيانٌ على طبقةٍ حُذفت لا يُختفي
   صامتاً — يبقى مرئياً حتى تقرّر فيه. والتشخيص الداخلي (__) يُرى
   دائماً ولا يُقفَل أبداً. */
export const vis=n=>{
 if(isInternal(n))return true;
 const l=layOf(n);
 return l?!l.off:true;
};
export const locked=n=>{
 if(isInternal(n))return false;
 const l=layOf(n);
 return l?!!l.lk:false;
};
export const plots=n=>{
 if(isInternal(n))return false;
 const l=layOf(n);
 return l?!!l.plot:true;
};
export function layOfEnt(s){
 if(!s)return null;
 const d=ENT[s.k];
 if(!d)return null;
 const e=d.byId(s.id);
 return e?d.lay(e):null;
}
/* ما لا يُحدَّد لا يُعدَّل ولا يُحذَف — حرسٌ في موضعٍ واحد */
export function pickable(s){
 const L=layOfEnt(s);
 if(!L)return true;
 return vis(L)&&!locked(L);
}
export const entVis=s=>{
 const L=layOfEnt(s);
 return L?vis(L):false;
};
export const entLocked=s=>{
 const L=layOfEnt(s);
 return L?locked(L):false;
};
export const anyHidden=()=>LAYS().some(l=>l.off);
export const anyLocked=()=>LAYS().some(l=>l.lk);
export const hiddenLayers=()=>LAYS().filter(l=>l.off).map(l=>l.n);
export const lockedLayers=()=>LAYS().filter(l=>l.lk).map(l=>l.n);
export const noPlotLayers=()=>LAYS().filter(l=>!l.off&&!l.plot)
 .map(l=>l.n);

/* ═══ العدّ ═══ */
export function layCounts(){
 const c={};
 const inc=L=>{if(L)c[L]=(c[L]||0)+1};
 ORD.forEach(d=>(S[d.coll]||[]).forEach(e=>inc(d.lay(e))));
 c["A-GRID"]=S.grid.xs.length+S.grid.ys.length;
 c["A-WALL-PATT"]=(S.opt.fill!=="none")?(c["A-WALL"]||0):0;
 c["A-REFR"]=(S.ref&&S.ref.ents)?S.ref.ents.length:0;
 c["A-SHET"]=+S.sheet.on?1:0;
 return c;
}
/* عدد الكيانات المستثناة من العرض والتصدير */
export function hiddenCount(){
 const c=layCounts();
 let n=0;
 LAYS().forEach(l=>{
  if(l.off&&l.n!=="A-WALL-PATT")n+=(c[l.n]||0);
 });
 return n;
}
export function noPlotCount(){
 const c=layCounts();
 let n=0;
 noPlotLayers().forEach(L=>{
  if(L==="A-WALL-PATT")return;
  n+=(c[L]||0);
 });
 return n;
}
/* الأوّليات المرئيّة وحدها — الترتيب يبقى كما بُني */
export const filterPrims=P=>(P||[]).filter(g=>vis(g.L||"0"));

/* ═══ resolve ═══
   mode: dark | light | plot
   يعيد ما يحتاجه الرسم والتصدير معاً:
     css    لونٌ نصّي
     aci    رقم DXF
     lw     وزنٌ بمئات المليمتر (0 = افتراضي)
     lt     مفتاح النوع · dxf اسمه · dash شُرَطُه بالمليمتر الورقي
     op     شفافية ٠–٩٠٪ · a معامل الرسم
     off · lk · plot
   ومُكاشٌ على النسخة والوضع: يُنادى لكل أوّليةٍ في كل إطار. */
const RC=new Map();
let RCV=-1;
function fresh(){
 if(RCV!==VER.g){RC.clear(); RCV=VER.g}
 return RC;
}
/* ثابتٌ واحدٌ يُعاد لكل طبقةٍ مجهولة — مجمَّدٌ لئلا يُفسِده مَن
   يعدّله ظنّاً أنه نسخةٌ خاصّة به؛ وله n كما للمسار السليم، وإلا
   عادت resolve(L).n بـundefined للمجهولة وحدها. */
const MISS=Object.freeze({n:null,css:"#e8eef4",aci:7,lw:0,lt:"solid",
 dxf:"CONTINUOUS",dash:[],op:0,a:1,off:0,lk:0,plot:1,miss:1});

export function resolve(n,mode){
 const m=(mode==="plot"||mode==="light")?mode:"dark";
 const k=m+"|"+n;
 const C=fresh();
 const hit=C.get(k);
 if(hit)return hit;
 const l=layOf(n);
 if(!l){C.set(k,MISS); return MISS}
 const t=ltOf(l.lt);
 const op=Math.max(0,Math.min(90,+l.op||0));
 const r={
  n,
  css:(m==="dark")?l.col:l.pcol,
  aci:l.aci, lw:l.lw|0,
  lt:l.lt, dxf:t.dxf, dash:t.mm,
  op, a:1-op/100,
  off:!!l.off, lk:!!l.lk, plot:!!l.plot};
 C.set(k,r);
 return r;
}
/* ═══ نسخة الجدول ═══
   تصفيةُ الأوّليات تتبع جدولَ الطبقات وحده، وكلُّ كاتبٍ فيه يُنادي
   invalidate صريحاً (setLay · showAll · isolate · plotAll ·
   stateApply · resetLays · normLays). فلو صُفِّيت على VER.g لأُعيدت
   تصفيةُ ستّين ألف أوّليةٍ مرجعية في كل إطارٍ من سحب جدار. */
let LV=1;
export const layVer=()=>LV;
export const invalidate=()=>{LV++; RCV=-1; IXV=-1};

/* ═══ الكتابة ═══
   لا تُنادى داخل edit(): المستدعي يلفّها كما يلفّ أي تعديل، فتدخل
   التاريخَ خطوةً واحدة ولو غيّرتَ عدّة حقول. */
const FLD={
 col:v=>HEX.test(String(v))?String(v).toLowerCase():null,
 pcol:v=>HEX.test(String(v))?String(v).toLowerCase():null,
 aci:v=>{const n=Math.round(+v); return (n>=0&&n<=256)?n:null},
 lw:v=>lwOk(v),
 lt:v=>LT[v]?v:null,
 op:v=>{const n=Math.round(+v||0); return Math.max(0,Math.min(90,n))},
 off:v=>(v?1:0), lk:v=>(v?1:0), plot:v=>(v?1:0),
 d:v=>String(v==null?"":v).slice(0,48)
};
export function setLay(n,f,v){
 const l=layOf(n);
 if(!l)return false;
 const fn=FLD[f];
 if(!fn)return false;
 const nv=fn(v);
 if(nv===null||nv===undefined)return false;
 /* القفل على طبقةٍ مساعدة بلا معنى: لا كياناتَ تُحدَّد عليها */
 if(f==="lk"&&AUX.has(n))return false;
 if(l[f]===nv)return true;
 l[f]=nv;
 invalidate();
 touch();
 return true;
}
export function showAll(){
 let n=0;
 LAYS().forEach(l=>{if(l.off){l.off=0; n++}});
 if(n){invalidate(); touch()}
 return n;
}
export function unlockAll(){
 let n=0;
 LAYS().forEach(l=>{if(l.lk){l.lk=0; n++}});
 if(n){invalidate(); touch()}
 return n;
}
export function plotAll(){
 let n=0;
 LAYS().forEach(l=>{if(!l.plot){l.plot=1; n++}});
 if(n){invalidate(); touch()}
 return n;
}
/* عزل: يُخفي ما عدا المذكورة — ويُعاد بـ showAll */
export function isolate(keep){
 const K=new Set(Array.isArray(keep)?keep:[keep]);
 let n=0;
 LAYS().forEach(l=>{
  const off=K.has(l.n)?0:1;
  if(l.off!==off){l.off=off; n++}
 });
 if(n){invalidate(); touch()}
 return n;
}
export function resetLays(){
 S.layers=DEFLAYS();
 invalidate(); touch();
 return S.layers.length;
}
export const toggleOff =L=>{const r=setLay(L,"off",vis(L)?1:0); return r};
export const toggleLock=L=>{const r=setLay(L,"lk",locked(L)?0:1); return r};

/* ═══ حالات الطبقات ═══
   لقطةٌ مسمّاة للحقول التي يُحتمَل تبديلها بين مراحل العمل. لا
   تحمل الوصف ولا الاسم: هيئةٌ لا هويّة. */
const SNAPF=["off","lk","plot","col","pcol","lw","lt","op"];
export const layStates=()=>Object.keys(S.layst||{});
export function stateSave(name){
 const nm=String(name||"").trim().slice(0,32);
 if(!nm)return false;
 S.layst=S.layst||{};
 const o={};
 LAYS().forEach(l=>{
  const r={};
  SNAPF.forEach(f=>{r[f]=l[f]});
  o[l.n]=r;
 });
 S.layst[nm]=o;
 touch();
 return nm;
}
export function stateApply(name){
 const o=(S.layst||{})[name];
 if(!o)return 0;
 let n=0;
 LAYS().forEach(l=>{
  const r=o[l.n];
  if(!r)return;
  SNAPF.forEach(f=>{
   if(r[f]===undefined)return;
   const nv=FLD[f]?FLD[f](r[f]):r[f];
   if(nv===null||nv===undefined)return;
   if(l[f]!==nv){l[f]=nv; n++}
  });
 });
 if(n){invalidate(); touch()}
 return n;
}
export function stateDel(name){
 if(!S.layst||!S.layst[name])return false;
 delete S.layst[name];
 touch();
 return true;
}
/* ═══ التطبيع ═══ يُنادى من ensureShape ═══
   جدولٌ محرَّرٌ يدوياً أو من إصدارٍ أقدم يُصلَح شكلاً: الناقص يُستكمل
   من مصنعه، والمجهول يُنبَذ، والترتيب يبقى كما حفظه المستخدم. */
export function normLays(){
 const def=DEFLAYS();
 const D=new Map(def.map(l=>[l.n,l]));
 const src=Array.isArray(S.layers)?S.layers:[];
 const out=[], seen=new Set();
 src.forEach(raw=>{
  if(!raw||typeof raw!=="object")return;
  const n=String(raw.n||"");
  if(!D.has(n)||seen.has(n))return;   /* مجهولةٌ أو مكرّرة */
  seen.add(n);
  const base=D.get(n), l={n};
  Object.keys(FLD).forEach(f=>{
   const v=(raw[f]===undefined)?base[f]:FLD[f](raw[f]);
   l[f]=(v===null||v===undefined)?base[f]:v;
  });
  if(AUX.has(n))l.lk=0;
  out.push(l);
 });
 /* المفقودة تُضاف في موضعها من الترتيب المصنعي */
 def.forEach((l,i)=>{
  if(seen.has(l.n))return;
  const at=out.findIndex(x=>def.findIndex(y=>y.n===x.n)>i);
  if(at<0)out.push(l); else out.splice(at,0,l);
 });
 S.layers=out;
 /* ═══ الهجرة ═══
    S.lay القديم كان {اسم:{off,lock}} — يُطوى في الجدول ثم يُطرَح.
    ويُقرأ مرّةً واحدة، فمشروعٌ محفوظٌ قبل هذه الخطوة يفتح على
    حالة طبقاته نفسها لا على المصنع. */
 if(S.lay&&typeof S.lay==="object"){
  const IX2=new Map(S.layers.map(l=>[l.n,l]));
  Object.keys(S.lay).forEach(n=>{
   const l=IX2.get(n);
   const o=S.lay[n];
   if(!l||!o||typeof o!=="object")return;
   if(o.off!==undefined)l.off=o.off?1:0;
   if(o.lock!==undefined&&!AUX.has(n))l.lk=o.lock?1:0;
  });
  delete S.lay;
 }
 if(!S.layst||typeof S.layst!=="object")S.layst={};
 Object.keys(S.layst).forEach(k=>{
  if(!S.layst[k]||typeof S.layst[k]!=="object")delete S.layst[k];
 });
 invalidate();
 return S.layers.length;
}
```

<a id="f-js-core-modify-js"></a>

---

## `js/core/laydef.js`

```javascript
/* ═══ تعريف الطبقات: المصنع ═══
   جدولٌ واحد يحمل ما كان متفرّقاً في موضعين: لون الشاشة الداكنة
   ورقم ACI ووزن الخطّ في LAYERS بـ state.js، ولون الورق في PRINT
   بـ theme.js.

   ولونان لا لونٌ واحد بقصد: الجدار على شاشةٍ داكنة قريبٌ من
   الأبيض، وعلى الورق أسود. ليس انجرافاً بل عكسُ خلفيةٍ — فلو
   اشتُقّ أحدهما من الآخر بمعادلةٍ لتغيّر مخرَجُك اليوم.

   وهو ورقةٌ في شجرة الاعتماد: لا يستورد شيئاً، فيستورده state.js
   بلا دورة. */

/* c لون ACI للـ DXF · lw وزن الخط ×100 مم · css لون الشاشة —
   حرفاً بحرف كما كانت في state.js. */
const BASE={
 "A-WALL":     {c:7, lw:50, css:"#e8eef4"},
 "A-WALL-PATT":{c:8, lw:13, css:"#6d7987"},
 "A-WALL-LOW": {c:8, lw:18, css:"#95a3b5"},
 "A-COLS":     {c:6, lw:50, css:"#d9a3e8"},
 "A-DOOR":     {c:3, lw:25, css:"#7fe0a6"},
 "A-GLAZ":     {c:4, lw:25, css:"#86ccf0"},
 "A-FIXT":     {c:5, lw:18, css:"#a4b6f0"},
 "A-STRS":     {c:5, lw:25, css:"#8fa6f0"},
 "A-ELEV":     {c:7, lw:50, css:"#e8eef4"},
 "A-SECT":     {c:7, lw:50, css:"#e8eef4"},
 "A-AREA":     {c:2, lw:18, css:"#e8c46a"},
 "A-DIMS":     {c:1, lw:13, css:"#f09a9a"},
 "A-ANNO":     {c:2, lw:18, css:"#d9c489"},
 "A-GRID":     {c:8, lw:9,  css:"#5d6876"},
 "A-SHET":     {c:7, lw:35, css:"#9daab4"},
 "A-REFR":     {c:8, lw:9,  css:"#69737f"}
};
export {BASE as LAYERS};   /* من كان يستورد LAYERS يبقى عاملاً */

/* ═══ ألوان الورق ═══ منقولةٌ من theme.js — وحُذفت من هناك ═══ */
export const PRN={
 "A-WALL":"#000000","A-WALL-PATT":"#6a6a6a","A-WALL-LOW":"#555555",
 "A-COLS":"#4a2d5c","A-DOOR":"#1a6b3c","A-GLAZ":"#12557f",
 "A-FIXT":"#3a4a7a","A-STRS":"#3a4a7a","A-ELEV":"#000000",
 "A-SECT":"#000000",
 "A-AREA":"#8a6d1f",
 "A-DIMS":"#a02020","A-ANNO":"#7a5f14","A-GRID":"#7a7a7a",
 "A-SHET":"#000000","A-REFR":"#999999"
};
/* ═══ أنواع الخطوط ═══
   الشُّرَط بالمليمتر على الورق — كارتفاع النصّ (txtMM)، فتُضرَب
   بالمقياس لتصير وحداتِ رسم. وdxf هو اسم النوع في DXF نفسه.

   وكلّها solid في المصنع: القدرة أُضيفت والمخرَج لم يتغيّر. */
export const LT={
 solid:  {n:"متّصل",         dxf:"CONTINUOUS", mm:[]},
 dash:   {n:"مشروح",         dxf:"DASHED",     mm:[6,3]},
 hidden: {n:"مخفيّ",          dxf:"HIDDEN",     mm:[3,2]},
 center: {n:"محوري",         dxf:"CENTER",     mm:[12,2,2,2]},
 dashdot:{n:"شرطة ونقطة",    dxf:"DASHDOT",    mm:[8,2,0.2,2]},
 dot:    {n:"منقّط",          dxf:"DOT",        mm:[0.2,3]}
};
export const ltOf=k=>LT[k]||LT.solid;

/* ═══ أوزان الخطّ ═══ بمئات المليمتر كما في DXF ═══ */
export const LWS=[
 [0,"افتراضي"],[5,"0.05"],[9,"0.09"],[13,"0.13"],[15,"0.15"],
 [18,"0.18"],[20,"0.20"],[25,"0.25"],[30,"0.30"],[35,"0.35"],
 [40,"0.40"],[50,"0.50"],[60,"0.60"],[70,"0.70"],[80,"0.80"],
 [100,"1.00"],[120,"1.20"],[140,"1.40"],[200,"2.00"]
];
export const lwOk=v=>{
 const n=Math.round(+v||0);
 let best=0, bd=1/0;
 LWS.forEach(([w])=>{
  const d=Math.abs(w-n);
  if(d<bd){bd=d; best=w}
 });
 return best;
};
/* ═══ الترتيب والوصف ═══
   الترتيب للعرض: الإنشائي أوّلاً ثم الفتحات ثم التأشير ثم المساعد.
   والوصف يُغني عن حفظ رموزٍ إنجليزية. */
export const ORDER=[
 "A-WALL","A-WALL-PATT","A-WALL-LOW","A-COLS",
 "A-DOOR","A-GLAZ","A-STRS","A-FIXT","A-ELEV","A-SECT",
 "A-AREA","A-DIMS","A-ANNO",
 "A-GRID","A-SHET","A-REFR"
];
export const DESC={
 "A-WALL":"الجدران","A-WALL-PATT":"تعبئة الجدران",
 "A-WALL-LOW":"السَّتَر والجدران المنخفضة","A-COLS":"الأعمدة",
 "A-DOOR":"الأبواب","A-GLAZ":"الشبابيك والفتحات",
 "A-STRS":"الدرج","A-FIXT":"الأدوات الصحية","A-ELEV":"الواجهات",
 "A-SECT":"المقاطع",
 "A-AREA":"المناطق والمساحات","A-DIMS":"الأبعاد والسلاسل",
 "A-ANNO":"النصوص والقوائد","A-GRID":"المحاور",
 "A-SHET":"الورقة وبلوك العنوان","A-REFR":"المرجع المستورد"
};
/* الطبقات المساعدة لا كياناتَ لها: تُرسَم ولا تُحدَّد، فلا قفلَ
   لها ولا معنى — ويُعطَّل مفتاحه في المدير */
export const AUX=new Set(["A-WALL-PATT","A-GRID","A-SHET","A-REFR"]);

/* ═══ سطرٌ واحد ═══ */
export const layRow=n=>{
 const b=BASE[n]||{};
 return {n,
  col:b.css||"#e8eef4",     /* الشاشة الداكنة — كما هو اليوم */
  pcol:PRN[n]||"#000000",   /* الورق والشاشة الفاتحة */
  aci:(b.c===undefined)?7:b.c,
  lw:lwOk(b.lw||0),
  lt:"solid",
  op:0,                     /* شفافية ٠–٩٠٪ */
  off:0, lk:0,
  /* المرجع المستورد لا يُطبَع: خلفيةٌ للرسم لا جزءٌ من اللوحة.
     وهذا تغيُّرٌ في المخرَج — نقرةٌ في المدير تعيده. */
  plot:(n==="A-REFR")?0:1,
  d:DESC[n]||n};
};
export const DEFLAYS=()=>{
 const seen=new Set(ORDER);
 const rest=Object.keys(BASE).filter(n=>!seen.has(n));
 return ORDER.concat(rest).filter(n=>BASE[n]).map(layRow);
};
```

<a id="f-js-core-layers-js"></a>

---

## `js/core/journal.js`

````javascript
/* ═══ سجلّ الأوامر ═══
   نسخة نصية أمينة لما يُدخل المستخدم. العمليات التي لا يمكن تمثيلها
   بسطر إدخال تُسجّل كشائبة بدلاً من أن توهم بإعادة مطابقة. */
export const JR={lines:[],taint:[],max:4000};
let MUTE=0;

export const jrMute=v=>{MUTE=v?1:0};
export function jrAdd(s){
 if(MUTE)return;
 s=String(s==null?"":s).trim();
 if(!s)return;
 if(s==="esc"&&JR.lines[JR.lines.length-1]==="esc")return;
 JR.lines.push(s);
 if(JR.lines.length>JR.max)JR.lines.shift();
}
export function jrTaint(why){
 if(MUTE)return;
 const at=JR.lines.length;
 const last=JR.taint[JR.taint.length-1];
 if(last&&last.why===why&&last.at===at)return;
 JR.taint.push({at,why:String(why||"عملية غير قابلة للتمثيل")});
 if(JR.taint.length>200)JR.taint.shift();
}
export const jrClear=()=>{JR.lines.length=0;JR.taint.length=0};
export const jrCount=()=>JR.lines.length;
export const jrTainted=()=>JR.taint.length;
export function jrText(){
 const H=[`# سجلّ CivilDraft — ${JR.lines.length} سطراً`];
 if(JR.taint.length){
  const w=[...new Set(JR.taint.map(t=>t.why))].join(" · ");
  H.push(`# مشوب: ${w} — لا يُعبَّر عنها بسطر، فالإعادة تختلف`);
 }
 return H.concat(JR.lines).join("\n");
}
export const jrPlan=()=>"```plan\n"+JR.lines.join("\n")+"\n```";
````

<a id="f-js-core-laydef-js"></a>

---

## `js/core/persist.js`

```javascript
/* ═══ الحفظ التلقائي والاستعادة (وحدة مستقلّة اختيارية) ═══
   يخزّن لقطة الحالة في IndexedDB دورياً وعند التغيّر، ويستعيدها بعد
   التعطّل. محليٌّ بالكامل (لا يُرسَل ولا يُصدَّر). لا يفرض شكلاً على S:
   يتلقّى serialize/restore بالحقن فلا يكسر تغليف الحالة.

   ⚠ ملاحظة تكامل: مشروع مِسطَر يملك بالفعل نظام حفظٍ تلقائي كاملاً
   في core/state.js (pack/snapshot/restore/autosave/saveNow) المبنيّ
   على io/store.js والمربوط في app.js حالياً. هذه الوحدة إضافةٌ
   منفصلة (قاعدة IndexedDB مختلفة) ولم تُربَط تلقائياً بـ app.js
   تفادياً لتشغيل نظامي حفظٍ متنافسين. اربطها يدوياً فقط إن كنت
   تنوي استبدال النظام القائم أو استخدامها لغرضٍ آخر (كمسوَّدات
   منفصلة مثلاً). */
const DB="mistar.autosave", STORE="snapshots", KEY="current", DB_VERSION=1;

function openDB(){
 return new Promise((res,rej)=>{
  if(typeof indexedDB==="undefined"){ rej(new Error("لا IndexedDB")); return; }
  const rq=indexedDB.open(DB,DB_VERSION);
  rq.onupgradeneeded=()=>{ const db=rq.result;
   if(!db.objectStoreNames.contains(STORE)) db.createObjectStore(STORE); };
  rq.onsuccess=()=>res(rq.result);
  rq.onerror=()=>rej(rq.error||new Error("فشل فتح القاعدة"));
 });
}
async function idbPut(key,val){ const db=await openDB();
 return new Promise((res,rej)=>{ const tx=db.transaction(STORE,"readwrite");
  tx.objectStore(STORE).put(val,key);
  tx.oncomplete=()=>{ db.close(); res(true); };
  tx.onerror=()=>{ db.close(); rej(tx.error); }; }); }
async function idbGet(key){ const db=await openDB();
 return new Promise((res,rej)=>{ const tx=db.transaction(STORE,"readonly");
  const rq=tx.objectStore(STORE).get(key);
  rq.onsuccess=()=>{ db.close(); res(rq.result||null); };
  rq.onerror=()=>{ db.close(); rej(rq.error); }; }); }
async function idbDel(key){ const db=await openDB();
 return new Promise((res)=>{ const tx=db.transaction(STORE,"readwrite");
  tx.objectStore(STORE).delete(key);
  tx.oncomplete=()=>{ db.close(); res(true); };
  tx.onerror=()=>{ db.close(); res(false); }; }); }

export function createAutosave(opts={}){
 const serialize=opts.serialize; const restore=opts.restore;
 if(typeof serialize!=="function")
  throw new Error("createAutosave يحتاج serialize()");
 const interval=Math.max(2000, opts.interval||8000);
 let timer=null, dirty=false, last=0, saving=false;

 async function save(force){
  if(saving) return; if(!force && !dirty) return;
  saving=true; dirty=false;
  try{ const snap={ v:1, ts:Date.now(),
    meta:(opts.meta?opts.meta():null), data:serialize() };
   await idbPut(KEY, snap); last=snap.ts; }
  catch(e){ dirty=true; }
  finally{ saving=false; }
 }
 return {
  markDirty(){ dirty=true; },
  saveNow(){ return save(true); },
  start(){ if(timer) return;
   timer=setInterval(()=>save(false), interval);
   window.addEventListener("visibilitychange",()=>{
    if(document.visibilityState==="hidden") save(true); });
   window.addEventListener("beforeunload",()=>{ save(true); }); },
  stop(){ if(timer){ clearInterval(timer); timer=null; } },
  async peek(){ const s=await idbGet(KEY).catch(()=>null);
   return s?{ts:s.ts, meta:s.meta}:null; },
  async recover(){ const s=await idbGet(KEY).catch(()=>null);
   if(!s||!s.data) return false;
   if(typeof restore==="function"){ restore(s.data); return true; }
   return false; },
  clear(){ return idbDel(KEY); },
  get lastSaved(){ return last; }
 };
}
```

<a id="f-js-core-pricing-js"></a>

---

## `js/core/trace.js`

```javascript
/* ═══ استنباط الجدران من الخربشة ═══
   حسابٌ محضٌ لا استشارة: يقرأ ضربات يدك ويعيد «خطّة» — مساراتٍ
   مقترحةً بأطوالها وزواياها وانحرافها عن الشبكة. لا يكتب في الحالة
   ولا يُنشئ جداراً: التطبيق أمرٌ صريح في tools/sketch.js بعد أن ترى
   الخطّة بالمليمتر.

   المراحل: تنظيف · تبسيط RDP · إسقاط القصير · مُدرَّج زوايا يستخرج
   دوران الشبكة السائد · قصّ الزوايا على مضاعفاته · دمج المتطابقات
   المتراكبة · حلّ الأركان (تقاطع محورين · عقدة · وصلة T).

   الأركان تُحَلّ بتقاطع المحورَين لا بمتوسّط النقاط: النقطة الناتجة
   تقع على محور كلٍّ منهما فتبقى الزاوية قائمةً بالضبط، ولا يُقاس
   انحرافٌ صامت. وحيث تعذّر ذلك يُستعمل القطب ويُبلَّغ الانحراف.

   دوال خالصة تُختبَر بلا متصفّح: node js/tests/trace.js            */
import {dist,lineX,nearOnSeg,bboxOf} from "./geom.js";
import {deg,clamp,D2R,R2D} from "./units.js";

const R=v=>Math.round(v);
const angOf=(a,b)=>deg(Math.atan2(b[1]-a[1],b[0]-a[0])*R2D);

/* الحدود تُشتقّ من حجم خربشتك نفسها — فلا رقمٌ سحريّ يفسد عند
   تغيير التكبير. وكلٌّ منها يُتجاوَز صراحةً من شريط الخيارات. */
/* حدٌّ أدنى مطلق لا يتبع قُطر الرسمة: خربشةٌ صغيرة (رقمٌ، إشارة) لها
   حجمٌ مادّيٌّ ثابتٌ تقريباً بصرف النظر عن اتساع اللوحة حولها، فحين
   تكبر الرسمة لا يجوز أن يكبر معها حدّ «هل هذه ضجيج؟» فيسمح لخربشةٍ
   صغيرة بالمرور. النسبيّ (أدناه) يبقى لضبط الرسومات الصغيرة جداً. */
const MIN_STROKE_ABS=800;
export function autoOpt(diag,opt){
 const d=Math.max(1000,+diag||10000);
 return Object.assign({
  eps      : Math.max(40, d*0.012),   /* تفاوت التبسيط */
  minSeg   : Math.max(200,d*0.050),   /* أقصر مسار يُقبَل */
  minStroke: Math.max(150,d*0.030,MIN_STROKE_ABS), /* أصغر ضربة ليست ضجيجاً */
  angTol   : 18,                      /* أقصى قصٍّ زاويّ */
  nodeTol  : Math.max(200,d*0.070),   /* تقارب الأطراف */
  offTol   : Math.max(150,d*0.045),   /* تقارب المتوازيات */
  gap      : Math.max(200,d*0.060)    /* فجوة الدمج على المحور */
 },opt||{});
}
/* ═══ التنظيف والتبسيط ═══ */
export function cleanStroke(P,tol){
 const T=Math.max(1,tol||8), o=[];
 (P||[]).forEach(p=>{
  if(!p||!isFinite(p[0])||!isFinite(p[1]))return;
  const q=o[o.length-1];
  const r=[R(p[0]),R(p[1])];
  if(q&&dist(q,r)<T)return;
  o.push(r);
 });
 return o;
}
/* Ramer–Douglas–Peucker — يحفظ الأركان ويُسقط الرجفة */
export function rdp(P,eps){
 if(!P||P.length<3)return (P||[]).slice();
 const keep=new Array(P.length).fill(false);
 keep[0]=keep[P.length-1]=true;
 const st=[[0,P.length-1]];
 let guard=0;
 while(st.length&&guard++<20000){
  const [i,j]=st.pop();
  if(j-i<2)continue;
  let bi=-1, bd=-1;
  for(let k=i+1;k<j;k++){
   const d=nearOnSeg(P[i],P[j],P[k][0],P[k][1]).d;
   if(d>bd){bd=d;bi=k}
  }
  if(bd>eps&&bi>0){keep[bi]=true; st.push([i,bi],[bi,j])}
 }
 return P.filter((p,i)=>keep[i]);
}
const mkSeg=(a,b,snapped,merged)=>({
 a:[R(a[0]),R(a[1])], b:[R(b[0]),R(b[1])],
 L:dist(a,b), ang:angOf(a,b),
 dev:0, snapped:snapped?1:0, merged:merged||1});

/* ═══ دوران الشبكة السائد ═══
   مُدرَّج بدرجةٍ واحدة موزونٌ بالطول، ثم متوسّطٌ موزون داخل النافذة
   ليعطي كسور الدرجة. الالتفاف حول ٩٠ محسوبٌ في الحالتين. */
export function gridAngle(segs){
 if(!segs||!segs.length)return 0;
 const B=new Array(90).fill(0);
 segs.forEach(s=>{B[R(s.ang)%90]+=s.L});
 let bi=0, bv=-1;
 for(let i=0;i<90;i++){
  let v=0;
  for(let d=-2;d<=2;d++)v+=B[(i+d+90)%90]*(3-Math.abs(d))/3;
  if(v>bv){bv=v;bi=i}
 }
 let w=0, m=0;
 segs.forEach(s=>{
  let d=(s.ang%90)-bi;
  while(d>45)d-=90;
  while(d<-45)d+=90;
  if(Math.abs(d)>4)return;
  w+=s.L; m+=s.L*d;
 });
 return ((bi+(w?m/w:0))%90+90)%90;
}
/* القصّ حول منتصف المسار: الطول والمركز محفوظان، الزاوية وحدها
   تُصحَّح — فلا يزحف مسارٌ عن موضع رسمك. */
export function snapAngles(segs,rot,tol){
 (segs||[]).forEach(s=>{
  let best=null, bd=1/0;
  for(let k=0;k<4;k++){
   const t=deg(rot+k*90);
   let d=Math.abs(t-s.ang);
   if(d>180)d=360-d;
   if(d<bd){bd=d;best=t}
  }
  s.dev=Math.round(bd*100)/100;
  if(bd>tol)return;                    /* حرّ — يُبلَّغ ولا يُقصّ */
  const mx=(s.a[0]+s.b[0])/2, my=(s.a[1]+s.b[1])/2;
  const ux=Math.cos(best*D2R), uy=Math.sin(best*D2R);
  s.a=[R(mx-ux*s.L/2), R(my-uy*s.L/2)];
  s.b=[R(mx+ux*s.L/2), R(my+uy*s.L/2)];
  s.ang=best; s.snapped=1; s.dev=0;
 });
 return segs;
}
/* ═══ دمج المتطابقات ═══
   الضربة المزدوجة على الجدار نفسه، والركن المرسوم مرّتين. يُعمَل على
   المقصوص وحده: الحرّ لا يُدمَج لأن زاويته غير موثوقة. */
function mergeGroup(arr,offTol,gap){
 const t=arr[0].ang;
 const u=[Math.cos(t*D2R),Math.sin(t*D2R)];
 const n=[-u[1],u[0]];
 const M=arr.map(s=>{
  const t0=u[0]*s.a[0]+u[1]*s.a[1];
  const t1=u[0]*s.b[0]+u[1]*s.b[1];
  return {L:s.L, off:n[0]*s.a[0]+n[1]*s.a[1],
   lo:Math.min(t0,t1), hi:Math.max(t0,t1)};
 });
 M.sort((p,q)=>p.off-q.off||p.lo-q.lo);
 const bins=[];
 M.forEach(m=>{
  const b=bins.find(x=>Math.abs(x.off-m.off)<=offTol);
  if(b){
   b.off=(b.off*b.w+m.off*m.L)/(b.w+m.L);
   b.w+=m.L; b.items.push(m);
  }else bins.push({off:m.off,w:m.L,items:[m]});
 });
 const out=[];
 bins.forEach(b=>{
  b.items.sort((p,q)=>p.lo-q.lo);
  let cur=null;
  b.items.forEach(m=>{
   if(cur&&m.lo-cur.hi<=gap){
    cur.hi=Math.max(cur.hi,m.hi); cur.n++; return;
   }
   if(cur)out.push(cur);
   cur={lo:m.lo,hi:m.hi,n:1};
  });
  if(cur)out.push(cur);
  out.forEach(c=>{if(c.off==null)c.off=b.off});
 });
 return out.map(c=>{
  const A=[c.off*n[0]+c.lo*u[0], c.off*n[1]+c.lo*u[1]];
  const B=[c.off*n[0]+c.hi*u[0], c.off*n[1]+c.hi*u[1]];
  const s=mkSeg(A,B,1,c.n);
  s.ang=t;
  return s;
 });
}
export function mergeCollinear(segs,offTol,gap){
 const free=(segs||[]).filter(s=>!s.snapped);
 const G=new Map();
 (segs||[]).filter(s=>s.snapped).forEach(s=>{
  const k=R(s.ang*2);                  /* نصف درجة */
  if(!G.has(k))G.set(k,[]);
  G.get(k).push(s);
 });
 let out=[];
 G.forEach(arr=>{out=out.concat(mergeGroup(arr,offTol,gap))});
 return out.concat(free);
}
/* ═══ حلّ الأركان ═══ */
export function joinNodes(segs,tol,ext){
 const E=[];
 (segs||[]).forEach((s,i)=>{
  E.push({i,k:"a",p:s.a}); E.push({i,k:"b",p:s.b});
 });
 const used=new Array(E.length).fill(false);
 const joined=new Array(E.length).fill(false);
 const stat={node:0,axis:0,tee:0};

 for(let i=0;i<E.length;i++){
  if(used[i])continue;
  const cl=[i]; used[i]=true;
  for(let j=i+1;j<E.length;j++){
   if(used[j]||E[j].i===E[i].i)continue;
   if(!cl.some(x=>dist(E[x].p,E[j].p)<=tol))continue;
   used[j]=true; cl.push(j);
  }
  if(cl.length<2)continue;
  let P=null;
  if(cl.length===2){
   const A=segs[E[cl[0]].i], B=segs[E[cl[1]].i];
   const par=(Math.abs(A.ang-B.ang)%180)<1;
   if(A.snapped&&B.snapped&&!par){
    const x=lineX(A.a,A.b,B.a,B.b);
    if(x&&dist(x,E[cl[0]].p)<=tol*2.5){P=x; stat.axis++}
   }
  }
  if(!P){
   P=[cl.reduce((s,x)=>s+E[x].p[0],0)/cl.length,
      cl.reduce((s,x)=>s+E[x].p[1],0)/cl.length];
   stat.node++;
  }
  const q=[R(P[0]),R(P[1])];
  cl.forEach(x=>{
   const s=segs[E[x].i];
   if(E[x].k==="a")s.a=q.slice(); else s.b=q.slice();
   E[x].p=q; joined[x]=true;
  });
 }
 /* وصلة T: طرفٌ وحيد يستند إلى جسم مسارٍ آخر */
 const EX=Math.max(1,ext||tol);
 E.forEach((e,x)=>{
  if(joined[x])return;
  const s=segs[e.i];
  let best=null, bd=tol;
  segs.forEach((o,j)=>{
   if(j===e.i)return;
   const par=(Math.abs(s.ang-o.ang)%180)<1;
   const r=nearOnSeg(o.a,o.b,e.p[0],e.p[1]);
   if(r.d>bd)return;
   let q=r.p;
   if(s.snapped&&o.snapped&&!par){
    const ix=lineX(s.a,s.b,o.a,o.b);
    if(ix&&nearOnSeg(o.a,o.b,ix[0],ix[1]).d<=EX
     &&dist(ix,e.p)<=tol)q=ix;
   }
   bd=r.d; best=q;
  });
  if(!best)return;
  const q=[R(best[0]),R(best[1])];
  if(e.k==="a")s.a=q; else s.b=q;
  e.p=q; stat.tee++;
 });
 return stat;
}
/* ═══ المدخل ═══ */
export function trace(strokes,opt){
 const raw=(strokes||[]).map(s=>cleanStroke(s,8))
  .filter(s=>s.length>1);
 const all=[];
 raw.forEach(s=>s.forEach(p=>all.push(p)));
 const bb=bboxOf(all);
 const diag=bb?Math.hypot(bb.x1-bb.x0,bb.y1-bb.y0):0;
 const O=autoOpt(diag,opt);
 const stat={strokes:raw.length,pts:all.length,
  noise:0,short:0,segs:0,free:0,merged:0,
  node:0,axis:0,tee:0,tiny:0};

 /* دوران الشبكة يُستخرَج من قِطَع RDP الخام (المُسقَط قصيرها فقط، بلا
    جسر) — عيّنةٌ كثيفة تُنصِّت الرجفة بالمتوسّط الموزون. الجسر أدناه
    يُبنى لاحقاً على هذا الدوران نفسه فلا يتأثّر برأيه الخاص. */
 let voteSegs=[];
 raw.forEach(P=>{
  const b=bboxOf(P);
  if(!b||Math.hypot(b.x1-b.x0,b.y1-b.y0)<O.minStroke)return;
  const Q=rdp(P,O.eps);
  for(let i=0;i<Q.length-1;i++){
   if(dist(Q[i],Q[i+1])<O.minSeg)continue;
   voteSegs.push(mkSeg(Q[i],Q[i+1],0,1));
  }
 });
 const rot=gridAngle(voteSegs);

 let segs=[];
 raw.forEach(P=>{
  const b=bboxOf(P);
  if(!b||Math.hypot(b.x1-b.x0,b.y1-b.y0)<O.minStroke){
   stat.noise++; return;
  }
  let Q=rdp(P,O.eps);
  /* اجسر رؤوس RDP الداخلية التي تُنتج قطعةً أقصر من minSeg بدل إسقاطها:
     رجفةٌ وسطية تقسم ضلعاً واحداً لا يجوز أن تكسر استمراريته. الطرفان
     (بداية الضربة ونهايتها) لا يُمسّان — القصّ يبقى داخلياً فقط. */
  let changed=true;
  while(changed&&Q.length>2){
   changed=false;
   for(let i=1;i<Q.length-1;i++){
    if(dist(Q[i-1],Q[i])<O.minSeg||dist(Q[i],Q[i+1])<O.minSeg){
     stat.short++;
     Q=Q.slice(0,i).concat(Q.slice(i+1));
     changed=true; break;
    }
   }
  }
  for(let i=0;i<Q.length-1;i++){
   const L=dist(Q[i],Q[i+1]);
   if(L<O.minSeg){stat.short++; continue}
   segs.push(mkSeg(Q[i],Q[i+1],0,1));
  }
 });
 snapAngles(segs,rot,O.angTol);
 const before=segs.length;
 segs=mergeCollinear(segs,O.offTol,O.gap);
 stat.merged=Math.max(0,before-segs.length);
 const js=joinNodes(segs,O.nodeTol,O.gap);
 stat.node=js.node; stat.axis=js.axis; stat.tee=js.tee;

 /* إعادة القياس بعد اللحم، ثم إسقاط ما تلاشى */
 segs.forEach(s=>{
  s.L=dist(s.a,s.b);
  s.ang=angOf(s.a,s.b);
  let bd=1/0;
  for(let k=0;k<4;k++){
   let d=Math.abs(deg(rot+k*90)-s.ang);
   if(d>180)d=360-d;
   if(d<bd)bd=d;
  }
  s.dev=Math.round(bd*100)/100;
 });
 /* المقصوص على الشبكة زاويته موثوقة فيكفيه حدٌّ أدنى صغير بعد اللحم؛
    الحرّ غير المقصوص لم يثبت انتماءه لمحورٍ فيبقى محكوماً بـ minSeg كاملاً */
 const keep=segs.filter(s=>s.L>=(s.snapped?Math.min(O.minSeg,200):O.minSeg));
 stat.tiny=segs.length-keep.length;
 stat.segs=keep.length;
 stat.free=keep.filter(s=>!s.snapped).length;
 keep.sort((p,q)=>q.L-p.L);
 return {segs:keep, rot, stat, opt:O, bbox:bboxOf(all)};
}
/* ═══ المعايرة ═══
   الخربشة بلا مقياس. تُعطي طولاً تعرفه لأطول مسار فتُضرَب الخطّة
   كلّها حول مركزها — تحويلٌ متشابه واحد، لا تعديلٌ متفرّق. */
export function scalePlan(segs,k,c){
 const K=+k||1;
 if(!(K>0)||Math.abs(K-1)<1e-9)return segs;
 const P=[];
 (segs||[]).forEach(s=>{P.push(s.a); P.push(s.b)});
 const b=bboxOf(P);
 const o=c||(b?[(b.x0+b.x1)/2,(b.y0+b.y1)/2]:[0,0]);
 const M=p=>[R(o[0]+(p[0]-o[0])*K), R(o[1]+(p[1]-o[1])*K)];
 (segs||[]).forEach(s=>{
  s.a=M(s.a); s.b=M(s.b);
  s.L=dist(s.a,s.b);
 });
 return segs;
}
export const snapPts=(segs,step)=>{
 const st=Math.max(1,step||1);
 let mx=0;
 const Q=p=>{
  const q=[Math.round(p[0]/st)*st, Math.round(p[1]/st)*st];
  mx=Math.max(mx,dist(p,q));
  return q;
 };
 (segs||[]).forEach(s=>{
  s.a=Q(s.a); s.b=Q(s.b);
  s.L=dist(s.a,s.b);
  s.ang=angOf(s.a,s.b);
 });
 return R(mx);
};
export const planSay=P=>{
 const t=P.stat;
 return `${t.segs} مساراً من ${t.strokes} ضربة`
  +` · دوران الشبكة ${P.rot.toFixed(2)}°`
  +(t.free?` · ${t.free} حرّاً لم يُقصّ`:"")
  +(t.merged?` · دُمج ${t.merged}`:"")
  +(t.short?` · تُخطّي ${t.short} قصيراً`:"")
  +(t.noise?` · ${t.noise} ضربة ضجيج`:"")
  +(t.tiny?` · تلاشى ${t.tiny} باللحم`:"");
};
export const cornerSay=P=>
 `الأركان: ${P.stat.axis} تقاطع محورَين · ${P.stat.node} عقدة`
 +` · ${P.stat.tee} وصلة T`;
```

<a id="f-js-core-underlay-js"></a>

---

## `js/core/sindex.js`

```javascript
/* ═══ فهرس المكان ═══
   شبكة خلايا بعرض مترين — كخلايا refGrid نفسها، وبعقد الكاش نفسه:
   تُبنى مرّةً لكل نسخة حالة (VER.n) فلا استنتاجَ يُعاد.

   وعقدٌ صريح: الفهرس يرشّح ولا يقرّر.
   يعيد المرشَّحين مرتَّبين بترتيب مجموعتهم في الحالة، لا بترتيب
   الخلايا — فترجيح التعادل يبقى كما كان: أوّلُ ما في المصفوفة
   يفوز، تماماً كالحلقة التي كانت تمسحها كلّها. ولو رتّبناهم
   بالخلايا لتبدّل ما يُصاب عند التراكب بلا سببٍ ظاهر.

   والكيان الضخم لا يُحشَر في مئة خليّة: يذهب إلى قائمة «الكبار»
   وتُفحَص مع كل سؤال. */
import {S,VER} from "./state.js";
import {bboxOf,bboxUnion,bboxHit} from "./geom.js";
import {ORD,ENT} from "./entreg.js";

export const CELL=2000;   /* مترَان */
const MAXC=48;            /* أكبر عددٍ من الخلايا لكيانٍ واحد */
const MAXQ=4096;          /* فوقه: المسح أرخص من عدّ الخلايا */

let VN=-1, G=null, QT=0;
const kx=v=>Math.floor(v/CELL);
const key=(cx,cy)=>cx+","+cy;

/* الصندوق يجمع المحيط والشكل ومنطقة الإصابة إن أُعلنت — فلا يفوت
   الفهرسَ موضعٌ قد يُسأل عنه */
function bboxEnt(d,e){
 let B=null;
 if(d.hbox){
  const h=d.hbox(e);
  if(h)B=bboxUnion(B,h);
 }
 if(d.outline){
  const o=d.outline(e);
  if(o&&o.length)B=bboxUnion(B,bboxOf(o));
 }
 if(d.shape){
  const sh=d.shape(e);
  if(sh){
   if(sh.t==="pt")B=bboxUnion(B,bboxOf([sh.p]));
   else if(sh.t==="seg")B=bboxUnion(B,bboxOf([sh.a,sh.b]));
   else if(sh.pts&&sh.pts.length)B=bboxUnion(B,bboxOf(sh.pts));
  }
 }
 return B;
}
function build(){
 const g={cell:new Map(), big:[], rec:{}, all:[], n:0};
 ORD.forEach(d=>{
  const A=S[d.coll]||[];
  const R=g.rec[d.k]=new Array(A.length);
  for(let i=0;i<A.length;i++){
   const e=A[i];
   const r={k:d.k,i,e,b:bboxEnt(d,e),q:0};
   R[i]=r; g.all.push(r); g.n++;
   if(!r.b){g.big.push(r); continue}
   const x0=kx(r.b.x0), x1=kx(r.b.x1);
   const y0=kx(r.b.y0), y1=kx(r.b.y1);
   if((x1-x0+1)*(y1-y0+1)>MAXC){g.big.push(r); continue}
   for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++){
    const k=key(cx,cy);
    let a=g.cell.get(k);
    if(!a){a=[]; g.cell.set(k,a)}
    a.push(r);
   }
  }
 });
 return g;
}
export function grid(){
 if(VN===VER.n&&G)return G;
 G=build(); VN=VER.n;
 return G;
}
export const invalidate=()=>{VN=-1};
export const stats=()=>{
 const g=grid();
 return {n:g.n, cells:g.cell.size, big:g.big.length};
};
export const expand=(b,t)=>({x0:b.x0-t,y0:b.y0-t,
 x1:b.x1+t,y1:b.y1+t});
export const boxAt=(x,y,t)=>({x0:x-t,y0:y-t,x1:x+t,y1:y+t});

/* ═══ السؤال ═══ {نوع: [سجلّ,…]} مرتَّبةً بترتيب المصفوفة ═══ */
export function query(box,kinds){
 const g=grid();
 const want=kinds
  ? new Set(Array.isArray(kinds)?kinds:[kinds]) : null;
 const out={};
 QT++;
 const put=r=>{
  if(r.q===QT)return;
  r.q=QT;
  if(want&&!want.has(r.k))return;
  if(r.b&&!bboxHit(r.b,box,0))return;
  (out[r.k]=out[r.k]||[]).push(r);
 };
 const x0=kx(box.x0), x1=kx(box.x1);
 const y0=kx(box.y0), y1=kx(box.y1);
 if((x1-x0+1)*(y1-y0+1)>MAXQ){
  g.all.forEach(put);
 }else{
  for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++){
   const a=g.cell.get(key(cx,cy));
   if(a)a.forEach(put);
  }
  g.big.forEach(put);
 }
 Object.keys(out).forEach(k=>out[k].sort((a,b)=>a.i-b.i));
 return out;
}
export function entsIn(box,kind){
 const q=query(box,[kind]);
 return (q[kind]||[]).map(r=>r.e);
}
export const entsAt=(x,y,t,kind)=>entsIn(boxAt(x,y,t||0),kind);

/* ═══ الأزواج ═══
   بترتيب i<j نفسه الذي كانت عليه الحلقتان المتداخلتان، فترتيب
   تقارير الفاحص لا يتبدّل. والفحص الدقيق يبقى عند المستدعي. */
export function forPairs(kind,fn,pad){
 const g=grid();
 const A=S[(ENT[kind]||{}).coll]||[];
 const R=g.rec[kind]||[];
 const P=pad||0;
 for(let i=0;i<A.length;i++){
  const r=R[i];
  const C=(r&&r.b)?(query(expand(r.b,P),[kind])[kind]||[]):R;
  for(let j=0;j<C.length;j++){
   const o=C[j];
   if(!o||o.i<=i)continue;
   fn(A[i],A[o.i],i,o.i);
  }
 }
}
```

<a id="f-js-core-stairs-js"></a>

---

## `js/core/batch.js`

```javascript
/* ═══ الحقول: مصدرٌ واحد ═══
   ثلاث قواعد يقوم عليها هذا الملفّ كلّه:

   ١ · لا يُكتب حقلٌ لم تكتبه. القراءة تُخبرك «متعدّد» ولا تُسوّي.
   ٢ · كل كتابةٍ تتحقّق قبل أن تقع، فالرفض لا يترك أثراً نصفياً.
   ٣ · الحقل يُطبَّق على من يملكه، ويُذكَر عدد من تخطّاه.

   ═══ وكان الوصفُ منقسماً ثلاثاً ═══
   FLD تحمل الاسمَ والنوعَ · SET تحمل الحدودَ والرسائلَ والقواعدَ ·
   وprops.js تحمل نسخةً ثالثةً من الحدود في سِمات HTML. فالحدُّ
   يُكتَب ثلاثاً وينجرف، والقاعدةُ المقسورة تسكن مُثبِّتَ حقلٍ آخر
   فتعمل من مدخلٍ وتغيب من مدخل.

   وبعد اليوم: الوصفُ بياناتٌ في FLD، والتحقّقُ في parseVal وحده،
   وما لا يُعبَّر عنه بمدىً يبقى شفرةً في GUARD — وهي ثلاثةٌ من
   سبعةَ عشرَ حقلاً لا أكثر.

   ═══ ومفرداتُ الوصف ═══
   k n t items      قائمةٌ كما هي — لا تتبدّل، فأشكالُ العناصر عقد
   hint             سطرٌ مساعد تحت الحقل
   min max          عددٌ أو دالّةٌ (e)=>عدد — والتابعُ يُنقَل ولا يُنسَخ
   minWhy maxWhy    نصٌّ أو دالّة — الرسالةُ تسمّي السبب لا تُبهِمه
   clip             "min" | "max" | "both" | 0 (رفض)
                    والسياسةُ مستخرَجةٌ من SET نفسها: الحدُّ الثابتُ
                    كان يُقصَر صامتاً (clamp) والتابعُ يُرفَض برسالة —
                    لأن القصرَ إلى قيمةٍ تتبدّل بحاضنها تضليل،
                    والقصرَ إلى ثابتٍ موثَّقٍ تسهيل.
   int wrap step    تدويرٌ · طيُّ زاويةٍ (deg) · خطوةُ العدّاد
   trim             نصٌّ يُقلَّم قبل القصّ
   read write after خطّافاتٌ حيث الحقلُ ليس e[k]=v
   force forceWhy   قاعدةٌ مقسورة تُقرأ من موضعٍ واحد
   accept           قيَمٌ تُقبَل ولا تُعرَض — لِلاأشكلٍ قائمٍ سلفاً    */
import {S,touch} from "./state.js";
import {Mx,Nx,deg,m3,rng3} from "./units.js";
import {entOf,NAME,KORDER,bumpOf,touchFn} from "./ents.js";
import {pickable} from "./layers.js";
import {wallById,wallLen,WTYPE,ALIGN,TMIN,TMAX} from "./walls.js";
import {opensOf,span,allowed,saySpans,OK,OKINDS,okName,
        MINW as OMINW} from "./opens.js";
import {areaById,netArea,FILLS} from "./areas.js";
import {dimById,dimValue,DK} from "./dims.js";
import {colById,CK,CT} from "./cols.js";
import {FK,FKINDS} from "./fixt.js";
import {SMIN_W} from "./stairs.js";

const R=v=>Math.round(v);
/* الحدُّ المحسوب: عددٌ أو دالّةٌ تقرأ الكيان */
const lim=(v,e)=>(typeof v==="function")?v(e):v;
const say=(v,e)=>(typeof v==="function")?v(e):v;
/* items مصفوفةُ أزواج [قيمة, تسمية] — والشكلُ عقدٌ مع quickprops */
const iVals =d=>(d.items||[]).map(x=>x[0]);
const iNames=d=>(d.items||[]).map(x=>x[1]);
/* الاسمُ في الرسالة بلا وحدته: «الارتفاع م: الأقصى ٢٫١٠ م» تكرارٌ */
const NM=d=>String((d&&d.n)||"").replace(/\s*[م°×]$/,"").trim();
/* حدُّ الكوّة من سماكة حاضنها — حسبةٌ واحدةٌ لثلاثة مواضع */
const wallT=o=>{
 const w=wallById(o&&o.wall);
 return (w&&w.t)||150;
};
const depWhy=o=>`المدى ${rng3(20,wallT(o)-40,"م")} في جدار `
 +`${m3(wallT(o))} م`;

/* ═══ وصف الحقول ═══ t: len | num | sel | chk | text ═══
   وقوائمُ العناصر مشتقّةٌ من مصادرها (ALIGN · WTYPE · OK · CK · CT ·
   FK · FILLS · DK) لا منسوخةً — فوسمٌ يتبدّل هناك يتبدّل هنا. */
export const FLD={
 wall:[
  {k:"t",n:"السماكة م",t:"len",min:TMIN,max:TMAX,clip:"both",
   hint:"وتنحيفُه دون كوّةٍ فيه يُرفَض ويُسمّى سببُه"},
  {k:"align",n:"المحاذاة",t:"sel",
   items:Object.keys(ALIGN).map(k=>[k,ALIGN[k]])},
  {k:"type",n:"النوع",t:"sel",
   items:Object.keys(WTYPE).map(k=>[k,WTYPE[k].n]),
   /* أثرٌ لا يقبله e[k]=v — منقولٌ من SET.wall.type */
   after:(w,v)=>{
    if(v==="low"&&w.h==null)w.h=S.meta.lowH;
    if(v!=="low")delete w.h;
   }}],
 open:[
  {k:"kind",n:"النوع",t:"sel",items:OKINDS.map(k=>[k,okName(k)])},
  {k:"w",n:"العرض م",t:"len",min:OMINW,clip:"min",
   hint:"المواضعُ الحرّة تحكمه — والرفضُ يذكرها"},
  {k:"h",n:"الارتفاع م",t:"len",min:100,clip:"min",
   /* منقولةٌ من SET.open.h — وحُذفت هناك، فلا نسختان */
   max:o=>Math.max(200,(+S.meta.wallH||3000)-(+o.sill||0)),
   maxWhy:o=>`مع جلسةٍ ${m3(+o.sill||0)} م من ارتفاع دورٍ `
    +`${m3(+S.meta.wallH||3000)} م`},
  {k:"sill",n:"الجلسة م",t:"len",min:0,clip:"min",
   max:o=>Math.max(0,(+S.meta.wallH||3000)-(+o.h||0)),
   maxWhy:o=>`مع ارتفاعٍ ${m3(+o.h||0)} م من ارتفاع دورٍ `
    +`${m3(+S.meta.wallH||3000)} م`,
   read:o=>o.sill||0,
   /* ═══ خرجت من مُثبِّتَين ═══
      SET.open.kind كانت تكتب o.sill=0 صامتةً، وSET.open.sill
      ترفض برسالة — مصدران لقاعدةٍ واحدة: تعمل عند تبديل النوع
      وتغيب عند كتابة الجلسة، ولا تعرفها اللوحةُ فلا تُقال.
      وهذه واحدةٌ تُقرأ في المسارَين، والقسرُ يُعلَن. */
   force:o=>/^(door|double|sliding)$/.test(o.kind||"")?0:null,
   forceWhy:"الباب جلسته صفر — غيّر نوعه أوّلاً"},
  {k:"swing",n:"جهة الفتح",t:"sel",
   /* بلفظِ اللوحة المفردة الأغنى — كان يفترق بين اللوحتين */
   items:[["left","يسار المسار"],["right","يمينه"]]},
  {k:"dep",n:"عمق الكوّة م",t:"len",clip:0,
   min:20, max:o=>wallT(o)-40,
   minWhy:depWhy, maxWhy:depWhy,
   read:o=>(o.dep!=null)?o.dep:R(wallT(o)*0.45),
   hint:"من وجه الجدار إلى قاع الكوّة"}],
 col:[
  {k:"kind",n:"الشكل",t:"sel",
   items:Object.keys(CK).map(k=>[k,CK[k]]),
   after:(c,v)=>{if(v==="circ"){c.h=c.w; c.rot=0}}},
  {k:"w",n:"العرض / القطر م",t:"len",min:100,max:4000,clip:"both",
   after:c=>{if(c.kind==="circ")c.h=c.w}},
  {k:"h",n:"العمق م",t:"len",min:100,max:4000,clip:"both"},
  {k:"rot",n:"الدوران °",t:"num",wrap:360},
  {k:"type",n:"المادة",t:"sel",
   items:Object.keys(CT).map(k=>[k,CT[k]])}],
 fix:[
  {k:"kind",n:"النوع",t:"sel",items:FKINDS.map(k=>[k,FK[k].n]),
   after:(f,v)=>{f.w=FK[v].w; f.d=FK[v].d}},
  {k:"w",n:"العرض م",t:"len",min:80,max:4000,clip:"both"},
  {k:"d",n:"العمق م",t:"len",min:80,max:4000,clip:"both"},
  {k:"rot",n:"الدوران °",t:"num",wrap:360},
  {k:"mir",n:"معكوسة",t:"chk",
   read:f=>f.mir?1:0,
   write:(f,v)=>{if(v)f.mir=1; else delete f.mir}}],
 stair:[
  {k:"w",n:"العرض م",t:"len",min:SMIN_W,max:6000,clip:"both"},
  {k:"n",n:"عدد القوائم",t:"num",min:2,max:80,clip:"both",int:1,
   hint:"القائمة = ارتفاع الدور ÷ العدد · والنائماتُ قائمةٌ أقلّ"},
  {k:"h",n:"ارتفاع الدور م",t:"len",min:200,max:8000,clip:"both",
   read:t=>(t.h!=null?t.h:S.meta.wallH)},
  {k:"up",n:"الاتجاه",t:"sel",
   items:[["up","صاعد"],["dn","هابط"]]},
  {k:"cut",n:"خطّ القطع",t:"num",min:0,max:0.95,clip:"both",
   step:0.05,hint:"نسبةٌ من الطول — صفرٌ يعني بلا قطع"}],
 area:[
  {k:"name",n:"الاسم",t:"text",max:40,read:a=>a.name||""},
  {k:"fill",n:"التعبئة",t:"sel",
   items:Object.keys(FILLS).map(k=>[k,FILLS[k]])},
  {k:"showArea",n:"أظهر المساحة",t:"chk"}],
 dim:[
  {k:"kind",n:"النوع",t:"sel",
   items:Object.keys(DK).map(k=>[k,DK[k]])},
  {k:"txt",n:"نصّ بديل",t:"text",max:24,trim:1,
   read:d=>d.txt||"",
   write:(d,v)=>{if(v)d.txt=v; else delete d.txt},
   hint:"يُكتَب مكان الرقم المقيس بعلامة * — والمُصدِّر يُبلِّغ عنه"}],
 chain:[
  {k:"total",n:"خطّ المجموع",t:"chk"}],
 anno:[
  {k:"hm",n:"الحجم ×",t:"num",min:0.4,max:6,clip:"both",step:0.1},
  {k:"al",n:"المحاذاة",t:"sel",
   items:[["bc","وسط"],["bl","يسار"],["mc","وسط أوسط"]],
   /* SET.anno.al كانت تقبل ml والقائمةُ لا تعرضه — لاأشكلٌ قائم.
      accept يُبقي القبول ولا يُغيّر ما يُعرَض. أضِف
      ["ml","وسط يسار"] إلى items إن أردتَ عرضه واحذف accept. */
   accept:["ml"]},
  {k:"pre",n:"سابقة المنسوب",t:"text",max:8,read:a=>a.pre||""}]
};
export const fldOf=(k,f)=>(FLD[k]||[]).find(x=>x.k===f)||null;
export const fldName=(k,f)=>{
 const x=fldOf(k,f);
 return x?x.n:f;
};

/* ═══ المُحلّلُ العامّ ═══
   مصدرُ التحقّق الوحيد: يقرأ t وitems وmin/max وclip من FLD ولا
   يعرف نوعَ كيانٍ باسمه. ولا يكتب — فصار الحدُّ قابلاً للسؤال قبل
   الكتابة، وهو ما يجعل اللوحةَ تعرضه بدل أن تُكرّره.
   ويعيد {ok,v} أو {ok:0,why} — والرميُ في applyVal وحدها، فيبقى
   عقدُ applyField كما هو. */
export function parseVal(d,raw,e){
 if(!d)return {ok:0,why:"حقلٌ مجهول"};
 if(d.t==="chk")return {ok:1,
  v:(raw===1||raw===true||raw==="1"||raw==="on"||raw==="true")?1:0};
 if(d.t==="sel"){
  const s=String(raw==null?"":raw);
  if(!iVals(d).some(v=>String(v)===s)&&!(d.accept||[]).includes(s))
   return {ok:0,why:`ليس من: ${iNames(d).join(" · ")}`};
  return {ok:1,v:s};
 }
 if(d.t==="text"){
  let s=String(raw==null?"":raw);
  if(d.trim)s=s.trim();
  const mx=lim(d.max,e);
  return {ok:1,v:(mx!=null)?s.slice(0,mx):s};
 }
 /* Mx للطول (مترٌ ⇒ مليمتر) وNx للعدد — كما في SET حرفاً بحرف،
    فحدُّ القبول والأرقامُ الهندية والفاصلةُ العربية لا تتبدّل. */
 const q=(d.t==="len")?Mx(raw):Nx(raw);
 if(q==null)return {ok:0,
  why:`«${raw}» ${(d.t==="len")?"ليس طولاً":"ليس رقماً"}`};
 if(d.wrap)return {ok:1,v:deg(q)};
 let v=(d.t==="len"||d.int)?R(q):q;
 const mn=lim(d.min,e), mx=lim(d.max,e);
 const fm=n=>(d.t==="len")?`${m3(n)} م`:String(n);
 const cl=d.clip||0;
 if(mn!=null&&v<mn){
  if(cl==="min"||cl==="both")v=mn;
  else{
   const w=say(d.minWhy,e);
   return {ok:0,why:`${NM(d)}: الأدنى ${fm(mn)}`+(w?` — ${w}`:"")};
  }
 }
 if(mx!=null&&v>mx){
  if(cl==="max"||cl==="both")v=mx;
  else{
   const w=say(d.maxWhy,e);
   return {ok:0,why:`${NM(d)}: الأقصى ${fm(mx)}`+(w?` — ${w}`:"")};
  }
 }
 return {ok:1,v};
}
/* ═══ ما لا يُعبَّر عنه بمدىً ═══
   ثلاثةٌ من سبعةَ عشرَ. تُعيد رسالةً أو "" ولا تكتب، وتُنادى بعد
   الكتابة فتقرأ الحالةَ الجديدة، والرفضُ يُرجِع القديمة.
   ورسائلُها منقولةٌ حرفاً بحرف — هي عقدٌ مع المستخدم. */
const GUARD={
 "wall.t":(w,t)=>{
  const n=opensOf(w.id).find(o=>o.kind==="niche"
   &&(+o.dep||0)>t-40);
  return n?`${n.id} كوّة عمقها ${m3(n.dep)} م — `
   +`السماكة ${m3(t)} م لا تكفيها`:"";
 },
 /* allowed(w,nw,o) تستثني o بالهويّة ولا تقرأ o.w، وحلقةُ التراكب
    تتخطّاه كذلك — فالفحصُ بعد الكتابة يطابق ما قبلها. */
 "open.w":(o,nw)=>{
  const w=wallById(o.wall);
  if(!w)return "جدارها غير موجود";
  const A=allowed(w,nw,o);
  if(!A.fits)
   return `${m3(nw)} م لا تتّسع في ${w.id} `
    +`(طوله ${m3(wallLen(w))} م)`;
  if(!A.spans.some(([a,b])=>o.s>=a-1&&o.s<=b+1))
   return `التوسيع إلى ${m3(nw)} م حول موضعها ${m3(o.s)} م `
    +`لا يتّسع — المواضع الحرّة ${saySpans(A)} م`;
  const lo=o.s-nw/2, hi=o.s+nw/2;
  for(const x of opensOf(w.id)){
   if(x===o)continue;
   const [a,b]=span(x);
   if(lo<b-1&&a<hi-1)
    return `التوسيع يصطدم بـ ${x.id} ${okName(x.kind)} `
     +`على ${m3(x.s)} م`;
  }
  return "";
 },
 "dim.kind":d=>(dimValue(d)<10)
  ? `${d.id}: نقطتاه متطابقتان في هذا الاتجاه` : ""
};

/* من يملك الحقل: قرارٌ صريح لا استنتاج من نجاح الكتابة.
   ولا when في FLD بقصد — ownsField هي الحكم، ولو أضفتُها صار
   للسؤال جوابان. ومفاتيحُ SET كانت ≡ مفاتيحَ FLD في كل نوعٍ بلا
   استثناء، فالاحتياطُ يقرأ الجدولَ نفسه. */
export function ownsField(kind,e,field){
 if(kind==="open"&&field==="swing")return !!OK[e.kind].sw;
 if(kind==="open"&&field==="dep")  return e.kind==="niche";
 if(kind==="col" &&field==="h")    return e.kind!=="circ";
 if(kind==="col" &&field==="rot")  return e.kind!=="circ";
 if(kind==="anno"&&field==="al")   return e.kind==="text";
 if(kind==="anno"&&field==="hm")   return e.kind!=="level";
 if(kind==="anno"&&field==="pre")  return e.kind==="level";
 return !!fldOf(kind,field);
}
/* ═══ قراءةُ القيمة ═══ من FLD لا من سلسلةِ شروط ═══
   واللوحتان تقرآن قراءةً واحدة — كان عمقُ الكوّة يُقرأ بدالّتين. */
export function fieldVal(kind,e,field){
 const d=fldOf(kind,field);
 if(d&&typeof d.read==="function")return d.read(e);
 const v=e[field];
 return (v==null)?"":v;
}
/* ═══ الكتابةُ الواحدة ═══
   بديلٌ حرفيٌّ لنداء SET[kind][field](e,raw): تعيد true/false
   (‏false = لا يملكه) وترمي Error على الرفض. */
export function applyVal(kind,field,e,raw){
 const d=fldOf(kind,field);
 if(!d)return false;
 if(!ownsField(kind,e,field))return false;
 const r=parseVal(d,raw,e);
 if(!r.ok)throw new Error(r.why);
 /* القاعدةُ المقسورة على حقلها: رفضٌ لا قسر — طلبتَ ما تمنعه
    القاعدة، والتصريحُ خيرٌ من تبديلٍ صامت. وهي رسالةُ SET نفسها. */
 if(typeof d.force==="function"){
  const f=d.force(e);
  if(f!=null&&String(f)!==String(r.v))
   throw new Error(d.forceWhy||`${NM(d)}: قيمةٌ مقسورة`);
 }
 const had=Object.prototype.hasOwnProperty.call(e,field);
 const old=e[field];
 if(typeof d.write==="function")d.write(e,r.v); else e[field]=r.v;
 const g=GUARD[`${kind}.${field}`];
 if(g){
  const msg=g(e,r.v,old);
  if(msg){
   if(had)e[field]=old; else delete e[field];
   throw new Error(msg);
  }
 }
 if(typeof d.after==="function")d.after(e,r.v,old);
 return true;
}
/* ═══ القواعدُ المقسورة بعد كتابةِ أيِّ حقل ═══
   تُطبَّق حين تُبدِّل نافذةً باباً كما تُطبَّق حين تكتب الجلسة،
   وتُعاد مُعلَنةً فتُقال. وكانت كتابةً صامتةً داخل مُثبِّتٍ آخر. */
export function forceRules(kind,e){
 const out=[];
 (FLD[kind]||[]).forEach(d=>{
  if(typeof d.force!=="function")return;
  if(!ownsField(kind,e,d.k))return;
  const f=d.force(e);
  if(f==null)return;
  const cur=e[d.k];
  if(String(cur==null?"":cur)===String(f))return;
  if(typeof d.write==="function")d.write(e,f); else e[d.k]=f;
  out.push({k:d.k,n:NM(d),t:d.t,v:f,why:d.forceWhy||""});
 });
 return out;
}
/* ═══ التجميع ═══ */
export function groupSel(list){
 const G={};
 (list||[]).forEach(s=>{
  if(!s||!FLD[s.k])return;
  if(!pickable(s))return;                /* المخفيّ والمقفل خارج */
  if(!entOf(s))return;
  (G[s.k]=G[s.k]||[]).push(s);
 });
 return G;
}
export const groupOrder=G=>KORDER.filter(k=>G[k]&&G[k].length);

/* ═══ القراءة ═══
   mixed=1 يعني «متعدّد»: الحقل يُعرَض فارغاً ولا يُكتب من تلقائه.
   وvalue:null نوعٌ لا يتصادم بنصٍّ حقيقيّ — فلا سلسلةَ خاصّة. */
export function readField(kind,list,field){
 if(!fldOf(kind,field))return {mixed:0,value:null,own:0,n:0};
 let v=null, first=1, mixed=0, own=0;
 (list||[]).forEach(s=>{
  const e=entOf(s);
  if(!e||!ownsField(kind,e,field))return;
  own++;
  const cur=fieldVal(kind,e,field);
  if(first){v=cur; first=0}
  else if(String(cur)!==String(v))mixed=1;
 });
 return {mixed,value:mixed?null:v,own,n:(list||[]).length};
}
/* ═══ التطبيق ═══
   خطوة تراجع واحدة. ما قُبل يُكتَب، وما رُفض يُذكَر باسمه وسببه،
   وما لا يملك الحقل يُعَدّ ولا يُلام. */
export function applyField(kind,list,field,raw,opt){
 const O=opt||{};
 const d=fldOf(kind,field);
 if(!d)throw new Error(`لا حقل «${field}» في ${NAME[kind]||kind}`);
 /* ═══ الفراغُ معنيان ═══
    في اللوحة الجماعية «لا تكتب»، وفي المفردة «اكتب فراغاً» (مسحُ
    اسمِ منطقةٍ أو نصٍّ بديل). ومظهرٌ واحدٌ لمعنيين لا يُحسَم
    بالتخمين ولا بسلوكٍ مخفيٍّ في مستمع الحدث: وسيطٌ صريحٌ في موضع
    النداء — فيلزم modify.js وai/ops كما يلزم اللوحة.
    ومسحُ نصٍّ جماعياً يحتاج زرّاً لا معنىً ثانياً للفراغ. */
 if(O.skipBlank&&(raw===""||raw==null))
  return {field, kind, done:0, ids:[], refused:[],
   noown:0, skipped:0, forced:[], blank:1};
 const done=[], ref=[], noown=[], gone=[], forced=[];
 (list||[]).forEach(s=>{
  if(!pickable(s)){gone.push(s.id); return}
  const e=entOf(s);
  if(!e){gone.push(s.id); return}
  if(!ownsField(kind,e,field)){noown.push(s.id); return}
  try{
   if(applyVal(kind,field,e,raw)===false){noown.push(s.id); return}
   done.push(s.id);
   /* بعد كلِّ كتابةٍ ناجحة — فالقاعدةُ تُطبَّق من كل مدخل */
   forceRules(kind,e).forEach(x=>
    forced.push(Object.assign({id:s.id},x)));
  }catch(err){ref.push({id:s.id,msg:err.message})}
 });
 if(done.length){
  /* النوع يُعلن نسختَه: تعديلُ نصٍّ بديلٍ في مئة بُعد لا يُبطِل
     اتحاد المضلّعات. والجدول هو المرجع لا شرطٌ هنا. */
  touchFn(bumpOf([{k:kind}]))();
 }
 return {field, kind, done:done.length, ids:done,
  refused:ref, noown:noown.length, skipped:gone.length, forced};
}
/* ═══ التثبيت المفرد ═══
   غلافٌ على applyField: كيانٌ واحد وحقلٌ واحد، فلا مسارَ ثانٍ
   بحدودٍ مكرَّرة. وبلا skipBlank — الفراغُ في المفردة قيمة. */
export function applyOne(s,field,raw){
 const r=applyField(s.k,[s],field,raw);
 if(r.done)return {ok:1, forced:r.forced,
  /* القسرُ يُقال: قيمةٌ تتبدّل بلا كلمةٍ أسوأُ من رفضٍ مُعلَن */
  msg:(r.forced&&r.forced.length)?sayForced(r.forced):""};
 if(r.refused.length)return {ok:0,msg:r.refused[0].msg};
 if(r.skipped)return {ok:0,msg:"الكيان لم يعد موجوداً"};
 return {ok:0,msg:`لا يملك ${NAME[s.k]||s.k} الحقل «${field}»`};
}
/* ═══ أوامر جماعية صريحة ═══ */
export function renumberCols(list,prefix){
 const P=String(prefix||"C").replace(/\d+$/,"").slice(0,6)||"C";
 const C=(list||[]).filter(pickable).map(s=>colById(s.id))
  .filter(Boolean);
 if(!C.length)throw new Error("لا أعمدة قابلة للترقيم");
 /* الترتيب من أعلى اليمين: y نازلاً ثم x نازلاً — قراءةً عربية */
 C.sort((a,b)=>(b.y-a.y)||(b.x-a.x));
 C.forEach((c,i)=>{c.tag=P+(i+1)});
 touch();
 return {n:C.length,first:C[0].tag,last:C[C.length-1].tag};
}
export function clearDimTxt(list){
 let n=0;
 (list||[]).filter(pickable).forEach(s=>{
  const d=dimById(s.id);
  if(d&&d.txt!=null){delete d.txt; n++}
 });
 if(n)touch();
 return n;
}
/* ═══ مسحُ الأسماء ═══
   الفراغُ في الجماعية يعني «لا تكتب»، فمسحُ أسماء عشرِ مناطقَ
   دفعةً واحدة لا سبيلَ إليه من الحقل. وزرٌّ صريحٌ خيرٌ من معنىً
   ثانٍ للفراغ يقع خلسة — كزرِّ «امسح النصّ البديل» للأبعاد. */
export function clearAreaNames(list){
 let n=0;
 (list||[]).filter(pickable).forEach(s=>{
  const a=areaById(s.id);
  if(a&&a.name){a.name=""; n++}
 });
 if(n)touch();
 return n;
}
export function nameAreasSeq(list,prefix){
 const P=String(prefix||"").trim().slice(0,24);
 const A=(list||[]).filter(pickable).map(s=>areaById(s.id))
  .filter(Boolean);
 if(!A.length)throw new Error("لا مناطق محدَّدة");
 A.sort((a,b)=>netArea(b)-netArea(a));   /* الأكبر أوّلاً */
 A.forEach((a,i)=>{a.name=(P?`${P} ${i+1}`:String(i+1))});
 touch();
 return {n:A.length,first:A[0].name};
}
/* ═══ التقرير ═══ */
const fmtV=(t,v)=>(t==="len")?`${m3(v)} م`:String(v);
export const sayForced=F=>(F||[]).map(x=>
 `${x.n} قُسِرت إلى ${fmtV(x.t,x.v)}`+(x.why?` — ${x.why}`:""))
 .join(" · ");
/* والقسرُ مجموعٌ بسببه في الجماعية: عشرون قسراً بسببٍ واحدٍ
   سطرٌ واحد. */
export function groupForced(F){
 const m=new Map();
 (F||[]).forEach(x=>{
  const k=x.n+"|"+x.why+"|"+fmtV(x.t,x.v);
  m.set(k,(m.get(k)||0)+1);
 });
 const out=[];
 m.forEach((n,k)=>{
  const [nm,why,v]=k.split("|");
  out.push(`${nm} قُسِرت إلى ${v} في ${n} عنصراً`
   +(why?` — ${why}`:""));
 });
 return out;
}
export function sayApply(r){
 const F=fldName(r.kind,r.field);
 const P=[`${F}: ${r.done} من ${NAME[r.kind]||r.kind}`];
 if(r.noown)P.push(`تُخطِّي ${r.noown} لا يملكها`);
 if(r.skipped)P.push(`${r.skipped} مخفيّ أو مقفل`);
 if(r.refused.length)P.push(`رُفض ${r.refused.length}`);
 if(r.forced&&r.forced.length)
  P.push(`قُسِرت ${r.forced.length} قيمة`);
 return P.join(" · ");
}
export const summary=G=>groupOrder(G)
 .map(k=>`${G[k].length} ${NAME[k]||k}`).join(" · ");
```

<a id="f-js-core-blocks-js"></a>

---

## `js/core/perf.js`

```javascript
/* ═══ القياس ═══
   عدّاداتٌ لا مؤقّتات: عددُ إعادات البناء وعددُ اختبارات التقاطع
   لا يتبدّلان بحاسبٍ آخر، والمللي ثانية يتبدّل بكل شيء. والدفعة
   الثامنة كلّها عن «ما يُبطَل»، فقياسُه هو التحقّق منها.

   ومُطفأٌ افتراضاً فكلفته صفر: bump يفحص علماً واحداً ويعود.
   ويستورد geom.js وحده — وهو ورقةٌ في الشجرة، فلا دورة. */
import {PERF as G, perfReset as gReset,
        WELD as GEOW} from "./geom.js";

export const P={on:0, frame:0, scene:0, bodies:0, loops:0, stamp:0,
 band:0, filter:0, prims:0};
export function perfClear(){
 P.frame=0; P.scene=0; P.bodies=0; P.loops=0; P.stamp=0;
 P.band=0; P.filter=0; P.prims=0;
 gReset();
 return P;
}
export function perfOn(v){
 P.on=v?1:0;
 if(P.on)perfClear();
 return P.on;
}
export const bump=(k,n)=>{
 if(P.on)P[k]=(P[k]||0)+((n==null)?1:n);
};

export function perfReport(){
 const L=[];
 L.push(`إطار ${P.frame} · مشهد ${P.scene} · أجسام ${P.bodies}`
  +` · حلقات ${P.loops} · بصمة ${P.stamp}`);
 L.push(`نطاقاتٌ مُعاد بناؤها ${P.band} · ${P.prims} أوّلية`
  +` · تصفية ${P.filter}`);
 L.push(`اتحاد ${G.union} · شظايا ${G.frags} ⇒ ${G.kept} مُبقاة`
  +` · حلقات ${G.rings}`);
 L.push(`اختبار تقاطع ${G.pairs.toLocaleString("en")}`
  +` · احتواء ${G.pip.toLocaleString("en")}`
  +` · خلايا ${G.cells}`);
 /* اللحم عملٌ عاديّ لا عطب: كلُّ زاويةٍ كسريّة تُنتِجه، فيُقال
    في القياس ولا يُقال في التصدير — والتقريرُ الذي يُطلَق دائماً
    لا يُقرأ أبداً. */
 if(G.weld||G.dup||G.nil)
  L.push(`عُقدٌ مُلحَمة ${G.weld} · قطعٌ مكرّرة ${G.dup}`
   +` · صفريّة ${G.nil} (حدُّ اللحم ${GEOW} مم)`);
 if(G.ms>1)L.push(`زمن الاتحاد ${Math.round(G.ms)} مس`);
 if(G.open){
  L.push(`⚠ ${G.open} قطعةً لم تُخَط — مدخلٌ معطوبٌ لا تفاوتُ `
   +`تدوير: اللحم يبلغ ${GEOW} مم`);
  if(G.openAt)L.push(`   آخرُ موضعٍ: `
   +`(${Math.round(G.openAt[0])}، ${Math.round(G.openAt[1])}) مم`);
 }
 return L.join("\n");
}
```

<a id="f-js-core-persist-js"></a>

---

## `js/core/inspect.js`

```javascript
/* ═══ الفاحص ═══
   يُنفَّذ بزرّ لا مع كل رسمة، ويجمع كل ما تفرّق من ملاحظات في قائمة
   واحدة قابلة للنقر. لا يصلح شيئاً ولا يحذف ولا يزحف — يخبرك
   وأنت تقرّر. هذا بديل validate() الذي كان يجري مع كل تحديث. */
import {S} from "./state.js";
import {m2,m3,sqm,pt2} from "./units.js";
import {pip,bboxOf,bboxHit,pArea,centroid} from "./geom.js";
import {looseEnds,wallLen,wallById,MINW,isArc} from "./walls.js";
import {badOpens,openState,opensOf,span,okName} from "./opens.js";
import {isStale,netArea,labelPt} from "./areas.js";
import {looseDims,isOverridden,dimValue,fmtLen,dimMid,
        chainCompare,chainPt,chainBounds,chainSum} from "./dims.js";
import {colOnWall,colsOverlap,colPoly,colBBox,
        colLabel} from "./cols.js";
import {fixOnWall,fixOverlap,fixBBox,fixCenter,
        fixName} from "./fixt.js";
import * as SI from "./sindex.js";
import {stCheck,stGeom} from "./stairs.js";
import {fitsSheet} from "./sheet.js";
import {hiddenLayers,lockedLayers,LNAME,hiddenCount,
        layCounts} from "./layers.js";
import {hasRef,refStats} from "./ref.js";
import {skipSummary} from "../io/dxfin.js";
import {CODE,codeCheck} from "./code.js";

export const SEV={er:"خطأ",wr:"تنبيه",in:"ملاحظة"};

/* كل نتيجة: {sev, code, msg, k, id, p} — p هدف القفز */
export function inspect(bbox,opt){
 const O=Object.assign({endTol:2,dimTol:30,chainTol:60},opt||{});
 const F=[];
 const add=(sev,code,msg,k,id,p)=>F.push({sev,code,msg,k,id,
  p:p?[Math.round(p[0]),Math.round(p[1])]:null});

 /* ١ — أطراف الجدران غير المتّصلة */
 looseEnds(O.endTol).forEach(e=>add("wr","end",
  `${e.id}: طرف ${e.end==="a"?"البداية":"النهاية"} لا يلامس شيئاً `
  +`${pt2(e.p)}`,
  "wall",e.id,e.p));

 /* ٢ — جدران أقصر من الحدّ الأدنى أو صفرية */
 S.walls.forEach(w=>{
  const L=wallLen(w);
  if(L<1)add("er","w0",`${w.id}: جدار صفري الطول`,
   "wall",w.id,w.a);
  else if(L<MINW)add("wr","wshort",
   `${w.id}: طوله ${m3(L)} م — أقصر من الحدّ الأدنى `
   +`${m3(MINW)} م`,"wall",w.id,w.a);
 });
 /* ٢ب — جدران قوسية: لا تُسقَط بعد في الواجهات/المقاطع */
 S.walls.forEach(w=>{
  if(isArc(w))add("in","warcnoelev",
   `${w.id}: جدار قوسي — يُستبعَد من الواجهات والمقاطع حالياً `
   +`(غير مدعوم بعد)`,"wall",w.id,w.a);
 });
 /* ٣ — جدران متطابقة تماماً */
 const seen=new Map();
 S.walls.forEach(w=>{
  const a=`${w.a[0]},${w.a[1]}`, b=`${w.b[0]},${w.b[1]}`;
  const k=(a<b)?`${a}|${b}`:`${b}|${a}`;
  if(seen.has(k))add("wr","wdup",
   `${w.id}: مسارُه مطابق لـ ${seen.get(k)} تماماً — جدار مكرَّر؟`,
   "wall",w.id,w.a);
  else seen.set(k,w.id);
 });
 /* ٤ — الفتحات المعطوبة */
 badOpens().forEach(o=>{
  const st=openState(o);
  const w=wallById(o.wall);
  const p=w?[(w.a[0]+w.b[0])/2,(w.a[1]+w.b[1])/2]:null;
  const T={over:"تخرج عن مدى جدارها",
   clash:"تتراكب مع فتحة أخرى على الجدار نفسه",
   orphan:"جدارها غير موجود"};
  add(st==="orphan"?"er":"wr","open",
   `${o.id} ${okName(o.kind)}: ${T[st]||st} — لم تُزحَف ولم `
   +`تُقلَّم`,"open",o.id,p);
 });
 /* ٥ — المناطق: القديمة والمتراكبة وبلا اسم */
 S.areas.forEach(a=>{
  if(isStale(a))add("wr","astale",
   `${a.id} ${a.name||""}: قديمة — تغيّر جدار يجاورها. الحلقة `
   +`المخزَّنة ${sqm(netArea(a))} م² لم تُمَسّ`,
   "area",a.id,labelPt(a));
  if(!a.name)add("in","aname",
   `${a.id}: بلا اسم · ${sqm(netArea(a))} م²`,
   "area",a.id,labelPt(a));
 });
 SI.forPairs("area",(A,B)=>{
  const ba=bboxOf(A.ring), bb=bboxOf(B.ring);
  if(!ba||!bb||!bboxHit(ba,bb,-1))return;
  const c=centroid(B.ring);
  if(pip(A.ring,c[0],c[1]))add("wr","aover",
   `${B.id} ${B.name||""} داخل ${A.id} ${A.name||""} — `
   +`منطقتان متراكبتان`,"area",B.id,c);
 });
 /* ٦ — الأبعاد المعلَّقة والمكتوبة يدوياً */
 looseDims(O.dimTol).forEach(d=>add("wr","dloose",
  `${d.id}: طرفٌ لا يصادف هندسةً — البُعد ${fmtLen(dimValue(d))} م `
  +`لم يُزحَف ولم يُحذَف`,"dim",d.id,dimMid(d)));
 S.dims.filter(isOverridden).forEach(d=>add("wr","dtxt",
  `${d.id}: نصّ بديل «${d.txt}» يُعرَض بدل المقاس الحقيقي `
  +`${fmtLen(dimValue(d))} م`,"dim",d.id,dimMid(d)));

 /* ٧ — السلاسل المخالفة للهندسة (تقرير) */
 S.chains.forEach(c=>{
  const r=chainCompare(c,O.chainTol);
  if(!r.off)return;
  const B=chainBounds(c);
  add("in","coff",
   `${c.id}: ${r.off} من ${r.rows.length} حدّاً خارج التفاوت — `
   +`المجموع المكتوب ${fmtLen(chainSum(c))} م`,
   "chain",c.id,chainPt(c,B[Math.floor(B.length/2)]));
 });
 /* ٨ — الأعمدة */
 S.cols.forEach(c=>{
  const b=colBBox(c);
  const on=colOnWall(c,2,b?SI.entsIn(SI.expand(b,4),"wall"):null);
  if(!on)add("in","kfree",
   `${c.id}${c.tag?" "+c.tag:""}: عمود منفرد لا يلامس جداراً — `
   +`${colLabel(c)}`,"col",c.id,[c.x,c.y]);
  /* المساحة لا تخصم العمود المنفرد: تُبلَّغ ولا تُخصَم */
  if(!on)SI.entsAt(c.x,c.y,0,"area").forEach(a=>{
   if(!pip(a.ring,c.x,c.y))return;
   add("in","kinarea",
    `${c.id}${c.tag?" "+c.tag:""} داخل ${a.id} ${a.name||""} — `
    +`المساحة المعروضة لا تخصمه `
    +`(${sqm(Math.abs(pArea(colPoly(c))))} م²)`,
    "col",c.id,[c.x,c.y]);
  });
 });
 SI.forPairs("col",(a,b)=>{
  if(!colsOverlap(a,b))return;
  add("wr","kover",
   `${a.id} و ${b.id} متراكبان — مقصود أم سهو؟`,
   "col",b.id,[b.x,b.y]);
 });

 /* ٩ — الأدوات الصحية */
 S.fixt.forEach(f=>{
  const b=fixBBox(f);
  const W=b?SI.entsIn(SI.expand(b,200),"wall"):null;
  if(!fixOnWall(f,150,W))add("in","ffree",
   `${f.id} ${fixName(f)}: ظهرها لا يلاصق جداراً`,
   "fix",f.id,fixCenter(f));
 });
 SI.forPairs("fix",(a,b)=>{
  if(!fixOverlap(a,b))return;
  add("wr","fover",
   `${a.id} ${fixName(a)} تتراكب مع ${b.id} ${fixName(b)}`,
   "fix",b.id,fixCenter(b));
 });

 /* ١٠ — الدرج: يُقاس ويُبلَّغ ولا يُصحَّح */
 S.stairs.forEach(t=>{
  const c=stCheck(t);
  const g=stGeom(t);
  const p=g?g.P(g.L/2,0):t.a;
  c.msgs.forEach(m=>add("wr","stair",`${t.id}: ${m}`,
   "stair",t.id,p));
  if(c.ok)add("in","stairok",
   `${t.id}: ${c.n} قائمة · ق ${m3(c.rise)} · ن ${m3(c.tread)} م `
   +`· 2ق+ن ${m3(c.rule)} م — داخل المدى المريح`,
   "stair",t.id,p);
 });
 /* ١١ — الطبقات المخفيّة والمقفلة
    ليس عيباً، لكنه سببٌ لغياب ما تتوقّع رؤيته. */
 const HD=hiddenLayers(), C=layCounts();
 if(HD.length){
  add("wr","lhid",
   `${HD.length} طبقة مخفيّة (${HD.map(LNAME).join(" · ")}) — `
   +`${hiddenCount()} كياناً لن يُرسَم ولن يُصدَّر`,null,null,null);
  HD.forEach(L=>{
   if(!(C[L]>0))return;
   add("in","lhidn",`${LNAME(L)}: ${C[L]} كياناً مخفيّاً`,
    null,null,null);
  });
 }
 const LK=lockedLayers();
 if(LK.length)add("in","llock",
  `${LK.length} طبقة مقفلة (${LK.map(LNAME).join(" · ")}) — `
  +`تُرى ولا تُحدَّد`,null,null,null);

 /* ١٢ — المرجع */
 if(hasRef()){
  const r=refStats();
  add("in","refn",
   `المرجع «${r.name||"بلا اسم"}»: ${r.n} كياناً على `
   +`${r.layers} طبقة (${r.shownLayers} ظاهرة) · الترميز `
   +`${r.enc} · جامدٌ لا يدخل الاتحاد ولا المساحات`,
   null,null,null);
  if(r.guessed&&Math.abs(r.k-1)<1e-9)
   add("wr","refunit",
    `وحدة المرجع مجهولة في الملفّ وفُرضت مليمتراً، ولم تُعايره `
    +`بعد — استعمل «معايرة المرجع» بمسافةٍ تعرفها قبل أن تقيس `
    +`عليه`,null,null,null);
  if(r.trunc)
   add("wr","reftrunc",
    `استُثني ${r.trunc} كياناً لتجاوز الحدّ — المرجع منقوص`,
    null,null,null);
  const sk=skipSummary(r.skip);
  if(sk)add("in","refskip",
   `تُخطّي من المرجع: ${sk}`,null,null,null);
  const ap=r.approx||{}, A=[];
  if(ap.spline)A.push(`${ap.spline} منحنى SPLINE (متقطّع)`);
  if(ap.ellipse)A.push(`${ap.ellipse} قطع ناقص`);
  if(ap.arc)A.push(`${ap.arc} قوساً بمقياس غير متساوٍ`);
  if(A.length)add("in","refapx",
   `تقريباتٌ في المرجع: ${A.join(" · ")}`,null,null,null);
  if(r.bbox&&bbox){
   const gap=Math.max(r.bbox.x0-bbox.x1, bbox.x0-r.bbox.x1,
                      r.bbox.y0-bbox.y1, bbox.y0-r.bbox.y1);
   if(gap>500000)add("wr","reffar",
    `المرجع يبعد عن رسمك ${m2(gap)} م — حاذِه أو انقله`,
    null,null,null);
  }
 }
 /* ١٣ — الورقة */
 if(+S.sheet.on){
  const f=fitsSheet(bbox);
  if(!f.ok)add("wr","sheet",
   `الرسم يتجاوز الإطار الداخلي بـ ${m2(f.over)} م — كبّر الورقة `
   +`أو صغّر المقياس أو أزِح الورقة`,null,null,null);
 }
 /* ١٤ — لا شيء مرسوم */
 if(!S.walls.length&&!S.cols.length)
  add("in","empty","لا جدران ولا أعمدة في المشروع",
   null,null,null);

 /* ١٥ — الاشتراطات: طبقةٌ ثانية تفحص التصميم لا الهندسة.
    نتائجها بالشكل نفسه فتقفز إلى مواضعها كبقيّة الملاحظات. */
 if(+CODE.on)codeCheck().forEach(f=>F.push(f));

 const ORD={er:0,wr:1,in:2};
 F.sort((a,b)=>ORD[a.sev]-ORD[b.sev]||a.code.localeCompare(b.code));
 return {list:F,
  er:F.filter(x=>x.sev==="er").length,
  wr:F.filter(x=>x.sev==="wr").length,
  in:F.filter(x=>x.sev==="in").length};
}
```

<a id="f-js-core-journal-js"></a>

---

## `js/core/code.js`

```javascript
/* ═══ فاحص الاشتراطات ═══
   الفاحص الحالي يفحص الهندسة: أطرافٌ لا تلتقي، فتحةٌ تخرج عن
   جدارها، منطقةٌ قديمة. وهذه طبقةٌ ثانية تفحص التصميم نفسه:
   أعرضُ البابُ كافٍ؟ أللغرفة ضوءٌ وتهوية؟ أالممرّ يمرّ منه اثنان؟

   والعقد نفسه يسري: يخبر ولا يصلح، وكل نتيجةٍ تقفز إلى موضعها.

   القيَم أدناه إرشاديةٌ لا نصّ نظام: تُعدَّل لتطابق الكود المعتمد
   في بلدك ومشروعك. وهي بالمليمتر كبقيّة الحالة. */
import {S} from "./state.js";
import {m2,m3,sqm} from "./units.js";
import {pip,bboxOf,centroid} from "./geom.js";
import {wallById} from "./walls.js";
import {openPt,okName} from "./opens.js";
import {netArea,labelPt} from "./areas.js";

const K="mistar.code";
export const CODE={
 on:1,
 doorW:800,      /* باب غرفة */
 doorWet:700,    /* باب دورة مياه */
 doorExt:900,    /* باب على جدار خارجي */
 doorH:2000,
 sillLow:800,    /* جلسة أدنى منها تحتاج حماية */
 light:0.10,     /* مساحة الزجاج ÷ مساحة الأرضية */
 vent:0.05,      /* القابل للفتح ÷ مساحة الأرضية */
 roomMin:6e6,    /* ٦ م² بالمليمتر المربّع */
 wetMin:1.5e6,
 corrW:1000,
 ceilH:2600};

const KEYS=Object.keys(CODE);
export function loadCode(){
 if(typeof localStorage==="undefined")return CODE;
 try{
  const d=JSON.parse(localStorage.getItem(K)||"null")||{};
  KEYS.forEach(k=>{if(typeof d[k]==="number")CODE[k]=d[k]});
 }catch(e){}
 return CODE;
}
export function saveCode(){
 if(typeof localStorage==="undefined")return;
 try{localStorage.setItem(K,JSON.stringify(CODE))}catch(e){}
}

/* ═══ الربط بين الفتحة والمنطقة ═══
   المنطقة حلقةٌ على أوجه الجدران، والفتحة نقطةٌ على مسار جدارها —
   فالقرب من الحلقة هو الانتماء. تفاوتٌ بسماكة الجدار لأن المسار
   قد يكون محورياً والحلقة على الوجه. */
function segD(p,a,b){
 const dx=b[0]-a[0], dy=b[1]-a[1];
 const L2=dx*dx+dy*dy;
 if(L2<1)return Math.hypot(p[0]-a[0],p[1]-a[1]);
 let t=((p[0]-a[0])*dx+(p[1]-a[1])*dy)/L2;
 t=t<0?0:(t>1?1:t);
 return Math.hypot(p[0]-a[0]-dx*t, p[1]-a[1]-dy*t);
}
function onRing(ring,p,tol){
 for(let i=0;i<ring.length;i++){
  if(segD(p,ring[i],ring[(i+1)%ring.length])<=tol)return true;
 }
 return false;
}
/* تصنيفٌ بالاسم: الفراغات تُسمّى بالعربية، والاسم أصدق دليلٍ
   متاح على وظيفة الفراغ. ما لا يُعرَف لا يُحاسَب بقاعدةٍ خاصّة. */
const CLS=[
 [/(دوره|دورة|حمام|حمّام|مرحاض|بانيو|wc)/i,"wet"],
 [/(ممر|ممشى|بهو|مدخل|درج)/i,"corr"],
 [/(مطبخ)/i,"kitchen"],
 [/(نوم|مجلس|صاله|صالة|معيشه|معيشة|مكتب|غرف)/i,"room"]];
const classOf=a=>{
 const n=String(a.name||"");
 for(const [rx,c] of CLS)if(rx.test(n))return c;
 return "";
};
const GLASS=/^(window|fixed)$/;
const DOORS=/^(door|double|sliding)$/;

export function codeCheck(){
 const F=[];
 const add=(sev,code,msg,k,id,p)=>F.push({sev,code,msg,k,id,
  p:p?[Math.round(p[0]),Math.round(p[1])]:null});
 if(!+CODE.on)return F;

 if(S.meta.wallH&&S.meta.wallH<CODE.ceilH)
  add("wr","c-ceil",
   `ارتفاع الدور ${m2(S.meta.wallH)} م دون الحدّ الإرشادي `
   +`${m2(CODE.ceilH)} م`,null,null,null);

 /* الربط يُبنى مرّةً: الفتحة قد تخصّ منطقتين (باب بينهما) */
 const inArea=new Map();          /* معرّف المنطقة ← فتحاتها */
 const ofOpen=new Map();          /* معرّف الفتحة ← مناطقها */
 S.areas.forEach(a=>inArea.set(a.id,[]));
 S.opens.forEach(o=>{
  const w=wallById(o.wall);
  if(!w)return;
  const q=openPt(w,o.s);
  const tol=Math.max(w.t,200);
  S.areas.forEach(a=>{
   if(!a.ring||a.ring.length<3)return;
   if(!onRing(a.ring,q,tol))return;
   inArea.get(a.id).push({o,w});
   const L=ofOpen.get(o.id)||[];
   L.push(a); ofOpen.set(o.id,L);
  });
 });

 /* ═══ الأبواب ═══ */
 S.opens.filter(o=>DOORS.test(o.kind)).forEach(o=>{
  const w=wallById(o.wall);
  if(!w)return;
  const p=openPt(w,o.s);
  const AS=ofOpen.get(o.id)||[];
  const wet=AS.some(a=>classOf(a)==="wet");
  const ext=(w.type==="ext");
  const min=ext?CODE.doorExt:(wet?CODE.doorWet:CODE.doorW);
  const why=ext?"على جدار خارجي":(wet?"لدورة مياه":"لغرفة");
  if(o.w<min)add(ext?"wr":"in","c-dw",
   `${o.id} ${okName(o.kind)}: عرضه ${m2(o.w)} م — الحدّ `
   +`الإرشادي ${m2(min)} م ${why}`,"open",o.id,p);
  if(o.h<CODE.doorH)add("in","c-dh",
   `${o.id}: ارتفاعه ${m2(o.h)} م دون ${m2(CODE.doorH)} م`,
   "open",o.id,p);
 });

 /* ═══ الشبابيك: الجلسة المنخفضة ═══ */
 S.opens.filter(o=>GLASS.test(o.kind)).forEach(o=>{
  if(o.sill>=CODE.sillLow)return;
  const w=wallById(o.wall);
  add("in","c-sill",
   `${o.id} ${okName(o.kind)}: جلسته ${m2(o.sill)} م دون `
   +`${m2(CODE.sillLow)} م — يحتاج حمايةً أو زجاجاً أمان`,
   "open",o.id,w?openPt(w,o.s):null);
 });

 /* ═══ المناطق: المساحة والضوء والتهوية والعرض ═══ */
 S.areas.forEach(a=>{
  if(!a.ring||a.ring.length<3)return;
  const cls=classOf(a);
  const A=netArea(a);
  const at=labelPt(a)||centroid(a.ring);
  const nm=a.name||a.id;

  if(cls==="room"&&A<CODE.roomMin)add("wr","c-amin",
   `${a.id} ${nm}: ${sqm(A)} م² دون الحدّ الإرشادي `
   +`${sqm(CODE.roomMin)} م² للغرفة`,"area",a.id,at);
  if(cls==="wet"&&A<CODE.wetMin)add("in","c-amin",
   `${a.id} ${nm}: ${sqm(A)} م² دون ${sqm(CODE.wetMin)} م² `
   +`لدورة المياه`,"area",a.id,at);

  /* الممرّ: أدنى ضلعٍ لصندوقه المحيط تقريبٌ معلَن، لا قياسُ عرضٍ
     حقيقي لمضلّعٍ منحرف. يُنبّه ولا يُجزَم. */
  if(cls==="corr"){
   const b=bboxOf(a.ring);
   const wdt=b?Math.min(b.x1-b.x0,b.y1-b.y0):0;
   if(wdt&&wdt<CODE.corrW)add("wr","c-corr",
    `${a.id} ${nm}: أضيق بُعدٍ لصندوقه ${m2(wdt)} م دون `
    +`${m2(CODE.corrW)} م — تقريبٌ من الصندوق المحيط، تحقّق `
    +`بالقياس`,"area",a.id,at);
  }
  if(cls!=="room"&&cls!=="kitchen")return;

  /* الضوء من الزجاج على الجدران الخارجية وحدها */
  const L=inArea.get(a.id)||[];
  const gl=L.filter(x=>GLASS.test(x.o.kind)&&x.w.type==="ext")
   .reduce((s,x)=>s+x.o.w*x.o.h,0);
  const vt=L.filter(x=>x.o.kind==="window"&&x.w.type==="ext")
   .reduce((s,x)=>s+x.o.w*x.o.h,0);
  if(!A)return;
  if(gl/A<CODE.light)add(gl?"wr":"er","c-light",
   `${a.id} ${nm}: زجاج ${sqm(gl)} م² على أرضية ${sqm(A)} م² `
   +`= ${(gl/A*100).toFixed(1)}% دون `
   +`${(CODE.light*100).toFixed(0)}% للإضاءة`
   +(gl?"":" — لا شباك على جدارٍ خارجي"),"area",a.id,at);
  else if(vt/A<CODE.vent)add("wr","c-vent",
   `${a.id} ${nm}: القابل للفتح ${sqm(vt)} م² `
   +`= ${(vt/A*100).toFixed(1)}% دون `
   +`${(CODE.vent*100).toFixed(0)}% للتهوية — الثابت لا يُهوّي`,
   "area",a.id,at);
 });
 return F;
}
```

<a id="f-js-core-cols-js"></a>

---

## `js/core/templates.js`

```javascript
/* ═══ قوالب مشاريع جاهزة ═══ */
const T = new Map();
export function defineTemplate(t) { if (!t || !t.name) throw new Error("template: name"); T.set(t.name, t); return t; }
export const templateList = () => [...T.values()].map(t => ({ name: t.name, title: t.title || t.name }));
export const getTemplate = name => T.get(name) || null;
export function build(name) {
  const t = T.get(name); if (!t) return null;
  return typeof t.build === "function" ? t.build() : JSON.parse(JSON.stringify(t.seed || {}));
}
export function apply(name, hooks) {
  const seed = build(name); if (!seed || !hooks) return null;
  (seed.walls || []).forEach(w => hooks.addWall?.(w));
  (seed.blocks || []).forEach(b => hooks.addBlock?.(b));
  if (seed.meta && hooks.setMeta) hooks.setMeta(seed.meta);
  return seed;
}
const room = (w, h, th = 150) => [
  { a: [0, 0], b: [w, 0], th }, { a: [w, 0], b: [w, h], th },
  { a: [w, h], b: [0, h], th }, { a: [0, h], b: [0, 0], th }
];
export function installDefaults() {
  defineTemplate({ name: "room", title: "غرفة مستطيلة", build: () => ({
    walls: room(4000, 3000), blocks: [{ block: "door", x: 1600, y: 0, rot: 0 }], meta: { scale: 50 }
  })});
  defineTemplate({ name: "studio", title: "استوديو", build: () => ({
    walls: room(6000, 4000), blocks: [
      { block: "door", x: 2500, y: 0, rot: 0 }, { block: "window", x: 4300, y: 4000, rot: 0 }
    ], meta: { scale: 75 }
  })});
  defineTemplate({ name: "office", title: "مكتب", build: () => ({
    walls: [...room(8000, 5000), { a: [5000, 0], b: [5000, 5000], th: 150 }],
    blocks: [
      { block: "door", x: 2000, y: 0, rot: 0 }, { block: "door", x: 5600, y: 2500, rot: Math.PI / 2 },
      { block: "window", x: 6300, y: 5000, rot: 0 }
    ], meta: { scale: 100 }
  })});
}
```

<a id="f-js-core-trace-js"></a>

---

