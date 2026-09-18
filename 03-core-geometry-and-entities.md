# CivilDraft — Core: Geometry & Entities

> الكيانات الهندسية الأساسية: الجدران، الأعمدة، الفتحات، السلالم، التركيبات، الإحداثيات، الوحدات.

**عدد الملفات:** 14

---

## `js/core/geom.js`

```javascript
/* ═══ الهندسة ═══
   دوالٌّ خالصة: لا تعرف الحالة ولا الطبقات ولا الوحدات، ولا تستورد
   شيئاً بقصد — ورقةُ الشجرة، وكلُّ شيءٍ يستوردها.
   والفهرسة هنا داخلية بمنطق core/sindex نفسه وبلا استيراده:
   sindex يقرأ S.walls، وهذه لا تعرف S. */

export const EPS=1e-9;
const R=v=>Math.round(v);
const now=()=>(typeof performance!=="undefined")
 ? performance.now() : Date.now();

/* ═══ العدّادات ═══
   ما يُقاس هو ما يُعَدّ: اختباراتُ التقاطع والاحتواء لا تتبدّل
   بحاسبٍ آخر، والمللي ثانية يتبدّل بكل شيء.
   وopen يتراكم فيُقرأ فرقُه حول النداء — فمن يُرشِّح مخرَج
   polyBool لا يُفقِد التقرير.
   وopenAt آخرُ موضعٍ لم يُخَط لا أوّلُه: المستدعي يقرأ الفرقَ في
   open ثم يقرأ الموضعَ يقيناً من ندائه هو — والأوّلُ يبقى من نداءٍ
   سابقٍ فيدلّ على غير مكانه. */
export const PERF={pairs:0,pip:0,frags:0,kept:0,union:0,
 stitch:0,open:0,openAt:null,weld:0,dup:0,nil:0,
 rings:0,cells:0,ms:0};
export function perfReset(){
 PERF.pairs=0; PERF.pip=0; PERF.frags=0; PERF.kept=0;
 PERF.union=0; PERF.stitch=0; PERF.open=0; PERF.openAt=null;
 PERF.weld=0; PERF.dup=0; PERF.nil=0;
 PERF.rings=0; PERF.cells=0; PERF.ms=0;
 return PERF;
}
export const polyStats=()=>Object.assign({},PERF);

/* ═══ أساسيات ═══ */
export const dist=(a,b)=>Math.hypot(b[0]-a[0],b[1]-a[1]);
export const dist2=(a,b)=>{
 const x=b[0]-a[0], y=b[1]-a[1];
 return x*x+y*y;
};
export const mid=(a,b)=>[(a[0]+b[0])/2,(a[1]+b[1])/2];
export const same=(a,b,t)=>{
 const T=(t==null)?1:t;
 return dist2(a,b)<=T*T;
};
export function rotPt(p,cx,cy,degv){
 const a=(degv||0)*Math.PI/180, c=Math.cos(a), s=Math.sin(a);
 const x=p[0]-cx, y=p[1]-cy;
 return [cx+x*c-y*s, cy+x*s+y*c];
}
/* ═══ الصناديق ═══ */
export function bboxOf(pts){
 if(!pts||!pts.length)return null;
 let x0=1/0,y0=1/0,x1=-1/0,y1=-1/0;
 for(let i=0;i<pts.length;i++){
  const p=pts[i];
  if(!p)continue;
  const x=+p[0], y=+p[1];
  if(!isFinite(x)||!isFinite(y))continue;
  if(x<x0)x0=x;
  if(x>x1)x1=x;
  if(y<y0)y0=y;
  if(y>y1)y1=y;
 }
 return (x0>x1)?null:{x0,y0,x1,y1};
}
export const bboxHit=(a,b,pad)=>{
 const p=pad||0;
 return !!a&&!!b&&(a.x0-p)<=b.x1&&(b.x0-p)<=a.x1
  &&(a.y0-p)<=b.y1&&(b.y0-p)<=a.y1;
};
export const bboxPad=(b,p)=>b
 ? {x0:b.x0-p,y0:b.y0-p,x1:b.x1+p,y1:b.y1+p} : null;
export const bboxIn=(b,x,y,p)=>{
 const q=p||0;
 return !!b&&x>=b.x0-q&&x<=b.x1+q&&y>=b.y0-q&&y<=b.y1+q;
};
export const bboxUnion=(a,b)=>{
 if(!a)return b?{x0:b.x0,y0:b.y0,x1:b.x1,y1:b.y1}:null;
 if(!b)return {x0:a.x0,y0:a.y0,x1:a.x1,y1:a.y1};
 return {x0:Math.min(a.x0,b.x0),y0:Math.min(a.y0,b.y0),
  x1:Math.max(a.x1,b.x1),y1:Math.max(a.y1,b.y1)};
};
function bboxAll(boxes){
 let r=null;
 for(let i=0;i<boxes.length;i++)if(boxes[i])r=bboxUnion(r,boxes[i]);
 return r;
}
/* ═══ قياسات الحلقة ═══ */
export function pArea(r){
 const n=(r||[]).length;
 if(n<3)return 0;
 let s=0;
 for(let i=0;i<n;i++){
  const a=r[i], b=r[(i+1)%n];
  s+=a[0]*b[1]-b[0]*a[1];
 }
 return s/2;
}
export const ccw=r=>(pArea(r)<0)
 ? (r||[]).slice().reverse() : (r||[]).slice();
export function perim(r){
 const n=(r||[]).length;
 if(n<2)return 0;
 let s=0;
 for(let i=0;i<n;i++)s+=dist(r[i],r[(i+1)%n]);
 return s;
}
export function centroid(r){
 const n=(r||[]).length;
 if(!n)return [0,0];
 const avg=()=>{
  let x=0,y=0;
  for(let i=0;i<n;i++){x+=r[i][0]; y+=r[i][1]}
  return [x/n,y/n];
 };
 if(n<3)return avg();
 let a=0,cx=0,cy=0;
 for(let i=0;i<n;i++){
  const p=r[i], q=r[(i+1)%n];
  const f=p[0]*q[1]-q[0]*p[1];
  a+=f; cx+=(p[0]+q[0])*f; cy+=(p[1]+q[1])*f;
 }
 if(Math.abs(a)<1e-9)return avg();
 return [cx/(3*a), cy/(3*a)];
}
/* تنظيف الحلقة: المكرّر المتلاصق يُطرَح، والرأس المستقيم كذلك —
   فضلعٌ واحد بدل ثلاثة، وأخفُّ على الرسم والتصدير والاتحاد. */
export function cleanRing(r,tol){
 const T=Math.max(0,(tol==null)?1:tol);
 const src=(r||[]).filter(p=>Array.isArray(p)
  &&isFinite(p[0])&&isFinite(p[1]));
 const a=[];
 for(let i=0;i<src.length;i++){
  const q=[R(src[i][0]),R(src[i][1])];
  if(a.length&&dist(a[a.length-1],q)<=T)continue;
  a.push(q);
 }
 while(a.length>1&&dist(a[0],a[a.length-1])<=T)a.pop();
 if(a.length<3)return a;
 const out=[];
 for(let i=0;i<a.length;i++){
  const p=a[(i-1+a.length)%a.length], c=a[i], n=a[(i+1)%a.length];
  const cr=(c[0]-p[0])*(n[1]-p[1])-(c[1]-p[1])*(n[0]-p[0]);
  const base=dist(p,n);
  if(base>T&&Math.abs(cr)/base<=T)continue;
  out.push(c);
 }
 return (out.length>2)?out:a;
}
/* ═══ اختبارات النقطة ═══ */
export function pip(poly,x,y){
 const n=(poly||[]).length;
 if(n<3)return false;
 PERF.pip++;
 let inside=false;
 for(let i=0,j=n-1;i<n;j=i++){
  const a=poly[i], b=poly[j];
  if((a[1]>y)!==(b[1]>y)){
   const t=(y-a[1])/(b[1]-a[1]);
   if(x<a[0]+t*(b[0]-a[0]))inside=!inside;
  }
 }
 return inside;
}
export function nearOnSeg(a,b,x,y){
 const dx=b[0]-a[0], dy=b[1]-a[1];
 const L2=dx*dx+dy*dy;
 let t=(L2<1e-12)?0:(((x-a[0])*dx+(y-a[1])*dy)/L2);
 t=(t<0)?0:((t>1)?1:t);
 const px=a[0]+dx*t, py=a[1]+dy*t;
 return {t, p:[px,py], d:Math.hypot(x-px,y-py)};
}
export const distSeg=(a,b,x,y)=>nearOnSeg(a,b,x,y).d;
export const onSeg=(a,b,x,y,tol)=>
 nearOnSeg(a,b,x,y).d<=((tol==null)?1:tol);
/* أقرب مسافة من نقطة إلى أي قطعة في قائمة — لعلامات الأطراف */
export const nearAny=(segs,p,skip)=>{
 let m=1/0;
 (segs||[]).forEach((s,i)=>{
  if(i===skip)return;
  const d=distSeg(s[0],s[1],p[0],p[1]);
  if(d<m)m=d;
 });
 return m;
};
export function distPoly(p,x,y){
 let m=1/0;
 for(let i=0,n=p.length;i<n;i++){
  const d=distSeg(p[i],p[(i+1)%n],x,y);
  if(d<m)m=d;
 }
 return m;
}

/* ═══ بنّاؤو المضلّعات ═══ */
export function bandPoly(x1,y1,x2,y2,t){
 const dx=x2-x1, dy=y2-y1, L=Math.hypot(dx,dy);
 if(L<1e-6)return null;
 const h=(t||0)/2, nx=-dy/L*h, ny=dx/L*h;
 return [[R(x1+nx),R(y1+ny)],[R(x2+nx),R(y2+ny)],
         [R(x2-nx),R(y2-ny)],[R(x1-nx),R(y1-ny)]];
}
export function rectPoly(cx,cy,w,h,degv){
 const a=(w||0)/2, b=(h||0)/2;
 const P=[[-a,-b],[a,-b],[a,b],[-a,b]];
 const g=(degv||0)*Math.PI/180, c=Math.cos(g), s=Math.sin(g);
 return P.map(p=>[R(cx+p[0]*c-p[1]*s), R(cy+p[0]*s+p[1]*c)]);
}
export function circPoly(cx,cy,r,n){
 const N=Math.max(8,Math.min(96,n||24)), out=[];
 for(let i=0;i<N;i++){
  const a=Math.PI*2*i/N;
  out.push([R(cx+r*Math.cos(a)), R(cy+r*Math.sin(a))]);
 }
 return out;
}

/* ═══ خطوط ومستطيلات مساعدة (يستعملها trace/osnap/modify/ents) ═══
   خطّان كاملان لا قطعتان محدودتان: يعيد نقطة تقاطع المحورين ولو
   وقعت خارج طرفَي القطعتين — لأدوات المطابقة والامتداد. */
export function lineX(a,b,c,d){
 const ex=b[0]-a[0], ey=b[1]-a[1];
 const fx=d[0]-c[0], fy=d[1]-c[1];
 const den=ex*fy-ey*fx;
 if(Math.abs(den)<1e-9)return null;
 const rx=c[0]-a[0], ry=c[1]-a[1];
 const t=(rx*fy-ry*fx)/den;
 return [a[0]+ex*t, a[1]+ey*t];
}
export function segSeg(p,q,a,b){
 const d1=[q[0]-p[0],q[1]-p[1]], d2=[b[0]-a[0],b[1]-a[1]];
 const den=d1[0]*d2[1]-d1[1]*d2[0];
 if(Math.abs(den)<1e-9)return false;
 const t=((a[0]-p[0])*d2[1]-(a[1]-p[1])*d2[0])/den;
 const u=((a[0]-p[0])*d1[1]-(a[1]-p[1])*d1[0])/den;
 return t>=-1e-9&&t<=1+1e-9&&u>=-1e-9&&u<=1+1e-9;
}
export const ptInRect=(p,r)=>
 p[0]>=r.x0&&p[0]<=r.x1&&p[1]>=r.y0&&p[1]<=r.y1;
export function segRect(a,b,r){
 if(ptInRect(a,r)||ptInRect(b,r))return true;
 const E=[[[r.x0,r.y0],[r.x1,r.y0]],[[r.x1,r.y0],[r.x1,r.y1]],
          [[r.x1,r.y1],[r.x0,r.y1]],[[r.x0,r.y1],[r.x0,r.y0]]];
 return E.some(e=>segSeg(a,b,e[0],e[1]));
}
/* window=1 يشترط الاحتواء الكامل · 0 يكفيه التلامس */
export function shapeInRect(sh,r,window){
 if(!sh)return false;
 if(sh.t==="pt")return ptInRect(sh.p,r);
 if(sh.t==="seg")return window
  ? (ptInRect(sh.a,r)&&ptInRect(sh.b,r))
  : segRect(sh.a,sh.b,r);
 if(!sh.pts||!sh.pts.length)return false;
 const all=sh.pts.every(p=>ptInRect(p,r));
 if(window)return all;
 if(all)return true;
 for(let i=0;i<sh.pts.length;i++)
  if(segRect(sh.pts[i],sh.pts[(i+1)%sh.pts.length],r))return true;
 return ptInRect(sh.pts[0],r);
}
/* هيكل محدَّب — لرقع الأركان وقت العرض */
export function hull(pts){
 const P=pts.slice().sort((a,b)=>a[0]-b[0]||a[1]-b[1]);
 if(P.length<3)return P;
 const cr=(o,a,b)=>(a[0]-o[0])*(b[1]-o[1])-(a[1]-o[1])*(b[0]-o[0]);
 const lo=[],up=[];
 P.forEach(p=>{
  while(lo.length>1&&cr(lo[lo.length-2],lo[lo.length-1],p)<=0)lo.pop();
  lo.push(p)});
 P.slice().reverse().forEach(p=>{
  while(up.length>1&&cr(up[up.length-2],up[up.length-1],p)<=0)up.pop();
  up.push(p)});
 lo.pop(); up.pop();
 return lo.concat(up);
}

/* ═══ تقاطع قطعتين ═══
   يعيد المعاملَين لا النقطة: التقسيم يقع على القطعة الأصلية فلا
   يتراكم خطأ التدوير. والمتوازيتان تُترَكان — الرؤوس تُسقَط عليهما
   في المرحلة التالية، وهو ما يجعل التلامس عقدةً. */
export function segInt(a,b,c,d){
 PERF.pairs++;
 const rx=b[0]-a[0], ry=b[1]-a[1];
 const sx=d[0]-c[0], sy=d[1]-c[1];
 const den=rx*sy-ry*sx;
 if(Math.abs(den)<1e-12)return null;
 const qx=c[0]-a[0], qy=c[1]-a[1];
 let t=(qx*sy-qy*sx)/den;
 let u=(qx*ry-qy*rx)/den;
 if(t<-1e-9||t>1+1e-9)return null;
 if(u<-1e-9||u>1+1e-9)return null;
 t=(t<0)?0:((t>1)?1:t);
 u=(u<0)?0:((u>1)?1:u);
 return {t,u};
}
/* ═══ تلامسُ محدَّبَين ═══
   ولِمَ لا يكفي «رأسٌ داخل الآخر»؟ شريطُ جدارٍ بسماكة ٢٠٠ يعبر
   عموداً ٤٠٠×٤٠٠ فلا رأسَ لأحدهما داخل الآخر: أضلاعُهما تتقاطع
   وحدها. فكان العمودُ على الجدار لا يُعرَف، والمتراكبان لا يُقالان.

   tol حدُّ الفصل بإشارته:
    · 0  التلامسُ الحدّيُّ تماسّ
    · +  فجوةٌ دونه تُعَدّ تماسّاً (هامش)
    · −  يشترط تداخلاً بمقداره — فتلاصقُ وجهين ليس تراكباً
   وهي إشارةُ pad في bboxHit نفسُها، فيُقرأ النداءان معاً.

   ولا تصحّ إلّا للمحدَّب: المقعَّرُ قد يُقال متلامساً وهو غيرُ
   متلامس (ولا العكسَ أبداً). ومدخلاتُ المشروع محدَّبةٌ كلُّها —
   شريطُ جدارٍ ومستطيلُ عمودٍ ومضلّعُ دائرةٍ ومستطيلُ أداة. */
export function convexHit(A,B,tol){
 const a=A||[], b=B||[];
 if(a.length<3||b.length<3)return false;
 const T=(tol==null)?0:tol;
 return !axisGap(a,b,T)&&!axisGap(b,a,T);
}
function axisGap(P,Q,T){
 for(let i=0,n=P.length;i<n;i++){
  const p=P[i], q=P[(i+1)%n];
  const ex=q[0]-p[0], ey=q[1]-p[1];
  const L=Math.hypot(ex,ey);
  if(L<1e-9)continue;                /* ضلعٌ منحلٌّ لا محورَ له */
  const nx=-ey/L, ny=ex/L;
  let a0=1/0,a1=-1/0,b0=1/0,b1=-1/0;
  for(let k=0;k<P.length;k++){
   const v=P[k][0]*nx+P[k][1]*ny;
   if(v<a0)a0=v;
   if(v>a1)a1=v;
  }
  for(let k=0;k<Q.length;k++){
   const v=Q[k][0]*nx+Q[k][1]*ny;
   if(v<b0)b0=v;
   if(v>b1)b1=v;
  }
  if(Math.min(a1,b1)-Math.max(a0,b0) < -T)return true;
 }
 return false;
}
/* ═══ شبكةٌ موحّدة الخلايا ═══
   منطق core/sindex نفسه: صندوقٌ لكل عنصر، وما امتدّ فوق حدٍّ
   يُفحَص دائماً. والخليّة تُشتَقّ من البيانات لا تُثبَّت: نموذجٌ
   بمقياس المتر وآخر بالمليمتر لا يشتركان في مقاس. */
const GBIG=64, GWIDE=4096;
function cellFor(box,n){
 if(!box||n<2)return 1000;
 const d=Math.max(box.x1-box.x0, box.y1-box.y0, 1);
 return Math.max(50, Math.min(1e7,
  Math.round(d/Math.sqrt(n)*1.5)||1000));
}
function gridOf(boxes,cell){
 const g=new Map(), big=[];
 for(let i=0;i<boxes.length;i++){
  const b=boxes[i];
  if(!b){big.push(i); continue}
  const x0=Math.floor(b.x0/cell), x1=Math.floor(b.x1/cell);
  const y0=Math.floor(b.y0/cell), y1=Math.floor(b.y1/cell);
  if((x1-x0+1)*(y1-y0+1)>GBIG){big.push(i); continue}
  for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++){
   const k=cx+","+cy;
   let a=g.get(k);
   if(!a){a=[]; g.set(k,a)}
   a.push(i);
  }
 }
 PERF.cells+=g.size;
 return {cell,g,big};
}
/* استعلامٌ بلا تخصيصٍ لكل نداء: بصمةُ دورٍ تمنع التكرار.
   يعيد -1 حين يكون الصندوق أوسع من أن يُرشَّح — فالمستدعي يمسح. */
function queryFn(G,n){
 const seen=new Int32Array(n).fill(0);
 let tick=0;
 return (b,out)=>{
  tick++;
  out.length=0;
  for(let k=0;k<G.big.length;k++){
   const i=G.big[k];
   if(seen[i]!==tick){seen[i]=tick; out.push(i)}
  }
  if(!b)return -1;
  const c=G.cell;
  const x0=Math.floor(b.x0/c), x1=Math.floor(b.x1/c);
  const y0=Math.floor(b.y0/c), y1=Math.floor(b.y1/c);
  if((x1-x0+1)*(y1-y0+1)>GWIDE)return -1;
  for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++){
   const a=G.g.get(cx+","+cy);
   if(!a)continue;
   for(let k=0;k<a.length;k++){
    const i=a[k];
    if(seen[i]!==tick){seen[i]=tick; out.push(i)}
   }
  }
  return out.length;
 };
}
/* ═══ حدُّ اللحم ═══
   ٢ مم قرارٌ معلَنٌ بحدَّيه:
   · فوق خطأ التدوير — R() يُخطئ نصف مليمتر لكل إحداثيّ، فحسابان
     لنقطةٍ واحدة يفترقان ١٫٤ مم، ومعهما إسقاطُ رأسٍ على وجهٍ
     يفترق بمقدار التفاوت نفسه.
   · ودون أنحف ما يُبنى — TMIN=50 مم، وأصغرُ حلقةٍ يقبلها الاتحاد
     ٤٠٠ مم². فلا يمكن أن يُلحَم شيءٌ ذو معنى هندسيّ.

   وفجوةٌ مقصودةٌ أضيق من ٢ مم تُلحَم. وهي أضيقُ من أن تُرسَم أو
   تُرى أو تُقاس، والسلوكُ القائم فيها أسوأ: الجدار يبدو متّصلاً
   والحلقةُ تتسرّب منه بلا كلمة. واللحمُ يُعَدّ فيُقرأ في القياس. */
export const WELD=2;

/* خريطةُ اللحم: أوّلُ ما يُصادَف في الجوار يصير ممثِّلاً.
   وخليّةُ الشبكة بمقدار التفاوت، فنقطتان دونه لا تفترقان أكثر من
   خليّةٍ في كل محور — والجوارُ ٣×٣ يكفي يقيناً.
   والترتيب يحكم الجواب، ومدخلُ polyBool مرتَّبٌ يقيناً (‏Map
   بترتيب الإدراج)، فالمخرَجُ ثابتٌ للمدخل نفسه. */
function weldMap(pts,tol){
 const T=Math.max(0.5,tol||WELD);
 const g=new Map(), rep=new Map();
 const kk=p=>R(p[0])+","+R(p[1]);
 let n=0;
 for(let i=0;i<pts.length;i++){
  const p=pts[i];
  const k=kk(p);
  if(rep.has(k))continue;
  const cx=Math.floor(p[0]/T), cy=Math.floor(p[1]/T);
  let best=null, bd=T*T;
  for(let a=-1;a<=1;a++)for(let b=-1;b<=1;b++){
   const list=g.get((cx+a)+","+(cy+b));
   if(!list)continue;
   for(let m=0;m<list.length;m++){
    const d=dist2(p,list[m]);
    if(d<=bd){bd=d; best=list[m]}
   }
  }
  if(best){rep.set(k,best); n++; continue}
  const q=[R(p[0]),R(p[1])];
  rep.set(k,q);
  const ck=cx+","+cy;
  let list=g.get(ck);
  if(!list){list=[]; g.set(ck,list)}
  list.push(q);
 }
 return {rep,n,key:kk};
}

function tag(arr,info){
 const I=info||{};
 try{
  ["open","weld","dup","nil"].forEach(k=>{
   Object.defineProperty(arr,k,{value:I[k]|0,
    enumerable:false,configurable:true,writable:true});
  });
  Object.defineProperty(arr,"at",{value:I.at||null,
   enumerable:false,configurable:true,writable:true});
  if(I.stats)Object.defineProperty(arr,"stats",{value:I.stats,
   enumerable:false,configurable:true,writable:true});
 }catch(e){}
 return arr;
}
/* ═══ الخياطة ═══
   قطعٌ غير موجَّهة ⇒ حلقاتٌ مغلقة.

   ═══ اللحم أوّلاً ═══
   رأسان لنقطةٍ واحدة يفترقان مليمتراً حين يُحسَبان من قطعتين
   مختلفتين — وجدارٌ بزاويةٍ كسريّة يُنتِج ذلك في كل ركن. فالمفتاح
   R(p) وحده يجعلهما عقدتين، فتُطرَح القطعةُ صامتةً وينفتح الحدّ
   وتغيب الحلقة، وأداةُ «منطقة» تلوم المستخدم على إغلاقٍ هو مُغلَق.
   وtol كانت مُعلَنةً في التوقيع مُهمَلةً في الشفرة.

   ═══ والتوحيد بعده لا قبله ═══
   شظيّتان متطابقتان تفترقان مليمتراً تصيران بعد اللحم قطعتين بين
   العقدتين نفسهما — فضلعٌ مزدوجٌ في الرسم البيانيّ، وإحداهما تبقى
   غير مستعملةٍ فتُعَدّ «مفتوحة». والتوحيد في polyBool يبقى
   مُرشِّحاً رخيصاً لا حاسماً.

   ═══ والعقدة تُفكّ بالزاوية ═══
   عند عقدةٍ بأربع قطعٍ (مربّعان يتلامسان برأس) أوّلُ ما يُصادَف
   يخيط الحلقتين في واحدةٍ تعبر نفسها — فتُحسَب مساحةٌ خاطئة
   وتُرسَم حدودٌ مقطوعة (الدفعة ٨ب).

   وما لم يُغلَق يُعَدّ ويُقال موضعُه، ولا يُلفَّق.
   والمُعاد مصفوفةٌ عليها خواصُّ غيرُ مُعدَّدة — فمن يقرأ length
   وforEach وJSON يبقى عاملاً. */
export function stitch(segs,tol){
 const T=Math.max(0.5,(tol==null)?WELD:tol);
 const raw=[];
 (segs||[]).forEach(s=>{
  if(!s||!s[0]||!s[1])return;
  if(!isFinite(s[0][0])||!isFinite(s[0][1])
   ||!isFinite(s[1][0])||!isFinite(s[1][1]))return;
  raw.push(s);
 });
 /* ١ — اللحم */
 const pts=[];
 for(let i=0;i<raw.length;i++){pts.push(raw[i][0],raw[i][1])}
 const W=weldMap(pts,T);
 const at=p=>W.rep.get(W.key(p))||[R(p[0]),R(p[1])];
 const K=p=>p[0]+","+p[1];
 /* ٢ — القطع بعد اللحم: الصفريّة تُطرَح والمكرّرة تُوحَّد */
 const E=[], seen=new Set();
 let dup=0, nil=0;
 for(let i=0;i<raw.length;i++){
  const a=at(raw[i][0]), b=at(raw[i][1]);
  const ka=K(a), kb=K(b);
  if(ka===kb){nil++; continue}
  const kk=(ka<kb)?(ka+"|"+kb):(kb+"|"+ka);
  if(seen.has(kk)){dup++; continue}
  seen.add(kk);
  E.push({a,b,used:0});
 }
 PERF.weld+=W.n; PERF.dup+=dup; PERF.nil+=nil;
 PERF.stitch+=E.length;
 /* ٣ — الرسم البيانيّ */
 const N=new Map();
 E.forEach((e,i)=>{
  [[K(e.a),0],[K(e.b),1]].forEach(([k,d])=>{
   let a=N.get(k);
   if(!a){a=[]; N.set(k,a)}
   const from=d?e.b:e.a, to=d?e.a:e.b;
   a.push({e:i,d,ang:Math.atan2(to[1]-from[1],to[0]-from[0])});
  });
 });
 /* ٤ — استخراج الوجوه */
 const rings=[];
 let open=0, oat=null;
 for(let i=0;i<E.length;i++){
  if(E[i].used)continue;
  const ring=[];
  const startK=K(E[i].a);
  let h={e:i,d:0}, guard=0, ok=false, lastTo=null;
  while(guard++<=E.length*2+8){
   const e=E[h.e];
   if(e.used)break;
   e.used=1;
   const from=h.d?e.b:e.a, to=h.d?e.a:e.b;
   ring.push(from);
   lastTo=to;
   const nk=K(to);
   if(nk===startK){ok=true; break}
   const list=N.get(nk);
   if(!list||list.length<2)break;
   /* دخلنا العقدة، فنخرج بالنصف الذي يلي عكسَ دخولنا دَوَراناً
      مع الساعة — وهو استخراجُ الوجوه القياسيّ. */
   const back=Math.atan2(from[1]-to[1],from[0]-to[0]);
   let nxt=null, bd=1/0;
   for(let k=0;k<list.length;k++){
    const c=list[k];
    if(E[c.e].used)continue;
    let d=back-c.ang;
    while(d<=1e-12)d+=Math.PI*2;
    while(d>Math.PI*2+1e-12)d-=Math.PI*2;
    if(d<bd){bd=d; nxt=c}
   }
   if(!nxt)break;
   h=nxt;
  }
  if(ok&&ring.length>2)rings.push(ring);
  else{
   open+=(ring.length||1);
   if(lastTo)oat=[lastTo[0],lastTo[1]];
  }
 }
 PERF.open+=open;
 if(oat)PERF.openAt=oat;
 return tag(rings,{open,weld:W.n,dup,nil,at:oat});
}
/* ═══ اتحاد المضلّعات ═══
   طريقة الشظايا: تُقسَّم الحدود عند كل تقاطعٍ وكل تلامس، ثم تُصفّى
   الشظيّة بجانبَيها، ثم تُخاط.

   والتصفية بجانبَين لا بمنتصفٍ واحد: «منتصفها داخل غيرها» يفشل
   حيث يشترك جداران وجهاً واحداً — المنتصف على الحدّ لا داخله، فتبقى
   الشظيّة نسختين وتنشقّ الخياطة. والجانبان يقولان الحقّ: الشظيّة
   على حدّ الاتحاد إن كان أحد جانبيها مغطّىً والآخر لا.

   والتلامس يُقسَّم كالتقاطع: طرفُ جدارٍ يلمس وجه آخر لا يُنشئ
   تقاطعاً (المعامل عند الطرف بالضبط)، فلو لم يُصر عقدةً لبقيت
   شظيّةٌ لا جوارَ لها.

   والفهرسة تحكم الكلفة: ألفٌ ومئتا قطعةٍ في مسكنٍ من عشرين غرفة
   تعني مليوناً وأربع مئة ألف اختبارِ تقاطعٍ بالمسح الكامل، وهي
   تُعاد مع كل تغيّرٍ هندسيّ. */
export function polyBool(polys,opt){
 const t0=now();
 const O=Object.assign({eps:1,minArea:1},opt||{});
 const eps=Math.max(0.5,O.eps);
 /* حدُّ اللحم مُعلَنٌ ومستقلّ: eps تفاوتُ الإسقاط والتقسيم، وهذا
    تفاوتُ العقدة — والثاني يجب أن يفوق الأول لأن رأسين يفترقان
    بمقدار الإسقاط ثم بخطأ التدوير معاً. */
 const wl=Math.max(WELD,eps*2,(+O.weld||0));
 PERF.union++;
 const P=(polys||[]).map(r=>cleanRing(r,0))
  .filter(r=>r&&r.length>2);
 if(!P.length)return tag([],{stats:{polys:0,frags:0,segs:0,ms:0}});
 if(P.length===1){
  const one=cleanRing(ccw(P[0]),1);
  return tag(one.length>2?[one]:[],
   {stats:{polys:1,frags:0,segs:0,ms:0}});
 }
 /* ١ — الصناديق والفهرسان */
 const pBox=P.map(r=>bboxOf(r));
 const PG=gridOf(pBox,cellFor(bboxAll(pBox),P.length));
 const qPoly=queryFn(PG,P.length);

 const SA=[], SB=[], SP=[];
 P.forEach((r,pi)=>{
  for(let i=0;i<r.length;i++){
   SA.push(r[i]); SB.push(r[(i+1)%r.length]); SP.push(pi);
  }
 });
 const ns=SA.length;
 const sBox=[];
 for(let i=0;i<ns;i++)sBox.push(bboxOf([SA[i],SB[i]]));
 const SG=gridOf(sBox,cellFor(bboxAll(sBox),ns));
 const qSeg=queryFn(SG,ns);

 /* ٢ — معاملات التقسيم */
 const cuts=new Array(ns);
 for(let i=0;i<ns;i++)cuts[i]=[0,1];
 const cand=[];
 for(let i=0;i<ns;i++){
  const A=SA[i], B=SB[i];
  const n2=qSeg(bboxPad(sBox[i],eps),cand);
  const full=(n2<0);
  const lim=full?ns:cand.length;
  for(let k=0;k<lim;k++){
   const j=full?k:cand[k];
   if(j===i||SP[j]===SP[i])continue;
   const C=SA[j], D=SB[j];
   const x=segInt(A,B,C,D);
   if(x){cuts[i].push(x.t); continue}
   /* التلامس والانطباق: رؤوسُ الأخرى تُسقَط على هذه */
   const r1=nearOnSeg(A,B,C[0],C[1]);
   if(r1.d<=eps)cuts[i].push(r1.t);
   const r2=nearOnSeg(A,B,D[0],D[1]);
   if(r2.d<=eps)cuts[i].push(r2.t);
  }
 }
 /* ٣ — الشظايا · مفتاحٌ غير مرتَّب ⇒ نسخةٌ واحدة.
    وهذا ترشيحٌ رخيصٌ لا حاسم: الحاسمُ في stitch بعد اللحم، لأن
    نسختين تفترقان مليمتراً لا يجمعهما مفتاحٌ مُدوَّر. */
 const key=p=>R(p[0])+","+R(p[1]);
 const F=new Map();
 for(let i=0;i<ns;i++){
  const A=SA[i], B=SB[i], L=dist(A,B);
  if(L<eps)continue;
  const ts=cuts[i].filter(t=>t>=0&&t<=1).sort((x,y)=>x-y);
  for(let k=0;k+1<ts.length;k++){
   if((ts[k+1]-ts[k])*L<eps)continue;
   const a=[R(A[0]+(B[0]-A[0])*ts[k]),   R(A[1]+(B[1]-A[1])*ts[k])];
   const b=[R(A[0]+(B[0]-A[0])*ts[k+1]), R(A[1]+(B[1]-A[1])*ts[k+1])];
   const ka=key(a), kb=key(b);
   if(ka===kb)continue;
   PERF.frags++;
   const kk=(ka<kb)?(ka+"|"+kb):(kb+"|"+ka);
   if(!F.has(kk))F.set(kk,{a,b});
  }
 }
 /* ٤ — التصفية بجانبَين */
 const pc=[];
 const covered=(x,y)=>{
  const n2=qPoly({x0:x,y0:y,x1:x,y1:y},pc);
  const full=(n2<0);
  const lim=full?P.length:pc.length;
  for(let k=0;k<lim;k++){
   const i=full?k:pc[k];
   if(!bboxIn(pBox[i],x,y,0))continue;
   if(pip(P[i],x,y))return true;
  }
  return false;
 };
 /* الإزاحة دون أنحف ما نبنيه (٥٠ مم سماكةً دُنيا) ولا تحت
    خطأ التدوير — فالمِجَسّ يقع في الجانب لا على الحدّ. */
 const off=Math.max(2,eps*2);
 const segs=[];
 F.forEach(f=>{
  const m=mid(f.a,f.b);
  const dx=f.b[0]-f.a[0], dy=f.b[1]-f.a[1];
  const L=Math.hypot(dx,dy)||1;
  const nx=-dy/L*off, ny=dx/L*off;
  const s1=covered(m[0]+nx, m[1]+ny);
  const s2=covered(m[0]-nx, m[1]-ny);
  if(s1===s2)return;         /* داخليّةٌ أو شاذّة */
  PERF.kept++;
  segs.push([f.a,f.b]);
 });
 /* ٥ — الخياطة والتنظيف */
 const st=stitch(segs,wl);
 const rings=[];
 st.forEach(r=>{
  const c=cleanRing(r,1);
  if(c.length<3)return;
  if(Math.abs(pArea(c))<O.minArea)return;
  rings.push(c);
 });
 PERF.rings+=rings.length;
 const ms=now()-t0;
 PERF.ms+=ms;
 return tag(rings,{open:st.open|0, weld:st.weld|0,
  dup:st.dup|0, nil:st.nil|0, at:st.at,
  stats:{polys:P.length, segs:segs.length, frags:F.size,
   weld:st.weld|0, dup:st.dup|0, ms:Math.round(ms)}});
}
```

<a id="f-js-core-inspect-js"></a>

---

## `js/core/ents.js`

```javascript
/* ═══ الكيانات: السياسة ═══
   الجدول في entreg.js، والسياسة هنا: ما يُحدَّد ويُعدَّل ويُحذَف.
   والمخفيّ والمقفل يُتخطّى لا يُعاد — ما لا يُحدَّد لا يُعدَّل ولا
   يُحذَف، وهذا الحرس أحقّ بموضعٍ واحد لا تسعة.

   الملفّ كان ٤٤٠ سطراً من الشروط المتسلسلة؛ صار تفويضاً. ولا
   سلوكَ تغيّر: ترتيب الإصابة نفسه، ومرشِّح الالتقاط يُمرَّر إلى
   حلقة كل نوعٍ كما كان — فالفتحة المخفيّة تُتخطّى وتستمرّ الحلقة،
   والجدار الأفضل إن كان مقفلاً يُتخطّى نوعُه كلّه. */
import {S,touch,touchGeom,touchOpen,touchView} from "./state.js";
import {shapeInRect} from "./geom.js";
import {ENT,ORD,HORD,KINDS,COLL,NAME,entDef} from "./entreg.js";
import {pickable} from "./layers.js";
import * as SI from "./sindex.js";

export {COLL,NAME,KINDS,entDef};
export const KORDER=KINDS;

/* البحث بالمعرّف — لسطر الإدخال · غير حسّاس لحالة الحرف */
export function findById(id){
 const q=String(id||"").trim().toUpperCase();
 if(!q)return null;
 for(const d of ORD){
  const e=(S[d.coll]||[]).find(x=>
   String(x.id).toUpperCase()===q);
  if(e)return {k:d.k,id:e.id};
 }
 return null;
}
export function entOf(s){
 if(!s)return null;
 const d=ENT[s.k];
 return d?d.byId(s.id):null;
}
const of=(s,fn,dflt)=>{
 if(!s)return dflt;
 const d=ENT[s.k];
 if(!d||!d[fn])return dflt;
 const e=d.byId(s.id);
 return e?d[fn](e):dflt;
};
/* ═══ الإصابة ═══ بترتيب الصِّغَر: الأداة أوّلاً والمنطقة آخراً ═══
   صندوقٌ واحد من الفهرس يخدم الأنواع كلّها. ويُوسَّع بـ 1.25 من
   التفاوت لأن التأشير يُصاب بـ T×1.2، وبـ 220 على الأقلّ لأن
   wallAt يقبل تفاوته الخاصّ (200) حين لا يُمرَّر إليه شيء. */
export function hitTest(x,y,tol){
 const T=tol||150;
 const Q=SI.query(SI.boxAt(x,y,Math.max(T*1.25,220)));
 for(const d of HORD){
  const c=Q[d.k];
  if(!c||!c.length)continue;
  const cand=c.map(r=>r.e);
  const ok=id=>pickable({k:d.k,id});
  const e=d.hit(x,y,T,tol,ok,cand);
  if(!e)continue;
  const s={k:d.k,id:e.id};
  if(pickable(s))return s;
 }
 return null;
}
/* ═══ الشكل والمحيط ═══ */
export const shapeOf  =s=>of(s,"shape",null);
export const outlineOf=s=>of(s,"outline",null);

/* ═══ المقابض ═══ المقفل يُرى ولا مقابض له ═══ */
export const gripsOf=s=>pickable(s)?of(s,"grips",[]):[];

/* ═══ أيَّ نسخةٍ يُقدّم تعديلُ هذا التحديد ═══
   من الجدول لا من شرطٍ مبثوث. وأقوى ما في القائمة يفوز: تحديدٌ
   فيه جدارٌ وبُعدٌ يُقدّم النسخة الهندسية. والمجهول geom لأن
   الافتراض الآمن يُكلِّف أداءً لا صحّة. */
const BW={geom:3,open:2,view:1};
export function bumpOf(list){
 let best="view", bw=1;
 (list||[]).forEach(s=>{
  const d=s&&ENT[s.k];
  const b=(d&&d.bump)||"geom";
  const w=BW[b]||3;
  if(w>bw){bw=w; best=b}
 });
 return best;
}
export const touchFn=b=>(b==="view")?touchView
 :((b==="open")?touchOpen:touchGeom);
export const isGeom=s=>bumpOf([s])==="geom";

/* لقطة قبل السحب — كل تحويل يُحسب من الأصل لا من الحالة الجارية */
export const grabOf=s=>of(s,"grab",null);

/* ═══ السحب ═══
   يتوقّف عند الحدّ ولا يُرفض: التوقّف مرئي فلا مفاجأة فيه. */
export function dragGrip(g,o,p,dx,dy){
 if(!o||!g||!g.s)return;
 const d=ENT[g.s.k];
 if(d&&d.drag)d.drag(o,g,p,dx,dy);
}
export function moveEnt(s,o,dx,dy){
 if(!o||!s)return;
 const d=ENT[s.k];
 if(d&&d.move)d.move(o,dx,dy);
}
/* ═══ الحذف ═══
   الحاضن يجرّ محتضنه: حذف الجدار يحذف فتحاته ويُبلَّغ العددُ لأنه
   فقدٌ لم تطلبه صراحة — وذلك بـ cascade في الجدول لا بشرطٍ هنا.
   المناطق لا تُحذَف بحذف جدار: تصير «قديمة» وأنت تقرّر. */
export function delEnts(list){
 const c={};
 ORD.forEach(d=>{c[d.coll]=0});
 const done={};
 let skip=0;
 (list||[]).forEach(s=>{
  /* الحرس الأخير: لا يُحذَف ما لا يُحدَّد — ولو وصل هنا */
  if(!pickable(s)){skip++; return}
  const d=ENT[s.k];
  if(!d)return;
  const e=d.byId(s.id);
  if(!e||!d.del(e))return;
  c[d.coll]++;
  (done[d.k]=done[d.k]||new Set()).add(s.id);
 });
 ORD.forEach(d=>{
  if(!d.cascade||!done[d.k]||!done[d.k].size)return;
  const add=d.cascade(done[d.k])||{};
  Object.keys(add).forEach(k=>{c[k]=(c[k]||0)+add[k]});
 });
 touch();
 c.skipped=skip;
 return c;
}
export function pickInRect(r,win,add,prev){
 const out=add?(prev||[]).slice():[];
 /* المخفيّ والمقفل يُعَدّان «موجودَين سلفاً» فلا يُضافان */
 const has=s=>out.some(x=>x.k===s.k&&x.id===s.id)||!pickable(s);
 const Q=SI.query(r);
 ORD.forEach(d=>(Q[d.k]||[]).forEach(rec=>{
  const s={k:d.k,id:rec.e.id};
  if(has(s))return;
  const sh=d.shape?d.shape(rec.e):null;
  if(sh&&shapeInRect(sh,r,win))out.push(s);
 }));
 return out;
}
export const allEnts=()=>{
 const out=[];
 ORD.forEach(d=>(S[d.coll]||[]).forEach(e=>out.push({k:d.k,id:e.id})));
 return out;
};
export const pickEnts=()=>allEnts().filter(pickable);

export const delSay=r=>{
 const P=[];
 ORD.forEach(d=>{if(r[d.coll])P.push(`${r[d.coll]} ${d.n}`)});
 return P.join(" و ")||"لا شيء";
};
```

<a id="f-js-core-fixt-js"></a>

---

## `js/core/entreg.js`

```javascript
/* ═══ سجلّ الأنواع ═══
   جدولٌ بدل ثماني سلاسل من الشروط. قبله كانت إضافة نوعٍ تعني
   تعديل ents.js في ثمانية مواضع و layers.js في موضعين
   و modify.js في موضع — وأيُّ موضعٍ يُنسى يعطب صامتاً: كيانٌ
   يُرسَم ولا يُحدَّد، أو يُحدَّد ولا يُحذَف.
   بعده: نوعٌ واحد = سطرٌ واحد هنا.

   وهو جدولٌ خالص لا يعرف الطبقات ولا حالتها: التصفية سياسةٌ
   تسكن ents.js، فلا دورةَ استيراد مع layers.js — بل layers.js
   يقرأ منه lay(e) فيسقط عنه معرفة الأنواع كلّها.

   الترتيبان مقصودان:
     hitO  ترتيب الإصابة — الأصغر أوّلاً فلا يحجب الجدارُ فتحته.
     pick  ترتيب العدّ والتقرير — يخدم pickInRect و allEnts
           و delSay و groupOrder، فلا أربع قوائم تتفرّق. */
import {S} from "./state.js";
import {clamp,deg} from "./units.js";
import {pip,nearOnSeg,bboxOf} from "./geom.js";
import {dir,band,wallById,wallLen,wallAt,delWall,
        isLow} from "./walls.js";
import {opensOf,openById,openPt,span,sAt,delOpen,nearestFree,okOf,
        MINW,EDGE} from "./opens.js";
import {areaById,areaAt,delArea,labelPt} from "./areas.js";
import {dimById,chainById,annoById,dimGeom,dimMid,chainPt,
        chainBounds,annoPt,delDim,delChain,delAnno,
        posFromPt} from "./dims.js";
import {colById,colPoly,colW,delCol} from "./cols.js";
import {fixById,fixPoly,fixW,fixD,frameOf,delFix} from "./fixt.js";
import {stById,stPoly,stGeom,delStair,SMIN_W} from "./stairs.js";

const R=v=>Math.round(v);
export const ENT={};
export function defEnt(d){ENT[d.k]=d; return d}

/* ═══ الجدار ═══ */
defEnt({k:"wall",coll:"walls",n:"جدار",pre:"W",pick:1,hitO:7,
 /* bump: أيَّ نسخةٍ يُقدّم تعديلُه — geom يُبطِل الاتحاد والحلقات
    وشبكة الأطراف والمراسي والبصمات · open الأجسام وحدها ·
    view لا شيء منها. والمجهول يُعَدّ geom: الافتراض آمن. */
 bump:"geom",
 byId:wallById,
 lay:w=>isLow(w)?"A-WALL-LOW":"A-WALL",
 /* wallAt يفضّل الأقرب إلى المحور ولا يقبل مرشِّحاً — فإن كان
    الأفضل مخفيّاً أو مقفلاً يُتخطّى النوع كلّه، كما كان */
 hit:(x,y,T,tol,ok,cand)=>wallAt(x,y,tol,cand),
 shape:w=>({t:"seg",a:w.a,b:w.b}),
 outline:w=>band(w),
 grips:w=>[{p:w.a.slice(),k:"a"},
  {p:[R((w.a[0]+w.b[0])/2),R((w.a[1]+w.b[1])/2)],k:"mid"},
  {p:w.b.slice(),k:"b"}],
 grab:w=>({e:w,a:w.a.slice(),b:w.b.slice()}),
 drag(o,g,p,dx,dy){
  const w=o.e;
  if(g.k==="a")w.a=[p[0],p[1]];
  else if(g.k==="b")w.b=[p[0],p[1]];
  else{w.a=[o.a[0]+dx,o.a[1]+dy]; w.b=[o.b[0]+dx,o.b[1]+dy]}
 },
 move(o,dx,dy){
  o.e.a=[o.a[0]+dx,o.a[1]+dy];
  o.e.b=[o.b[0]+dx,o.b[1]+dy];
 },
 del:delWall,
 /* حذف الجدار يحذف فتحاته: الحاضن زال فلا معنى لبقائها،
    ويُبلَّغ العدد لأنه فقدٌ لم تطلبه صراحة */
 cascade(ids){
  const kill=S.opens.filter(o=>ids.has(o.wall));
  S.opens=S.opens.filter(o=>!ids.has(o.wall));
  return {opens:kill.length};
 }});

/* ═══ الفتحة ═══ */
defEnt({k:"open",coll:"opens",n:"فتحة",pre:"O",pick:2,hitO:6,
 bump:"open",
 byId:openById,
 lay:o=>okOf(o.kind).lay,
 /* منطقة الإصابة تختلف عن الشكل: openPt يُزيح بالمحاذاة، ونصفُ
    العرض يدخل في نصف قطر الإصابة — فتُعلَن للفهرس صريحاً */
 hbox(o){
  const w=wallById(o.wall);
  if(!w)return null;
  const [a,b]=span(o);
  const B=bboxOf([openPt(w,a),openPt(w,b),openPt(w,o.s)]);
  if(!B)return null;
  const r=o.w/2;
  return {x0:B.x0-r,y0:B.y0-r,x1:B.x1+r,y1:B.y1+r};
 },
 hit(x,y,T,tol,ok,cand){
  for(const o of (cand||S.opens)){
   if(!ok(o.id))continue;
   const w=wallById(o.wall);
   if(!w)continue;
   const c=openPt(w,o.s);
   if(Math.hypot(x-c[0],y-c[1])<Math.max(o.w/2,T))return o;
  }
  return null;
 },
 shape(o){
  const w=wallById(o.wall);
  if(!w)return null;
  const d=dir(w);
  if(!d)return null;
  const [a,b]=span(o);
  return {t:"seg",
   a:[R(w.a[0]+d.ux*a),R(w.a[1]+d.uy*a)],
   b:[R(w.a[0]+d.ux*b),R(w.a[1]+d.uy*b)]};
 },
 grips(o){
  const w=wallById(o.wall);
  if(!w)return [];
  const [a,b]=span(o);
  return [{p:openPt(w,o.s),k:"c"},
          {p:openPt(w,a),k:"e0"},
          {p:openPt(w,b),k:"e1"}];
 },
 grab:o=>({e:o,s:o.s,w:o.w}),
 drag(o,g,p){
  const op=o.e, w=wallById(op.wall);
  if(!w)return;
  const s=sAt(w,p), L=wallLen(w);
  if(g.k==="c"){
   /* الفترات الحرّة لا الغلاف: السحب لا يعبر فتحةً قائمة.
      يتوقّف عند الحدّ ولا يرفض — التوقّف مرئيٌّ فلا مفاجأة فيه.
      وكان القصّ على [lo,hi] يُنشئ clash في أشهر تفاعلٍ في
      البرنامج، بينما مقبض الحدّ والمُثبِّت وaddOpen يرفضونه. */
   const q=nearestFree(w,op.w,s,op);
   if(q!=null)op.s=q;
   return;
  }
  /* حدّ الفتحة: يغيّر العرض والمركز معاً والطرف الآخر ثابت */
  const fix=(g.k==="e0")?(o.s+o.w/2):(o.s-o.w/2);
  let lo=Math.min(fix,s), hi=Math.max(fix,s);
  lo=Math.max(lo,EDGE); hi=Math.min(hi,L-EDGE);
  if(hi-lo<MINW)return;
  const nw=R(hi-lo), ns=R((lo+hi)/2);
  for(const x of opensOf(w.id)){
   if(x===op)continue;
   const [a,b]=span(x);
   if(ns-nw/2<b-1&&a<ns+nw/2-1)return;
  }
  op.w=nw; op.s=ns;
 },
 move(o,dx,dy){
  /* الفتحة تنزلق على جدارها — الإزاحة تُسقَط على مساره، ثم
     تُقصَر على الفترة الحرّة لا على الغلاف */
  const w=wallById(o.e.wall);
  if(!w)return;
  const d=dir(w);
  if(!d)return;
  const q=nearestFree(w,o.e.w,R(o.s+dx*d.ux+dy*d.uy),o.e);
  if(q!=null)o.e.s=q;
 },
 del:delOpen,
 noDup:1});          /* تُنسَخ مع جدارها لا وحدها */

/* ═══ المنطقة ═══ */
defEnt({k:"area",coll:"areas",n:"منطقة",pre:"A",pick:6,hitO:9,
 bump:"view",
 byId:areaById,
 lay:()=>"A-AREA",
 hit:(x,y,T,tol,ok,cand)=>areaAt(x,y,cand),
 shape:a=>({t:"poly",pts:a.ring}),
 outline:a=>a.ring,
 grips(a){
  const g=[{p:labelPt(a),k:"L"}];
  if(a.ring.length<=40)
   a.ring.forEach((p,i)=>g.push({p:p.slice(),k:"v"+i}));
  return g;
 },
 grab:a=>({e:a,ring:a.ring.map(p=>p.slice()),
  lp:a.lp?a.lp.slice():null, lc:labelPt(a)}),
 drag(o,g,p){
  const a=o.e;
  /* سحب التسمية يجعل موضعها صريحاً، فلا تزحف بعدها أبداً */
  if(g.k==="L"){a.lp=[p[0],p[1]]; return}
  const i=parseInt(g.k.slice(1),10);
  if(!(i>=0&&i<a.ring.length))return;
  a.ring[i]=[p[0],p[1]];
 },
 move(o,dx,dy){
  o.e.ring=o.ring.map(p=>[p[0]+dx,p[1]+dy]);
  if(o.lp)o.e.lp=[o.lp[0]+dx,o.lp[1]+dy];
 },
 del:delArea});

/* ═══ البُعد ═══ */
defEnt({k:"dim",coll:"dims",n:"بُعد",pre:"D",pick:7,hitO:4,
 bump:"view",
 byId:dimById,
 lay:()=>"A-DIMS",
 hit(x,y,T,tol,ok,cand){
  for(const d of (cand||S.dims)){
   if(!ok(d.id))continue;
   const g=dimGeom(d);
   if(g&&nearOnSeg(g.p1,g.p2,x,y).d<T)return d;
  }
  return null;
 },
 shape(d){
  const g=dimGeom(d);
  return g?{t:"seg",a:g.p1,b:g.p2}:null;
 },
 grips:d=>[{p:d.a.slice(),k:"a"},{p:d.b.slice(),k:"b"},
  {p:dimMid(d),k:"pos"}],
 grab:d=>({e:d,kind:d.kind,a:d.a.slice(),b:d.b.slice(),pos:d.pos}),
 drag(o,g,p){
  const d=o.e;
  if(g.k==="a")d.a=[p[0],p[1]];
  else if(g.k==="b")d.b=[p[0],p[1]];
  else d.pos=posFromPt(d.kind,d.a,d.b,p);
 },
 move(o,dx,dy){
  o.e.a=[o.a[0]+dx,o.a[1]+dy];
  o.e.b=[o.b[0]+dx,o.b[1]+dy];
  o.e.pos=(o.e.kind==="h")?(o.pos+dy)
   :((o.e.kind==="v")?(o.pos+dx):o.pos);
 },
 del:delDim});

/* ═══ السلسلة ═══ */
defEnt({k:"chain",coll:"chains",n:"سلسلة",pre:"C",pick:8,hitO:5,
 bump:"view",
 byId:chainById,
 lay:()=>"A-DIMS",
 hit(x,y,T,tol,ok,cand){
  for(const c of (cand||S.chains)){
   if(!ok(c.id))continue;
   const B=chainBounds(c);
   if(B.length<2)continue;
   if(nearOnSeg(chainPt(c,B[0]),chainPt(c,B[B.length-1]),x,y).d<T)
    return c;
  }
  return null;
 },
 shape(c){
  const B=chainBounds(c);
  return {t:"seg",a:chainPt(c,B[0]),b:chainPt(c,B[B.length-1])};
 },
 grips(c){
  const B=chainBounds(c);
  return [{p:chainPt(c,B[0]),k:"base"},
          {p:chainPt(c,B[B.length-1]),k:"end"}];
 },
 grab:c=>({e:c,axis:c.axis,base:c.base.slice(),pos:c.pos}),
 drag(o,g,p,dx,dy){
  /* الطرفان يحرّكان السلسلة كاملةً: القيَم مكتوبة ولا تُشَدّ */
  const c=o.e;
  c.base=[o.base[0]+dx,o.base[1]+dy];
  c.pos=(c.axis==="h")?p[1]:p[0];
 },
 move(o,dx,dy){
  o.e.base=[o.base[0]+dx,o.base[1]+dy];
  o.e.pos=(o.e.axis==="h")?(o.pos+dy):(o.pos+dx);
 },
 del:delChain});

/* ═══ التأشير ═══ */
defEnt({k:"anno",coll:"anno",n:"تأشير",pre:"T",pick:9,hitO:3,
 bump:"view",
 byId:annoById,
 lay:()=>"A-ANNO",
 /* القائد يُصاب على كل قطعةٍ من مساره، لا على وترِ طرفيه */
 hbox:a=>(a.kind==="lead")?bboxOf(a.pts):null,
 hit(x,y,T,tol,ok,cand){
  for(const a of (cand||S.anno)){
   if(!ok(a.id))continue;
   const p=annoPt(a);
   if(Math.hypot(x-p[0],y-p[1])<T*1.2)return a;
   if(a.kind==="lead"){
    for(let i=0;i<a.pts.length-1;i++)
     if(nearOnSeg(a.pts[i],a.pts[i+1],x,y).d<T)return a;
   }
  }
  return null;
 },
 shape(a){
  if(a.kind==="lead")
   return {t:"seg",a:a.pts[0],b:a.pts[a.pts.length-1]};
  return {t:"pt",p:[a.x,a.y]};
 },
 grips(a){
  if(a.kind!=="lead")return [{p:[a.x,a.y],k:"p"}];
  return a.pts.map((p,i)=>({p:p.slice(),k:"p"+i}));
 },
 grab:a=>({e:a,x:a.x,y:a.y,
  pts:a.pts?a.pts.map(p=>p.slice()):null}),
 drag(o,g,p){
  const a=o.e;
  if(a.kind!=="lead"){a.x=p[0]; a.y=p[1]; return}
  const i=parseInt(g.k.slice(1),10);
  if(i>=0&&i<a.pts.length)a.pts[i]=[p[0],p[1]];
 },
 move(o,dx,dy){
  if(o.pts)o.e.pts=o.pts.map(p=>[p[0]+dx,p[1]+dy]);
  else{o.e.x=o.x+dx; o.e.y=o.y+dy}
 },
 del:delAnno});

/* ═══ العمود ═══ */
defEnt({k:"col",coll:"cols",n:"عمود",pre:"K",pick:3,hitO:2,
 bump:"geom",
 byId:colById,
 lay:()=>"A-COLS",
 hit(x,y,T,tol,ok,cand){
  for(const c of (cand||S.cols)){
   if(!ok(c.id))continue;
   const p=colPoly(c);
   if(p&&pip(p,x,y))return c;
  }
  return null;
 },
 shape:c=>({t:"poly",pts:colPoly(c)}),
 outline:c=>colPoly(c),
 grips(c){
  const g=[{p:[c.x,c.y],k:"c"}];
  if(c.kind==="circ")g.push({p:[R(c.x+colW(c)/2),c.y],k:"r"});
  else{
   const p=colPoly(c);
   g.push({p:p[2].slice(),k:"sz"});
   g.push({p:[R((p[1][0]+p[2][0])/2),R((p[1][1]+p[2][1])/2)],
    k:"rot"});
  }
  return g;
 },
 grab:c=>({e:c,x:c.x,y:c.y,w:c.w,h:c.h,rot:c.rot}),
 drag(o,g,p){
  const c=o.e;
  if(g.k==="c"){c.x=p[0]; c.y=p[1]; return}
  if(g.k==="r"){
   c.w=clamp(R(Math.hypot(p[0]-c.x,p[1]-c.y)*2),100,4000);
   c.h=c.w; return;
  }
  if(g.k==="rot"){
   c.rot=deg(Math.round(
    Math.atan2(p[1]-c.y,p[0]-c.x)*180/Math.PI*10)/10);
   return;
  }
  /* المقاس من الرُّكن: يُقاس في الإطار المحلّي فلا يتأثّر بالدوران */
  const a=(c.rot||0)*Math.PI/180;
  const dxl=(p[0]-c.x)*Math.cos(a)+(p[1]-c.y)*Math.sin(a);
  const dyl=-(p[0]-c.x)*Math.sin(a)+(p[1]-c.y)*Math.cos(a);
  c.w=clamp(R(Math.abs(dxl)*2),100,4000);
  c.h=clamp(R(Math.abs(dyl)*2),100,4000);
 },
 move(o,dx,dy){o.e.x=o.x+dx; o.e.y=o.y+dy},
 del:delCol,
 dupDrop:["tag"]});   /* الوسم لا يُنسَخ — يُرقَّم */

/* ═══ الأداة الصحية ═══ */
defEnt({k:"fix",coll:"fixt",n:"أداة",pre:"F",pick:4,hitO:1,
 bump:"view",
 byId:fixById,
 lay:()=>"A-FIXT",
 hit(x,y,T,tol,ok,cand){
  for(const f of (cand||S.fixt)){
   if(!ok(f.id))continue;
   if(pip(fixPoly(f),x,y))return f;
  }
  return null;
 },
 shape:f=>({t:"poly",pts:fixPoly(f)}),
 outline:f=>fixPoly(f),
 grips(f){
  const P=frameOf(f);
  return [{p:[f.x,f.y],k:"c"},
          {p:P(0,fixD(f)),k:"rot"},
          {p:P(fixW(f)/2,fixD(f)),k:"sz"}];
 },
 grab:f=>({e:f,x:f.x,y:f.y,w:f.w,d:f.d,rot:f.rot}),
 drag(o,g,p){
  const f=o.e;
  if(g.k==="c"){f.x=p[0]; f.y=p[1]; return}
  if(g.k==="rot"){
   f.rot=deg(Math.round(
    (Math.atan2(p[1]-f.y,p[0]-f.x)*180/Math.PI-90)*10)/10);
   return;
  }
  const a=(f.rot||0)*Math.PI/180;
  const u=(p[0]-f.x)*Math.cos(a)+(p[1]-f.y)*Math.sin(a);
  const v=-(p[0]-f.x)*Math.sin(a)+(p[1]-f.y)*Math.cos(a);
  f.w=clamp(R(Math.abs(u)*2),80,4000);
  f.d=clamp(R(Math.abs(v)),80,4000);
 },
 move(o,dx,dy){o.e.x=o.x+dx; o.e.y=o.y+dy},
 del:delFix});

/* ═══ الدرج ═══ */
defEnt({k:"stair",coll:"stairs",n:"درج",pre:"S",pick:5,hitO:8,
 bump:"view",
 byId:stById,
 lay:()=>"A-STRS",
 hit(x,y,T,tol,ok,cand){
  for(const t of (cand||S.stairs)){
   if(!ok(t.id))continue;
   const p=stPoly(t);
   if(p&&pip(p,x,y))return t;
  }
  return null;
 },
 shape(t){
  const p=stPoly(t);
  return p?{t:"poly",pts:p}:null;
 },
 outline:t=>stPoly(t),
 grips(t){
  const g=stGeom(t);
  if(!g)return [];
  return [{p:t.a.slice(),k:"a"},{p:t.b.slice(),k:"b"},
          {p:g.P(g.L/2,0),k:"mid"},
          {p:g.P(g.L/2,g.hw),k:"w"}];
 },
 grab:t=>({e:t,a:t.a.slice(),b:t.b.slice(),w:t.w}),
 drag(o,g,p,dx,dy){
  const t=o.e;
  if(g.k==="a"){t.a=[p[0],p[1]]; return}
  if(g.k==="b"){t.b=[p[0],p[1]]; return}
  if(g.k==="mid"){
   t.a=[o.a[0]+dx,o.a[1]+dy];
   t.b=[o.b[0]+dx,o.b[1]+dy];
   return;
  }
  const G=stGeom({a:o.a,b:o.b,w:o.w,n:t.n});
  if(!G)return;
  const v=(p[0]-o.a[0])*G.nx+(p[1]-o.a[1])*G.ny;
  t.w=clamp(R(Math.abs(v)*2),SMIN_W,6000);
 },
 move(o,dx,dy){
  o.e.a=[o.a[0]+dx,o.a[1]+dy];
  o.e.b=[o.b[0]+dx,o.b[1]+dy];
 },
 del:delStair});

/* ═══ الترتيبان ═══ تُبنى مرّةً بعد الإعلان كلّه ═══ */
const V=Object.keys(ENT).map(k=>ENT[k]);
export const ORD =V.slice().sort((a,b)=>a.pick-b.pick);
export const HORD=V.slice().sort((a,b)=>a.hitO-b.hitO);
export const KINDS=ORD.map(d=>d.k);
export const COLL=ORD.reduce((o,d)=>{o[d.k]=d.coll; return o},{});
export const NAME=ORD.reduce((o,d)=>{o[d.k]=d.n;    return o},{});
export const entDef=k=>ENT[k]||null;
```

<a id="f-js-core-ents-js"></a>

---

## `js/core/coords.js`

```javascript
/* ═══ فكّ الإحداثيات · التقييد الزاوي ═══
   مساعدة إدخال خالصة: تعينك على إصابة النقطة التي قصدتها،
   ولا تُعدّل شيئاً بعد وقوعها. */
import {M,norm,D2R,R2D,deg} from "./units.js";
export {D2R,R2D};

export const polar=(o,L,a)=>
 [Math.round(o[0]+L*Math.cos(a*D2R)),Math.round(o[1]+L*Math.sin(a*D2R))];
export const angOf=(a,b)=>deg(Math.atan2(b[1]-a[1],b[0]-a[0])*R2D);
export const lenOf=(a,b)=>Math.hypot(b[0]-a[0],b[1]-a[1]);

/* 3,4 مطلق · @5,3 نسبي · 5<45 قطبي · @5<45 قطبي نسبي
   5 مسافة في الاتجاه الحالي · <45 قفل زاوية · 3x4 مقاس */
export function parsePt(s,base,dir){
 s=norm(s).replace(/\s+/g,"");
 if(!s)return null;
 let rel=false;
 if(s[0]==="@"){
  rel=true; s=s.slice(1);
  if(!base)return {k:"err",m:"@ يحتاج نقطة أساس"};
 }
 let m=/^<(-?\d+(?:\.\d+)?)$/.exec(s);
 if(m)return {k:"ang",a:+m[1]};
 m=/^(-?\d*\.?\d+(?:mm|cm|m|مم|سم|م)?)<(-?\d+(?:\.\d+)?)$/.exec(s);
 if(m){
  const L=M(m[1]), o=rel?base:[0,0];
  return {k:"pt",p:polar(o,L,+m[2])};
 }
 m=/^(-?\d*\.?\d+)[,](-?\d*\.?\d+)$/.exec(s);
 if(m){
  const x=M(m[1]), y=M(m[2]);
  return {k:"pt",p:rel?[Math.round(base[0]+x),Math.round(base[1]+y)]
                      :[x,y]};
 }
 m=/^(-?\d*\.?\d+)[x*](-?\d*\.?\d+)$/.exec(s);
 if(m)return {k:"dim",w:M(m[1]),d:M(m[2])};
 m=/^(-?\d*\.?\d+(?:mm|cm|m|مم|سم|م)?)$/.exec(s);
 if(m){
  const L=M(m[1]);
  if(!base)return {k:"len",L};
  if(!dir)return {k:"err",m:"حرّك المؤشر لتحديد الاتجاه ثم اكتب المسافة"};
  return {k:"pt",dde:1,
   p:[Math.round(base[0]+dir[0]*L),Math.round(base[1]+dir[1]*L)]};
 }
 return null;
}
export function trackAngles(mode,inc,extra){
 if(mode==="ortho")return [0,90,180,270];
 if(mode!=="polar")return null;
 const st=Math.max(1,Math.min(90,inc||15)), A=[];
 for(let a=0;a<360;a+=st)A.push(a);
 (extra||[]).forEach(v=>{const x=deg(+v); if(!A.includes(x))A.push(x)});
 return A.sort((a,b)=>a-b);
}
/* يقيّد p على أقرب زاوية متاحة — إسقاط عمودي */
export function constrain(base,p,mode,inc,extra,tolDeg){
 if(!base)return null;
 const A=trackAngles(mode,inc,extra); if(!A)return null;
 const dx=p[0]-base[0], dy=p[1]-base[1];
 if(Math.hypot(dx,dy)<1)return null;
 const a=deg(Math.atan2(dy,dx)*R2D);
 let best=null,bd=1e9;
 A.forEach(t=>{
  let d=Math.abs(t-a); if(d>180)d=360-d;
  if(d<bd){bd=d;best=t}
 });
 if(best==null)return null;
 const tol=(tolDeg!=null)?tolDeg
  :((mode==="ortho")?90:Math.min(12,(inc||15)/2));
 if(bd>tol)return null;
 const ux=Math.cos(best*D2R), uy=Math.sin(best*D2R);
 const t=dx*ux+dy*uy;
 return {a:best,L:t,
  p:[Math.round(base[0]+ux*t),Math.round(base[1]+uy*t)]};
}
```

<a id="f-js-core-dims-js"></a>

---

## `js/core/blocks.js`

```javascript
/* ═══ مكتبة العناصر/الرموز (Blocks) ═══
   الكتلة مجموعة أوّليات محلية بالمليمتر. المثيل مرجع وتحويل
   (موضع/دوران/مقياس/مرآة/طبقة)، وexplode يعيد أوّليات عالمية. */

const DEFS = new Map();
let seq = 1;

export function defineBlock(def) {
  if (!def || !def.name) throw new Error("block: name مطلوب");
  const d = { base: [0, 0], prims: [], ...def };
  if (!Array.isArray(d.prims)) d.prims = [];
  DEFS.set(String(d.name), d);
  return d;
}
export const getBlock = name => DEFS.get(name) || null;
export const hasBlock = name => DEFS.has(name);
export const blockList = () =>
  [...DEFS.values()].map(d => ({ name: d.name, title: d.title || d.name }));
export const removeBlock = name => DEFS.delete(name);

export function defineFromPrims(name, title, prims, base = [0, 0]) {
  const local = (prims || []).map(p => translate(p, -(base[0] || 0), -(base[1] || 0)));
  return defineBlock({ name, title, prims: local, base: [0, 0] });
}

export function makeInstance(name, opts = {}) {
  if (!DEFS.has(name)) throw new Error("block غير معرّف: " + name);
  return {
    id: opts.id || "b" + seq++,
    block: name,
    x: Number.isFinite(+opts.x) ? +opts.x : 0,
    y: Number.isFinite(+opts.y) ? +opts.y : 0,
    rot: Number.isFinite(+opts.rot) ? +opts.rot : 0,
    scale: Number.isFinite(+opts.scale) && +opts.scale > 0 ? +opts.scale : 1,
    mirror: !!opts.mirror,
    layer: opts.layer || "0"
  };
}

function xform(inst) {
  const c = Math.cos(inst.rot || 0), s = Math.sin(inst.rot || 0);
  const k = Number.isFinite(+inst.scale) && +inst.scale > 0 ? +inst.scale : 1;
  const mx = inst.mirror ? -1 : 1;
  return ([px, py]) => {
    const lx = px * k * mx, ly = py * k;
    return [inst.x + lx * c - ly * s, inst.y + lx * s + ly * c];
  };
}

function arcPts(c, r, a0, a1) {
  const span = a1 - a0;
  const n = Math.max(2, Math.ceil(Math.abs(span) / (Math.PI / 12)));
  const out = [];
  for (let i = 0; i <= n; i++) {
    const a = a0 + span * i / n;
    out.push([c[0] + r * Math.cos(a), c[1] + r * Math.sin(a)]);
  }
  return out;
}

export function explode(inst) {
  const def = DEFS.get(inst && inst.block);
  if (!def) return [];
  const T = xform(inst), out = [];
  for (const p of def.prims) {
    if (p.t === "line") {
      out.push({ t: "line", a: T(p.a), b: T(p.b), layer: inst.layer });
    } else if (p.t === "pline") {
      out.push({ t: "pline", pts: p.pts.map(T), closed: !!p.closed, layer: inst.layer });
    } else if (p.t === "circle") {
      out.push({ t: "pline", pts: arcPts(p.c, p.r, 0, Math.PI * 2).map(T), closed: true, layer: inst.layer });
    } else if (p.t === "arc") {
      out.push({ t: "pline", pts: arcPts(p.c, p.r, p.a0, p.a1).map(T), closed: false, layer: inst.layer });
    }
  }
  return out;
}

export function bbox(inst) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  for (const p of explode(inst)) {
    const pts = p.t === "line" ? [p.a, p.b] : p.pts;
    for (const [x, y] of pts) {
      minX = Math.min(minX, x); minY = Math.min(minY, y);
      maxX = Math.max(maxX, x); maxY = Math.max(maxY, y);
    }
  }
  if (!isFinite(minX)) return { minX: 0, minY: 0, maxX: 0, maxY: 0 };
  return { minX, minY, maxX, maxY };
}

export const toJSON = () => ({ defs: [...DEFS.values()] });
export function fromJSON(data) {
  if (!data || !Array.isArray(data.defs)) return;
  DEFS.clear();
  data.defs.forEach(d => defineBlock(d));
}

function translate(p, dx, dy) {
  const t = ([x, y]) => [x + dx, y + dy];
  if (p.t === "line") return { ...p, a: t(p.a), b: t(p.b) };
  if (p.t === "pline") return { ...p, pts: p.pts.map(t) };
  if (p.t === "circle" || p.t === "arc") return { ...p, c: t(p.c) };
  return p;
}

export function installDefaults() {
  [
    { name: "door", title: "باب مفرد", prims: [
      { t: "line", a: [0, 0], b: [0, 900] },
      { t: "arc", c: [0, 0], r: 900, a0: 0, a1: Math.PI / 2 }
    ]},
    { name: "window", title: "نافذة", prims: [
      { t: "line", a: [0, 0], b: [1200, 0] },
      { t: "line", a: [0, 200], b: [1200, 200] },
      { t: "line", a: [0, 100], b: [1200, 100] }
    ]},
    { name: "table", title: "طاولة", prims: [
      { t: "pline", pts: [[0, 0], [1200, 0], [1200, 700], [0, 700]], closed: true }
    ]}
  ].forEach(defineBlock);
}
```

<a id="f-js-core-boq-js"></a>

---

## `js/core/walls.js`

```javascript
/* ═══ الجدران ═══
   الجدار كائن صريح: مسار a→b وسماكة ومحاذاة.
   لا heal · لا لحم تلقائي · لا تقريب صامت — ما رسمته هو ما يُخزَّن.

   align: أي وجه يقع عليه المسار المرسوم
     c  المسار في المنتصف
     l  المسار على الوجه الأيسر  (الجسم يمتدّ يميناً)
     r  المسار على الوجه الأيمن  (الجسم يمتدّ يساراً)
   واليسار واليمين بالنسبة لاتجاه الرسم a→b. */
import {S,VER,touchGeom} from "./state.js";
import {newId,R2D,clamp,m2,m3} from "./units.js";
import {bandPoly,nearOnSeg,pip,dist,bboxOf,distSeg,cleanRing} from "./geom.js";

const R=v=>Math.round(v);
export const MINW=50;                 /* أقصر جدار مقبول */
export const TMIN=50, TMAX=1000;      /* حدود السماكة */

export const WTYPE={
 ext:{n:"خارجي",lay:"A-WALL"},
 int:{n:"داخلي",lay:"A-WALL"},
 low:{n:"سترة", lay:"A-WALL-LOW"}};
export const ALIGN={c:"مركزي",l:"الوجه الأيسر",r:"الوجه الأيمن"};
export const isWType=t=>!!WTYPE[t];
export const isLow=w=>!!(w&&w.type==="low");
export const lowH=w=>Math.max(200,Math.round(+(w&&w.h)||1000));

export function dir(w){
 if(!w||!w.a||!w.b)return null;
 const dx=w.b[0]-w.a[0], dy=w.b[1]-w.a[1], L=Math.hypot(dx,dy);
 if(L<1e-6)return null;
 const ux=dx/L, uy=dy/L;
 return {ux,uy,nx:-uy,ny:ux,L,ang:Math.atan2(uy,ux)*R2D};
}
const straightLen=w=>(w&&w.a&&w.b)
 ? Math.hypot(w.b[0]-w.a[0],w.b[1]-w.a[1]) : 0;
/* طول الجدار: الوتر للمستقيم، وطول القوس الفعلي للقوسي — BOQ
   والفاحص وأي مستهلكٍ آخر يريد الطول الحقيقي لا الوتر، فلا نُبقي
   الفرق صامتاً (يخالف عقد المشروع). */
export const wallLen=w=>{
 if(!isArc(w))return straightLen(w);
 const P=arcParams(w);
 return P?Math.abs(P.sweep)*P.R:straightLen(w);
};

/* ═══ الجدران القوسية ═══
   القوس يُمثَّل بحقلٍ اختياريّ bulge = tan(θ/4) بأسلوب DXF: θ زاوية
   القوس المحصورة من a إلى b، وإشارته تحدّد الجهة (موجب = عكس عقارب
   الساعة). bulge=0/غياب ⇒ جدارٌ مستقيمٌ بسلوكه القديم تماماً.
   والقوسُ يُقطَّع إلى مضلّعٍ في band() فيمرّ عبر الالتقاط والمناطق
   والصناديق دون أن تعرف بقيّةُ المحرّك أنه قوس. */
export const isArc=w=>!!(w&&isFinite(+w.bulge)&&Math.abs(+w.bulge)>1e-4);
export function arcParams(w){
 if(!isArc(w)||!w.a||!w.b)return null;
 const ax=w.a[0],ay=w.a[1],bx=w.b[0],by=w.b[1];
 const dx=bx-ax, dy=by-ay, L=Math.hypot(dx,dy);
 if(L<1e-6)return null;
 const bg=+w.bulge;
 const k=(1-bg*bg)/(2*bg);
 const cx=ax+dx/2-dy/2*k, cy=ay+dy/2+dx/2*k;
 const R=Math.hypot(ax-cx,ay-cy);
 const a0=Math.atan2(ay-cy,ax-cx);
 const sweep=4*Math.atan(bg);          /* موقَّع */
 const a1=a0+sweep;
 return {cx,cy,R,a0,a1,sweep,L,bulge:bg};
}
/* عدد القطع كافٍ لنعومةٍ لا تُرى زواياها في أي تكبير معقول */
const arcSegs=P=>{
 const R=P.R, sweep=Math.abs(P.sweep);
 const n=Math.ceil(sweep/(Math.PI/24));      /* ~7.5° لكل قطعة */
 return clamp(n, 6, 180);
};
/* نقاط على القوس المُزاح off عن مساره (off=0 المسار نفسه).
   الإزاحة نصفُ قطرٍ نحو المركز أو بعيداً عنه بحسب الإشارة. */
export function arcTess(w,off){
 const P=arcParams(w);
 if(!P)return null;
 const N=arcSegs(P), out=[];
 /* الجهة: القوس عكس/مع الساعة يحدّد أيّ إزاحةٍ تُقرِّب من المركز.
    نستعمل نصف قطرٍ فعّال: R - off*sign(sweep) لا يهم للبناء طالما
    الوجهان متماثلان في band. */
 const s=(P.sweep>=0)?1:-1;
 const Roff=P.R - (off||0)*s;
 for(let i=0;i<=N;i++){
  const a=P.a0+P.sweep*i/N;
  out.push([P.cx+Roff*Math.cos(a), P.cy+Roff*Math.sin(a)]);
 }
 return out;
}
export function arcLen(w){
 const P=arcParams(w);
 return P?Math.abs(P.sweep)*P.R:wallLen(w);
}
/* حساب bulge من ثلاث نقاط: بداية، نهاية، ونقطة على القوس */
export function bulgeFrom3(a,b,onArc){
 const ax=a[0],ay=a[1],bx=b[0],by=b[1],px=onArc[0],py=onArc[1];
 /* مركز الدائرة المارّة بالنقاط الثلاث */
 const d=2*(ax*(by-py)+bx*(py-ay)+px*(ay-by));
 if(Math.abs(d)<1e-6)return 0;               /* على استقامة */
 const ux=((ax*ax+ay*ay)*(by-py)+(bx*bx+by*by)*(py-ay)
   +(px*px+py*py)*(ay-by))/d;
 const uy=((ax*ax+ay*ay)*(px-bx)+(bx*bx+by*by)*(ax-px)
   +(px*px+py*py)*(bx-ax))/d;
 const cx=ux, cy=uy;
 const a0=Math.atan2(ay-cy,ax-cx);
 const a1=Math.atan2(by-cy,bx-cx);
 const ap=Math.atan2(py-cy,px-cx);
 /* الاتّجاه: هل النقطة على القوس بين a وb عكس الساعة؟ */
 const norm=x=>{while(x<=-Math.PI)x+=2*Math.PI;while(x>Math.PI)x-=2*Math.PI;return x};
 let sweepCCW=norm(a1-a0); if(sweepCCW<0)sweepCCW+=2*Math.PI;
 let toP=norm(ap-a0); if(toP<0)toP+=2*Math.PI;
 let sweep=(toP<=sweepCCW)?sweepCCW:(sweepCCW-2*Math.PI);
 const bg=Math.tan(sweep/4);
 return Math.abs(bg)<1e-4?0:clamp(bg,-8,8);
}

/* إزاحة محور الجسم عن المسار على العمود الأيسر n=(-uy,ux) */
export const alignOff=w=>{
 const t=(w&&w.t)||0;
 return (w.align==="l")?(-t/2):((w.align==="r")?(t/2):0);
};
/* الخطّ المركزي الفعلي — عليه تُقاس الفتحات */
export function centerLine(w){
 const d=dir(w);
 if(!d)return null;
 const o=alignOff(w);
 return {a:[w.a[0]+d.nx*o, w.a[1]+d.ny*o],
         b:[w.b[0]+d.nx*o, w.b[1]+d.ny*o]};
}
/* جسم الجدار مستطيلاً — بلا أي تعديل على البيانات.
   القوسي: شريطٌ بين قوسين مُزاحين ±t/2، مضلّعاً مغلقاً. */
export function band(w){
 if(isArc(w)){
  const outer=arcTess(w, w.t/2), inner=arcTess(w, -w.t/2);
  if(!outer||!inner)return null;
  const poly=outer.concat(inner.slice().reverse())
   .map(p=>[R(p[0]),R(p[1])]);
  return cleanRing(poly,0.5);
 }
 const c=centerLine(w);
 if(!c)return null;
 return bandPoly(c.a[0],c.a[1],c.b[0],c.b[1],w.t);
}
/* وجهَا الجدار قطعتين — لأدوات القياس والمرجع */
export function faces(w){
 const c=centerLine(w), d=dir(w);
 if(!c||!d)return null;
 const h=w.t/2;
 return {
  l:[[R(c.a[0]+d.nx*h),R(c.a[1]+d.ny*h)],
     [R(c.b[0]+d.nx*h),R(c.b[1]+d.ny*h)]],
  r:[[R(c.a[0]-d.nx*h),R(c.a[1]-d.ny*h)],
     [R(c.b[0]-d.nx*h),R(c.b[1]-d.ny*h)]]};
}
/* ═══ خريطة المعرّفات ═══
   على النسخة الهندسية: تحرّكُ بُعدٍ أو نصٍّ لا يبنيها من جديد. */
let MAP=null, MVER=-1;
export function wallById(id){
 if(MVER!==VER.g){
  MAP=new Map();
  S.walls.forEach(w=>MAP.set(w.id,w));
  MVER=VER.g;
 }
 return MAP.get(id)||null;
}
export function addWall(a,b,t,type,align,h,bulge){
 const A=[R(a[0]),R(a[1])], B=[R(b[0]),R(b[1])];
 if(Math.hypot(B[0]-A[0],B[1]-A[1])<MINW)
  throw new Error("الطول أقل من 5 سم");
 const ty=isWType(type)?type:"int";
 const df=(ty==="ext")?S.meta.tExt
  :((ty==="low")?S.meta.tLow:S.meta.tInt);
 const w={id:newId("W"),a:A,b:B,
  t:clamp(R(t||df),TMIN,TMAX), type:ty,
  align:ALIGN[align]?align:"c"};
 if(ty==="low")w.h=Math.max(200,R(h||S.meta.lowH));
 const bg=+bulge;
 if(isFinite(bg)&&Math.abs(bg)>1e-4)w.bulge=clamp(bg,-8,8);
 S.walls.push(w); touchGeom();
 return w;
}
export function delWall(w){
 const i=S.walls.indexOf(w);
 if(i<0)return false;
 S.walls.splice(i,1); touchGeom();
 return true;
}
/* إصابة: داخل الجسم أوّلاً، وإلا قرب المسار بتفاوت الشاشة.
   list مرشَّحو الفهرس — والغياب يعني المسح الكامل. */
export function wallAt(x,y,tol,list){
 let best=null,bd=1/0;
 (list||S.walls).forEach(w=>{
  const p=band(w);
  if(p&&pip(p,x,y)){
   const c=centerLine(w);
   const d=nearOnSeg(c.a,c.b,x,y).d;
   if(d<bd){bd=d;best=w}
   return;
  }
  const r=nearOnSeg(w.a,w.b,x,y);
  if(r.d<(tol||200)&&r.d<bd){bd=r.d;best=w}
 });
 return best;
}
export const wallsBBox=()=>{
 const P=[];
 S.walls.forEach(w=>{
  const p=band(w);
  if(p)p.forEach(q=>P.push(q));
  else{P.push(w.a);P.push(w.b)}
 });
 return bboxOf(P);
};
/* ═══ فهرس صناديق الأجسام ═══
   شبكةٌ بخلايا مترين — كشبكة الأطراف وشبكة المراسي، وبعقدها:
   تُبنى مرّةً لكل نسخةٍ هندسية.

   تخدم بصمة المناطق: كانت تمسح S.walls كلَّها وتبني band لكلٍّ،
   لكل منطقةٍ في كل إطار — خمسون منطقةً وثلاث مئة جدارٍ = ١٥٠٠٠
   بناء band. وموضعها هنا لا في core/sindex لأن areas → sindex
   → entreg → areas دورةٌ حقيقية: جسم entreg يُنفَّذ أوّلاً فيقع
   areaById في نطاق التصريح المؤقّت. والمشروع يتجنّبها سلفاً
   بتمرير المرشَّحين وسيطاً (colOnWall · fixOnWall). */
const BCELL=2000;
let BG=null, BGV=-1;
function bandGrid(){
 if(BGV===VER.g&&BG)return BG;
 const g=new Map(), big=[];
 const put=(k,i)=>{
  let a=g.get(k);
  if(!a){a=[]; g.set(k,a)}
  a.push(i);
 };
 S.walls.forEach((w,i)=>{
  const b=bboxOf(band(w)||[w.a,w.b]);
  if(!b){big.push(i); return}
  const x0=Math.floor(b.x0/BCELL), x1=Math.floor(b.x1/BCELL);
  const y0=Math.floor(b.y0/BCELL), y1=Math.floor(b.y1/BCELL);
  /* الجدار الممتدّ لا يُحشَر في مئة خليّة: يُفحَص دائماً */
  if((x1-x0+1)*(y1-y0+1)>64){big.push(i); return}
  for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++)
   put(cx+","+cy,i);
 });
 BG={g,big}; BGV=VER.g;
 return BG;
}
/* الجدران التي قد يلمس جسمها الصندوق — مرشَّحون لا قرار.
   والترتيب بترتيب S.walls فلا يتبدّل جوابٌ يعتمد عليه، ولا
   تتبدّل بصمةُ منطقةٍ محفوظة. */
export function wallsIn(box){
 if(!box)return S.walls.slice();
 const {g,big}=bandGrid();
 const x0=Math.floor(box.x0/BCELL), x1=Math.floor(box.x1/BCELL);
 const y0=Math.floor(box.y0/BCELL), y1=Math.floor(box.y1/BCELL);
 if((x1-x0+1)*(y1-y0+1)>4096)return S.walls.slice();
 const seen=new Set(big);
 for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++){
  const a=g.get(cx+","+cy);
  if(a)a.forEach(i=>seen.add(i));
 }
 return [...seen].sort((a,b)=>a-b).map(i=>S.walls[i]);
}
export const bandGridStats=()=>{
 const {g,big}=bandGrid();
 return {cells:g.size,big:big.length,ver:BGV};
};
/* ═══ الأطراف غير المتّصلة — معلومة عرض لا تعديل ═══
   الطرف حرّ إن لم يلامس مسار جدار آخر بتفاوت مذكور.
   تُرسَم عليه علامة، ولا يُلحَم إلا بأمرك على تحديد صريح.

   كاش على النسخة الهندسية وعلى تفاوت الاستدعاء معاً — يُستدعى مع
   كل حركة مؤشّر عبر drawEnds. وكان على النسخة العامّة، فيُعاد
   بناؤه مع كل إطارٍ أثناء سحب أي شيء. */
const ECELL=2000;
function segGrid(){
 const g=new Map();
 const put=(k,i)=>{
  let a=g.get(k);
  if(!a){a=[]; g.set(k,a)}
  a.push(i);
 };
 S.walls.forEach((w,i)=>{
  const x0=Math.floor(Math.min(w.a[0],w.b[0])/ECELL);
  const x1=Math.floor(Math.max(w.a[0],w.b[0])/ECELL);
  const y0=Math.floor(Math.min(w.a[1],w.b[1])/ECELL);
  const y1=Math.floor(Math.max(w.a[1],w.b[1])/ECELL);
  /* جدارٌ ممتدّ جدّاً يُفحَص دائماً بدل أن يُحشَر في مئة خليّة */
  if((x1-x0+1)*(y1-y0+1)>64){put("*",i); return}
  for(let cx=x0;cx<=x1;cx++)for(let cy=y0;cy<=y1;cy++)
   put(cx+","+cy,i);
 });
 return g;
}
let LCACHE=null, LVER=-1, LTOL=null;
export function looseEnds(tol){
 const T=Math.max(1,tol==null?2:tol);
 if(LVER===VER.g&&LTOL===T&&LCACHE)return LCACHE;
 const G=segGrid();
 const r=Math.ceil(T/ECELL);
 const test=(list,p,skip)=>{
  if(!list)return false;
  for(const j of list){
   if(j===skip)continue;
   const w=S.walls[j];
   if(distSeg(w.a,w.b,p[0],p[1])<=T)return true;
  }
  return false;
 };
 const free=(p,skip)=>{
  const cx=Math.floor(p[0]/ECELL), cy=Math.floor(p[1]/ECELL);
  for(let i=-r;i<=r;i++)for(let j=-r;j<=r;j++)
   if(test(G.get((cx+i)+","+(cy+j)),p,skip))return false;
  return !test(G.get("*"),p,skip);
 };
 const out=[];
 S.walls.forEach((w,i)=>{
  [["a",w.a],["b",w.b]].forEach(([k,p])=>{
   if(!free(p,i))return;
   out.push({id:w.id,end:k,p:p.slice(),w});
  });
 });
 LCACHE=out; LVER=VER.g; LTOL=T;
 return LCACHE;
}
```

---

# الأدوات (`js/tools/`)

<a id="f-js-tools-annotate-js"></a>

---

## `js/core/opens.js`

```javascript
/* ═══ الفتحات ═══
   الفتحة كائن صريح على جدار: s المسافة من بداية مساره إلى مركزها.
   لا تُقطع الجدار في البيانات — الطرح يقع في render.js وقت العرض،
   فحذفها يعيد الجدار كاملاً بلا أثر.

   الأنواع: sub=1 تقطع الجسم · part=1 تُرقّقه ولا تقطعه
            sw=1 تقبل قلب جهة الفتح · pan=1 تقبل عدد مصاريع */
import {S,VER,touchOpen} from "./state.js";
import {newId,clamp,m2,m3,rng2,dm2} from "./units.js";
import {dir,centerLine,wallById,wallLen,alignOff,isArc} from "./walls.js";

const R=v=>Math.round(v);
export const OK={
 door:   {n:"باب مفرد",   lay:"A-DOOR", sub:1, sw:1},
 double: {n:"باب مزدوج",  lay:"A-DOOR", sub:1, sw:1},
 sliding:{n:"باب سحب",    lay:"A-DOOR", sub:1},
 window: {n:"شباك",       lay:"A-GLAZ", sub:1, pan:1},
 fixed:  {n:"شباك ثابت",  lay:"A-GLAZ", sub:1, pan:1},
 opening:{n:"فتحة صافية", lay:"A-GLAZ", sub:1},
 arch:   {n:"فتحة مقنطرة",lay:"A-GLAZ", sub:1},
 niche:  {n:"كوّة",        lay:"A-GLAZ", part:1}
};
export const OKINDS=Object.keys(OK);
export const okOf=k=>OK[k]||OK.door;
export const okName=k=>okOf(k).n;
export const isPart=o=>!!okOf(o&&o.kind).part;
export const panOf=o=>clamp(R(+(o&&o.pan)||1),1,6);
export const depOf=(o,t)=>clamp(
 R(+(o&&o.dep)||R(t*0.45)),20,Math.max(20,t-40));

export const MINW=100;          /* أضيق فتحة مقبولة */
export const EDGE=50;           /* أقلّ ما يبقى من الجدار على كل جانب */

export const opensOf=id=>S.opens.filter(o=>o.wall===id);
export const openById=id=>S.opens.find(o=>o.id===id)||null;
export const span=o=>[o.s-o.w/2, o.s+o.w/2];

/* ═══ حالة الفتحة — تقرير لا إصلاح ═══
   ok سليمة · over تخرج عن مدى جدارها · clash تتراكب مع أخرى
   تُرسَم الحالتان الأخيرتان بالأحمر وتُذكران في لوحة الحالة. */
export function openState(o){
 const w=wallById(o.wall);
 if(!w)return "orphan";
 const L=wallLen(w);
 const [a,b]=span(o);
 if(a<-1||b>L+1)return "over";
 for(const x of opensOf(o.wall)){
  if(x===o)continue;
  const [c,d]=span(x);
  if(a<d-1&&c<b-1)return "clash";
 }
 return "ok";
}
/* ═══ المعطوبة ═══
   تُمسَح في كل مشهد — أي في كل إطارٍ أثناء سحب أي شيء. وopenState
   صار أثقل بعد الدفعة ٤ (يقيس على الفترات الحرّة لا على الغلاف)،
   فمئةُ فتحةٍ تعني مئةَ مسحٍ للفتحات المجاورة ستّين مرّةً في
   الثانية.
   والمفتاح نسختا الهندسة والفتحات: تحرّكُ بُعدٍ لا يُعطِب فتحة. */
let BO=null, BOK="";
export function badOpens(){
 const k=`${VER.g}|${VER.o}`;
 if(BO&&BOK===k)return BO;
 const out=[];
 S.opens.forEach(o=>{if(openState(o)!=="ok")out.push(o)});
 BO=out; BOK=k;
 return BO;
}
export const badStats=()=>({key:BOK,n:BO?BO.length:-1});

/* ═══ الفترات الحرّة لمركز فتحةٍ بعرض W ═══
   الجدار يُقصّ عند كل فتحةٍ قائمة، فيبقى مدىً متقطّع. والدالّة
   القديمة (allowed) كانت تعيد غلافاً واحداً متّصلاً — يَعِد بموضعٍ
   مشغول، والمُثبِّت يرفضه بسببٍ لا يذكره الوعد.
   والنتيجة مرتَّبةٌ بالموضع لا بترتيب S.opens، فلا يتوقّف الجواب
   على ترتيب المصفوفة. */
export function freeSpans(w,W,skip){
 const L=wallLen(w);
 const lo=W/2+EDGE, hi=L-W/2-EDGE;
 if(hi<lo)return {L,spans:[],fits:false};
 /* كل فتحةٍ تمنع مركزاً يقع في [a−W/2 , b+W/2] */
 const block=[];
 opensOf(w.id).forEach(o=>{
  if(o===skip)return;
  const [a,b]=span(o);
  block.push([a-W/2, b+W/2]);
 });
 block.sort((p,q)=>p[0]-q[0]);
 const out=[];
 let s=lo;
 block.forEach(([a,b])=>{
  if(b<=s)return;                    /* خلف موضعنا */
  if(a>s+1)out.push([R(s),R(Math.min(a,hi))]);
  s=Math.max(s,b);
 });
 if(s<hi-1)out.push([R(s),R(hi)]);
 const spans=out.filter(([a,b])=>b-a>=1);
 return {L,spans,fits:spans.length>0};
}
/* أقرب موضعٍ حرٍّ إلى s — للسحب: يتوقّف عند الحدّ ولا يرفض.
   وعند تساوي المسافتين يبقى في فترته: السحب لا يقفز فوق فتحةٍ
   قائمة إلى الجهة الأخرى منها. */
export function nearestFree(w,W,s,skip){
 const F=freeSpans(w,W,skip);
 if(!F.fits)return null;
 const from=(skip&&isFinite(+skip.s))?+skip.s:s;
 let best=null, bd=1/0, bf=1/0;
 F.spans.forEach(([a,b])=>{
  const q=clamp(s,a,b);
  const d=Math.abs(q-s), f=Math.abs(q-from);
  if(d<bd-0.5||(Math.abs(d-bd)<=0.5&&f<bf)){bd=d; bf=f; best=q}
 });
 return best==null?null:R(best);
}
/* الغلاف المتّصل — للتوافق ولعرض الحدَّين الأقصيَين.
   spans فيه التفصيل، وfits يعني «يوجد موضعٌ حرّ» لا «المدى متّصل». */
export function allowed(w,W,skip){
 const F=freeSpans(w,W,skip);
 if(!F.fits)
  return {lo:0,hi:0,L:F.L,fits:false,spans:[],split:false};
 return {lo:F.spans[0][0], hi:F.spans[F.spans.length-1][1],
  L:F.L, fits:true, spans:F.spans, split:F.spans.length>1};
}
/* نصٌّ للرسائل: يذكر الفترات كلّها لا غلافها */
export const saySpans=F=>((F&&F.spans)||[])
 .map(([a,b])=>rng2(a,b)).join(" أو ")||"لا موضع";

/* ═══ جدول الفتحات ═══
   تقريرٌ يُجمَع عند العرض: لا حقل يُخزَّن على الفتحة. والرمز مشتقٌّ
   من ترتيب الجدول نفسه، فلا يُكتَب ولا يُصدَّر — كجدول المساحات،
   إلّا أن المناطق تُسمّى بيدك والفتحات تُجمَع بمقاسها. */
const MK={door:"ب",double:"ب",sliding:"ب",niche:"ك"};
export function openSchedule(){
 const G=new Map();
 S.opens.forEach(o=>{
  const pan=okOf(o.kind).pan?panOf(o):1;
  const dep=(o.kind==="niche")?(o.dep||0):0;
  const k=`${o.kind}|${o.w}|${o.h}|${o.sill||0}|${pan}|${dep}`;
  let r=G.get(k);
  if(!r){
   r={kind:o.kind,w:o.w,h:o.h,sill:o.sill||0,pan,dep,n:0,ids:[]};
   G.set(k,r);
  }
  r.n++; r.ids.push(o.id);
 });
 const rows=[...G.values()];
 rows.sort((a,b)=>(MK[a.kind]||"ش").localeCompare(MK[b.kind]||"ش","ar")
  ||(b.w-a.w)||(b.h-a.h));
 const c={};
 rows.forEach(r=>{
  const p=MK[r.kind]||"ش";
  c[p]=(c[p]||0)+1;
  r.mark=p+c[p];
 });
 return {rows, total:S.opens.length,
  bad:S.opens.filter(o=>openState(o)!=="ok").length};
}
export function addOpen(w,s,kind,W,H,sill,ex){
 if(!w)throw new Error("لا جدار مستهدف");
 if(isArc(w))throw new Error("الفتحات على الجدران المستقيمة فقط "
  +"— الجدار قوسيّ");
 const K=OK[kind]?kind:"door";
 const L=wallLen(w);
 W=Math.max(MINW,R(W||900));
 H=Math.max(MINW,R(H||2100));
 sill=Math.max(0,R(sill||0));
 s=R(s);
 if(W+EDGE*2>L)
  throw new Error(`العرض ${m2(W)} م لا يتّسع في ${w.id} `
   +`(طوله ${m2(L)} م · الأقصى ${m2(L-EDGE*2)} م)`);
 /* الفترات الحرّة لا الغلاف: الرسالة تذكر ما يُقبَل فعلاً */
 const A=allowed(w,W,null);
 if(!A.fits)
  throw new Error(`لا موضع حرٌّ بعرض ${m2(W)} م على ${w.id} — `
   +`الفتحات القائمة تشغل مداه`);
 if(s-W/2<-1||s+W/2>L+1)
  throw new Error(`الموضع ${m2(s)} م يخرج عن ${w.id} — `
   +`المواضع الحرّة ${saySpans(A)} م`);
 for(const o of opensOf(w.id)){
  const [a,b]=span(o);
  if(s-W/2<b&&a<s+W/2)
   throw new Error(`تتراكب مع ${o.id} (${rng2(a,b,"م")}) — `
    +`المواضع الحرّة ${saySpans(A)} م`);
 }
 const o={id:newId("O"),wall:w.id,kind:K,s,w:W,h:H,sill,
  hinge:"start",swing:"left"};
 if(ex){
  if(ex.hinge==="end")o.hinge="end";
  if(ex.swing==="right")o.swing="right";
  if(ex.pan!=null)o.pan=clamp(R(ex.pan),1,6);
  const dp=(ex.dep!=null)?ex.dep:ex.d;
  if(dp!=null){
   /* القيد عند المنفذ لا عند التعديل وحده: كوّةٌ أعمق من جدارها
      كانت تُنشأ بلا اعتراض وتُرسَم صحيحةً (depOf يقصّها عند
      العرض) ثم تمنع تعديل سماكة جدارها إلى الأبد. */
   const mx=Math.max(20,w.t-40);
   const d=R(dp);
   if(K==="niche"&&d>mx)
    throw new Error(`عمق الكوّة ${m3(d)} م لا يكفيه جدارٌ سماكته `
     +`${m3(w.t)} م — الأقصى ${m3(mx)} م`);
   o.dep=clamp(d,20,mx);
  }
  if(ex.face)o.face=(ex.face==="r")?"r":"l";
 }
 S.opens.push(o); touchOpen();
 return o;
}
export function delOpen(o){
 const i=S.opens.indexOf(o);
 if(i<0)return false;
 S.opens.splice(i,1); touchOpen();
 return true;
}
/* الإسقاط على مسار الجدار — لحساب s من نقرة */
export function sAt(w,p){
 const d=dir(w);
 if(!d)return 0;
 return R((p[0]-w.a[0])*d.ux+(p[1]-w.a[1])*d.uy);
}
/* موضع مركز الفتحة على محور الجسم — للمقابض والرموز */
export function openPt(w,s){
 const d=dir(w);
 if(!d)return [w.a[0],w.a[1]];
 const o=alignOff(w);
 return [R(w.a[0]+d.ux*s+d.nx*o), R(w.a[1]+d.uy*s+d.ny*o)];
}
/* ═══ رموز الفتحات ═══
   تُبنى بالإحداثيات العالمية مباشرة: لا بلوكات في هذه المرحلة —
   المصدِّر يقرأ الأوّليات نفسها. */
const LN=(L,a,b,x)=>Object.assign({t:"line",L,
 a:[R(a[0]),R(a[1])], b:[R(b[0]),R(b[1])]},x||{});
const PL=(L,pts,cl)=>({t:"poly",L,
 pts:pts.map(p=>[R(p[0]),R(p[1])]),cl:cl===0?0:1});
const AC=(L,c,r,a0,a1)=>({t:"arc",L,cx:R(c[0]),cy:R(c[1]),
 r:Math.max(1,R(r)),a0,a1});
const ang=(a,b)=>Math.atan2(b[1]-a[1],b[0]-a[0])*180/Math.PI;

export function openPrims(o){
 const w=wallById(o.wall);
 if(!w)return [];
 const d=dir(w);
 if(!d)return [];
 const K=okOf(o.kind), L=K.lay;
 const off=alignOff(w), t=w.t;
 /* الإطار المحلّي: P(u,v) — u على طول الجدار من حدّ الفتحة الأول،
    v عبر السماكة حول محور الجسم (−t/2 … +t/2) */
 const [s0]=span(o);
 const P=(u,v)=>[w.a[0]+d.ux*(s0+u)+d.nx*(off+v),
                 w.a[1]+d.uy*(s0+u)+d.ny*(off+v)];
 const W=o.w, out=[];

 /* الكوّة تظهر من ترقيق الجسم نفسه — لا رمز لها */
 if(K.part)return out;

 if(o.kind==="door"||o.kind==="double"){
  const leaf=(hs,sg,side)=>{
   /* hs موضع المفصّلة على u · sg اتجاه الورقة · side جهة الفتح */
   const R0=W*0.98, lt=Math.max(20,W*0.05);
   const H=P(hs,0);
   const tip=P(hs, side*R0);
   const far=P(hs+sg*R0, 0);
   out.push(PL(L,[H, tip, P(hs-sg*lt, side*R0), P(hs-sg*lt,0)],1));
   const a1=ang(H,tip), a2=ang(H,far);
   out.push((side*sg>0)?AC(L,H,R0,a2,a1):AC(L,H,R0,a1,a2));
  };
  const side=(o.swing==="left")?1:-1;
  if(o.kind==="door"){
   const sg=(o.hinge==="end")?-1:1;
   leaf((sg>0)?0:W, sg, side);
  }else{
   leaf(0, 1, side);
   leaf(W,-1, side);
  }
  return out;
 }
 if(o.kind==="sliding"){
  const p=t*0.30;
  out.push(LN(L,P(0,-t/2),P(W,-t/2)));
  out.push(LN(L,P(0, t/2),P(W, t/2)));
  out.push(PL(L,[P(W*0.02, p*0.2),P(W*0.54, p*0.2),
                 P(W*0.54, p*1.0),P(W*0.02, p*1.0)],1));
  out.push(PL(L,[P(W*0.46,-p*1.0),P(W*0.98,-p*1.0),
                 P(W*0.98,-p*0.2),P(W*0.46,-p*0.2)],1));
  out.push(LN(L,P(0,0),P(W,0)));            /* السكّة */
  return out;
 }
 if(o.kind==="window"||o.kind==="fixed"){
  const n=panOf(o), g=(o.kind==="fixed")?0.14:0.20;
  out.push(LN(L,P(0,-t/2),P(W,-t/2)));
  out.push(LN(L,P(0, t/2),P(W, t/2)));
  out.push(LN(L,P(0,-t*g),P(W,-t*g)));
  out.push(LN(L,P(0, t*g),P(W, t*g)));
  for(let i=1;i<n;i++){
   const u=W*i/n, k=Math.max(10,W*0.012);
   out.push(PL(L,[P(u-k,-t/2),P(u+k,-t/2),
                  P(u+k, t/2),P(u-k, t/2)],1));
  }
  return out;
 }
 if(o.kind==="arch"){
  out.push(LN(L,P(0,-t/2),P(0,t/2)));
  out.push(LN(L,P(W,-t/2),P(W,t/2)));
  out.push(LN(L,P(W*0.05,0),P(W*0.95,0),
   {dash:[Math.max(20,t*0.5),Math.max(16,t*0.4)]}));
  return out;
 }
 /* فتحة صافية: عضادتان فقط */
 out.push(LN(L,P(0,-t/2),P(0,t/2)));
 out.push(LN(L,P(W,-t/2),P(W,t/2)));
 return out;
}
/* علامة تحذير على الفتحة المعطوبة — تُرسَم فوق كل شيء وتُرى دائماً */
export function badPrims(o){
 const w=wallById(o.wall);
 if(!w)return [];
 const c=openPt(w,o.s);
 const r=Math.max(180,w.t*0.8);
 return [
  {t:"arc",L:"__BAD",cx:c[0],cy:c[1],r:R(r),a0:0,a1:359.9,bad:1},
  {t:"line",L:"__BAD",bad:1,
   a:[R(c[0]-r*0.7),R(c[1]-r*0.7)], b:[R(c[0]+r*0.7),R(c[1]+r*0.7)]}];
}
export const openLabel=o=>`${okName(o.kind)} ${dm2(o.w,o.h,"م")}`
 +(o.sill?` · جلسة ${m2(o.sill)} م`:"");
```

<a id="f-js-core-osnap-js"></a>

---

## `js/core/cols.js`

```javascript
/* ═══ الأعمدة ═══
   العمود كائن مستقلّ بمركز ومقاس ودوران. يُدمَج في الجسم المصمَّت
   وقت العرض — لا في البيانات — فحذفه يعيد الجدار كما كان.

   الدائرة تُقرَّب 32 ضلعاً في الاتحاد لأن polyBool تعمل على
   مضلّعات؛ وعند العرض المستقلّ تُرسَم قوساً حقيقياً وتُصدَّر CIRCLE. */
import {S,touchGeom,txtH} from "./state.js";
import {newId,clamp,D2R,deg,m2,m3,sqm,ltr,dm2,pt2} from "./units.js";
import {pip,pArea,bboxOf,bboxHit,nearOnSeg,convexHit} from "./geom.js";
import {band} from "./walls.js";

const R=v=>Math.round(v);
const NSEG=32;

export const CK={rect:"مستطيل",circ:"دائري"};
export const CT={conc:"خرسانة",steel:"حديد",stone:"حجر"};
export const CMIN=100, CMAX=4000;

export const colById=id=>S.cols.find(c=>c.id===id)||null;
export const colW=c=>Math.max(CMIN,(c&&c.w)||CMIN);
export const colH=c=>(c&&c.kind==="circ")
 ? colW(c) : Math.max(CMIN,(c&&c.h)||CMIN);
export const colArea=c=>(c.kind==="circ")
 ? Math.PI*(colW(c)/2)*(colW(c)/2) : colW(c)*colH(c);

/* ═══ المضلّع العالميّ ═══ */
export function colPoly(c){
 if(!c)return null;
 if(c.kind==="circ"){
  const r=colW(c)/2, out=[];
  for(let i=0;i<NSEG;i++){
   const a=i/NSEG*Math.PI*2;
   out.push([R(c.x+r*Math.cos(a)), R(c.y+r*Math.sin(a))]);
  }
  return out;
 }
 const a=(c.rot||0)*D2R, ca=Math.cos(a), sa=Math.sin(a);
 const hw=colW(c)/2, hh=colH(c)/2;
 return [[-hw,-hh],[hw,-hh],[hw,hh],[-hw,hh]]
  .map(p=>[R(c.x+p[0]*ca-p[1]*sa), R(c.y+p[0]*sa+p[1]*ca)]);
}
export const colBBox=c=>bboxOf(colPoly(c));
const nearRing=(ring,x,y)=>{
 let d=1/0;
 for(let i=0;i<ring.length;i++){
  const r=nearOnSeg(ring[i],ring[(i+1)%ring.length],x,y);
  if(r.d<d)d=r.d;
 }
 return d;
};
export const colAt=(x,y,tol)=>{
 const T=tol||0;
 let best=null, ba=1/0;
 S.cols.forEach(c=>{
  const p=colPoly(c);
  if(!p)return;
  if(!(pip(p,x,y)||(T>0&&nearRing(p,x,y)<=T)))return;
  const ar=colArea(c);
  if(ar<ba){ba=ar;best=c}
 });
 return best;
};
/* ═══ الإنشاء ═══ */
export function addCol(kind,p,w,h,rot,type,tag){
 const K=CK[kind]?kind:"rect";
 const W=clamp(R(w||300),CMIN,CMAX);
 const H=(K==="circ")?W:clamp(R(h||W),CMIN,CMAX);
 const c={id:newId("K"),kind:K,
  x:R(p[0]),y:R(p[1]),w:W,h:H,
  rot:(K==="circ")?0:deg(+rot||0),
  type:CT[type]?type:"conc"};
 /* التطابق التامّ في المركز لا معنى له — التراكب يُبلَّغ ولا يُرفَض */
 const dup=S.cols.find(o=>Math.abs(o.x-c.x)<20&&Math.abs(o.y-c.y)<20);
 if(dup)throw new Error(`${dup.id} على المركز نفسه `
  +`${pt2([c.x,c.y])} — أزِحه أو عدّل مقاسه`);
 if(tag)c.tag=String(tag).slice(0,10);
 S.cols.push(c); touchGeom();
 return c;
}
export function delCol(c){
 const i=S.cols.indexOf(c);
 if(i<0)return false;
 S.cols.splice(i,1); touchGeom();
 return true;
}
export const nextTag=pre=>{
 const P=String(pre||"C").replace(/\d+$/,"").slice(0,6)||"C";
 const re=new RegExp("^"+P.replace(/[.*+?^${}()|[\]\\]/g,"\\$&")
  +"(\\d+)$");
 let n=0;
 S.cols.forEach(c=>{
  const m=re.exec(c.tag||"");
  if(m)n=Math.max(n,parseInt(m[1],10));
 });
 return P+(n+1);
};
/* ═══ العمود على جدار؟ ═══ تقريرٌ للفاحص لا رابطة تُحفَظ ═══
   walls مرشَّحو الفهرس — والغياب يعني المسح الكامل. */
export function colOnWall(c,tol,walls){
 const T=(tol==null)?2:tol;      /* والصفرُ صفرٌ — لا افتراضٌ يمحوه */
 const b=colBBox(c);
 if(!b)return null;
 const cp=colPoly(c);
 for(const w of (walls||S.walls)){
  const bp=band(w);
  if(!bp)continue;
  const wb=bboxOf(bp);
  if(!wb||!bboxHit(wb,b,T))continue;
  /* بالأضلاع لا بالرؤوس: عمودٌ مركزُه على محور جدارٍ أنحفَ منه
     لا يُدخِل رأساً في الآخر، ومع ذلك يعبره. */
  if(convexHit(cp,bp,T))return w.id;
 }
 return null;
}
export function colsOverlap(a,b){
 const A=colPoly(a), B=colPoly(b);
 if(!A||!B)return false;
 if(!bboxHit(bboxOf(A),bboxOf(B),-1))return false;
 return convexHit(A,B,-1);       /* التلاصقُ وجهاً بوجهٍ ليس تراكباً */
}
/* ═══ الأوّليات ═══
   المدمَج لا حدّ خاصّ له: حدّه من الاتحاد نفسه، ويبقى الوسم
   وصليب المركز. المستقلّ يُرسَم محيطاً وهاشوراً. */
export function colPrims(c,solo){
 const L="A-COLS", out=[], h=txtH();
 if(solo){
  if(c.kind==="circ")
   out.push({t:"arc",L,cx:c.x,cy:c.y,r:R(colW(c)/2),
    a0:0,a1:359.9,kid:c.id});
  else
   out.push({t:"poly",L,pts:colPoly(c),cl:1,kid:c.id});
  out.push({t:"hatch",L,loops:[colPoly(c)],
   pat:(c.type==="steel")?"ANSI31":"SOLID",
   sc:h*(c.type==="steel"?2.2:1.0), kid:c.id});
 }
 /* صليب المركز — علامة خفيفة تدلّ على مركز الشبكة */
 const s=Math.max(60,Math.min(colW(c),colH(c))*0.18);
 out.push({t:"line",L,a:[R(c.x-s),c.y],b:[R(c.x+s),c.y],kid:c.id});
 out.push({t:"line",L,a:[c.x,R(c.y-s)],b:[c.x,R(c.y+s)],kid:c.id});
 if(c.tag)
  out.push({t:"text",L,s:c.tag,x:c.x,
   y:R(c.y+Math.max(colH(c),colW(c))/2+h*0.35),
   h:h*0.9,al:"bc",kid:c.id});
 return out;
}
export const colLabel=c=>(c.kind==="circ")
 ? ltr(`⌀${m2(colW(c))}`)+" م" : dm2(colW(c),colH(c),"م");
export const colName=c=>`${CK[c.kind]||"عمود"} ${colLabel(c)}`
 +` · ${CT[c.type]||""}`;
```

<a id="f-js-core-coords-js"></a>

---

## `js/core/fixt.js`

```javascript
/* ═══ الأدوات الصحية والمطبخية ═══
   رموزٌ فوق الجسم لا جزءٌ منه: لا تدخل الاتحاد ولا تقطع جداراً ولا
   تُطرَح منه — لأنها أثاثٌ لا بناء.

   الإطار المحلّي: الأصل ظهر الأداة (ما يلاصق الجدار)، u على عرضها
   و v إلى الأمام. فوضعها على جدار يعني ضبط دورانها وحده.
   والإلصاق أمرٌ يُنفَّذ عند الوضع، لا رابطةٌ تُحفَظ. */
import {S,touchView} from "./state.js";
import {newId,clamp,D2R,deg,m2,dm2} from "./units.js";
import {pip,bboxOf,bboxHit,nearOnSeg,distSeg,segSeg,
        convexHit} from "./geom.js";
import {band} from "./walls.js";

const R=v=>Math.round(v);
export const FK={
 wc:    {n:"كرسي إفرنجي", w:400,  d:700},
 bidet: {n:"شطّاف",        w:380,  d:600},
 ur:    {n:"مبولة",        w:380,  d:350},
 lav:   {n:"مغسلة",        w:550,  d:450},
 sink:  {n:"حوض مطبخ",     w:800,  d:500},
 shower:{n:"دُش",           w:900,  d:900},
 tub:   {n:"بانيو",        w:1700, d:750},
 wm:    {n:"غسّالة",        w:600,  d:600},
 fd:    {n:"صفاية أرضية",  w:150,  d:150}
};
export const FKINDS=Object.keys(FK);
export const fkOf=k=>FK[k]||FK.wc;
export const fixById=id=>S.fixt.find(f=>f.id===id)||null;
export const fixW=f=>Math.max(80,+((f&&f.w))||fkOf(f&&f.kind).w);
export const fixD=f=>Math.max(80,+((f&&f.d))||fkOf(f&&f.kind).d);
export const fixName=f=>fkOf(f&&f.kind).n;

/* الإطار: P(u,v) — u على العرض من المركز، v من الظهر إلى الأمام */
export function frameOf(f){
 const a=(f.rot||0)*D2R, ca=Math.cos(a), sa=Math.sin(a);
 const m=(f.mir?-1:1);
 return (u,v)=>[R(f.x+(u*m)*ca-v*sa), R(f.y+(u*m)*sa+v*ca)];
}
export function fixPoly(f){
 const P=frameOf(f), w=fixW(f)/2, d=fixD(f);
 return [P(-w,0),P(w,0),P(w,d),P(-w,d)];
}
export const fixBBox=f=>bboxOf(fixPoly(f));
export const fixCenter=f=>{
 const P=frameOf(f);
 return P(0,fixD(f)/2);
};
export const fixAt=(x,y)=>{
 let best=null,ba=1/0;
 S.fixt.forEach(f=>{
  if(!pip(fixPoly(f),x,y))return;
  const a=fixW(f)*fixD(f);
  if(a<ba){ba=a;best=f}
 });
 return best;
};
export function addFix(kind,p,rot,ex){
 const K=FK[kind]?kind:"wc";
 const d=fkOf(K);
 const f={id:newId("F"),kind:K,x:R(p[0]),y:R(p[1]),
  rot:deg(+rot||0), w:d.w, d:d.d};
 if(ex){
  if(ex.w)f.w=clamp(R(ex.w),80,4000);
  if(ex.d)f.d=clamp(R(ex.d),80,4000);
  if(ex.mir)f.mir=1;
 }
 S.fixt.push(f); touchView();
 return f;
}
export function delFix(f){
 const i=S.fixt.indexOf(f);
 if(i<0)return false;
 S.fixt.splice(i,1); touchView();
 return true;
}
/* ═══ الإسناد إلى جدار ═══
   يبحث عن أقرب وجهِ جدارٍ ويعيد الموضع والدوران — أمرٌ يُنفَّذ عند
   الوضع، لا رابطةٌ تُحفَظ. الأداة بعده إحداثيات صريحة. */
export function snapToWall(p,tol){
 const T=Math.max(50,tol||1200);
 let best=null, bd=T;
 S.walls.forEach(w=>{
  const bp=band(w);
  if(!bp)return;
  for(let i=0;i<bp.length;i++){
   const A=bp[i], B=bp[(i+1)%bp.length];
   const r=nearOnSeg(A,B,p[0],p[1]);
   if(r.d>=bd)continue;
   const dx=B[0]-A[0], dy=B[1]-A[1], L=Math.hypot(dx,dy);
   if(L<1)continue;
   /* العمود الداخل إلى الفراغ: من الوجه نحو النقطة */
   let nx=-dy/L, ny=dx/L;
   if((p[0]-r.p[0])*nx+(p[1]-r.p[1])*ny<0){nx=-nx;ny=-ny}
   bd=r.d;
   best={p:[R(r.p[0]),R(r.p[1])],
    rot:deg(Math.atan2(ny,nx)*180/Math.PI-90),
    wall:w.id, d:R(r.d)};
  }
 });
 return best;
}
/* المسافةُ بين قطعتين — الظهرُ مع وجه الجدار. محلّيّةٌ لا مُصدَّرة:
   عقدُ هذا الملفّ لا عقدُ الهندسة. */
const segGap=(p,q,a,b)=>segSeg(p,q,a,b)?0
 :Math.min(distSeg(a,b,p[0],p[1]), distSeg(a,b,q[0],q[1]),
           distSeg(p,q,a[0],a[1]), distSeg(p,q,b[0],b[1]));

export function fixOnWall(f,tol,walls){
 const T=(tol==null)?120:tol;
 const P=frameOf(f), w=fixW(f)/2;
 const p=P(-w,0), q=P(w,0);      /* الظهرُ قطعةٌ لا ثلاثُ نقاط:
    ثلاثُ عيّناتٍ تفوت جداراً قصيراً يقع بينها. */
 const fb=bboxOf([p,q]);
 for(const wl of (walls||S.walls)){
  const bp=band(wl);
  if(!bp)continue;
  const wb=bboxOf(bp);
  if(!wb||!bboxHit(wb,fb,T))continue;
  for(let i=0;i<bp.length;i++)
   if(segGap(p,q,bp[i],bp[(i+1)%bp.length])<=T)return wl.id;
 }
 return null;
}
export function fixOverlap(a,b){
 const A=fixPoly(a), B=fixPoly(b);
 if(!bboxHit(bboxOf(A),bboxOf(B),-1))return false;
 return convexHit(A,B,-1);
}
/* ═══ الرموز ═══ */
const ELL=(P,cu,cv,ru,rv,n)=>{
 const out=[], N=n||20;
 for(let i=0;i<N;i++){
  const a=i/N*Math.PI*2;
  out.push(P(cu+ru*Math.cos(a), cv+rv*Math.sin(a)));
 }
 return out;
};
export function fixPrims(f){
 const L="A-FIXT", out=[], P=frameOf(f);
 const w=fixW(f), d=fixD(f), hw=w/2;
 const LN=(a,b)=>out.push({t:"line",L,a,b,fid:f.id});
 const PL=(pts,cl)=>out.push({t:"poly",L,pts,
  cl:cl===0?0:1,fid:f.id});
 const AR=(c,r)=>out.push({t:"arc",L,cx:c[0],cy:c[1],
  r:R(Math.max(2,r)),a0:0,a1:359.9,fid:f.id});
 const K=f.kind;

 if(K==="wc"||K==="bidet"){
  /* خزّان عند الظهر ثم قصعة بيضاوية */
  const tk=d*0.20;
  PL([P(-hw,0),P(hw,0),P(hw,tk),P(-hw,tk)],1);
  PL(ELL(P,0,tk+(d-tk)*0.52,hw*0.92,(d-tk)*0.50,22),1);
  if(K==="wc")LN(P(0,tk),P(0,tk+(d-tk)*0.12));
  return out;
 }
 if(K==="ur"){
  PL([P(-hw,0),P(hw,0),P(hw,d*0.30),
      P(hw*0.62,d*0.86),P(0,d),P(-hw*0.62,d*0.86),
      P(-hw,d*0.30)],1);
  PL(ELL(P,0,d*0.46,hw*0.52,d*0.30,16),1);
  return out;
 }
 if(K==="lav"){
  PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
  PL(ELL(P,0,d*0.52,hw*0.74,d*0.34,22),1);
  AR(P(0,d*0.52),Math.min(hw,d)*0.07);
  LN(P(0,0),P(0,d*0.12));                   /* الخلّاط */
  return out;
 }
 if(K==="sink"){
  PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
  const g=Math.min(w,d)*0.08;
  PL([P(-hw+g,g),P(hw-g,g),P(hw-g,d-g),P(-hw+g,d-g)],1);
  AR(P(-w*0.22,d*0.5),Math.min(hw,d)*0.06);
  AR(P( w*0.22,d*0.5),Math.min(hw,d)*0.06);
  LN(P(0,0),P(0,g*1.4));
  return out;
 }
 if(K==="shower"){
  PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
  LN(P(-hw,0),P(hw,d)); LN(P(hw,0),P(-hw,d));
  AR(P(0,d*0.5),Math.min(hw,d)*0.11);
  return out;
 }
 if(K==="tub"){
  PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
  const g=Math.min(w,d)*0.09;
  PL(ELL(P,0,d*0.5,hw-g,d*0.5-g,26),1);
  AR(P(-hw+g*2.2,d*0.5),Math.min(hw,d)*0.06);
  return out;
 }
 if(K==="wm"){
  PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
  AR(P(0,d*0.55),Math.min(hw,d)*0.42);
  LN(P(-hw,d*0.18),P(hw,d*0.18));
  return out;
 }
 /* صفاية أرضية */
 PL([P(-hw,0),P(hw,0),P(hw,d),P(-hw,d)],1);
 LN(P(-hw,0),P(hw,d)); LN(P(hw,0),P(-hw,d));
 return out;
}
export const fixLabel=f=>`${fixName(f)} ${dm2(fixW(f),fixD(f),"م")}`;
```

<a id="f-js-core-geom-js"></a>

---

## `js/core/stairs.js`

```javascript
/* ═══ الدرج ═══
   قِلعة مستقيمة: مسار سيرٍ من a إلى b وعرضٌ w وعددُ قوائم n.
   القياسات تُحسَب وتُعرَض ولا تُصحَّح: القائمة والنائمة و2ق+ن
   تُفحَص في inspect.js، فتُنبّه ولا تعدّل n ولا الطول.

   الأصل الفعليّ ثلاث كمّيات مخزَّنة: a, b, n. كل ما عداها مشتقّ
   عند الرسم. */
import {S,touchView,txtH} from "./state.js";
import {newId,clamp,D2R,R2D,deg,m2,m3,ltr,rng3} from "./units.js";
import {dist,pip,bboxOf} from "./geom.js";

const R=v=>Math.round(v);
export const stById=id=>S.stairs.find(s=>s.id===id)||null;
export const SMIN_W=600, SMIN_L=600;

/* المدى المريح — يُقاس ولا يُفرَض */
export const RISE_OK=[150,200];
export const TREAD_MIN=250;
export const RULE_OK=[580,650];        /* 2ق + ن */

export function stGeom(st){
 if(!st||!st.a||!st.b)return null;
 const dx=st.b[0]-st.a[0], dy=st.b[1]-st.a[1];
 const L=Math.hypot(dx,dy);
 if(L<1)return null;
 const ux=dx/L, uy=dy/L, nx=-uy, ny=ux;
 const hw=Math.max(SMIN_W,st.w)/2;
 const n=clamp(R(st.n)||2,2,80);
 const treads=n-1;                     /* آخر قائمة تصل البسطة */
 const tread=L/treads;
 const H=Math.max(200,+st.h||S.meta.wallH);
 const rise=H/n;
 const P=(s,v)=>[R(st.a[0]+ux*s+nx*v), R(st.a[1]+uy*s+ny*v)];
 return {L,ux,uy,nx,ny,hw,n,treads,tread,rise,H,P,
  ang:deg(Math.atan2(uy,ux)*R2D)};
}
export const stPoly=st=>{
 const g=stGeom(st);
 if(!g)return null;
 return [g.P(0,-g.hw),g.P(g.L,-g.hw),g.P(g.L,g.hw),g.P(0,g.hw)];
};
export const stBBox=st=>bboxOf(stPoly(st)||[]);
export const stAt=(x,y)=>{
 for(const s of S.stairs){
  const p=stPoly(s);
  if(p&&pip(p,x,y))return s;
 }
 return null;
};
export function addStair(a,b,w,n,ex){
 const A=[R(a[0]),R(a[1])], B=[R(b[0]),R(b[1])];
 const L=dist(A,B);
 if(L<SMIN_L)
  throw new Error(`طول القِلعة ${m3(L)} م — الأدنى ${m3(SMIN_L)} م`);
 const st={id:newId("S"),a:A,b:B,
  w:clamp(R(w||1000),SMIN_W,6000),
  n:clamp(R(n)||12,2,80),
  up:(ex&&ex.up==="dn")?"dn":"up",
  cut:0};
 if(ex){
  if(ex.h)st.h=clamp(R(ex.h),200,8000);
  if(ex.cut!=null)st.cut=clamp(+ex.cut||0,0,0.95);
 }
 S.stairs.push(st); touchView();
 return st;
}
export function delStair(st){
 const i=S.stairs.indexOf(st);
 if(i<0)return false;
 S.stairs.splice(i,1); touchView();
 return true;
}
/* ═══ الفحص — يقيس ولا يعدّل ═══ */
export function stCheck(st){
 const g=stGeom(st);
 if(!g)return {ok:0,msgs:["قِلعة صفرية الطول"],
  rise:0,tread:0,rule:0,n:0,treads:0};
 const m=[];
 const rule=2*g.rise+g.tread;
 if(g.rise<RISE_OK[0]||g.rise>RISE_OK[1])
  m.push(`القائمة ${m3(g.rise)} م خارج المدى المريح `
   +`${rng3(RISE_OK[0],RISE_OK[1],"م")}`);
 if(g.tread<TREAD_MIN)
  m.push(`النائمة ${m3(g.tread)} م أقلّ من ${m3(TREAD_MIN)} م`);
 if(rule<RULE_OK[0]||rule>RULE_OK[1])
  m.push(`قاعدة 2ق+ن = ${m3(rule)} م خارج `
   +`${rng3(RULE_OK[0],RULE_OK[1],"م")}`);
 if(st.w<900)
  m.push(`العرض ${m3(st.w)} م أقلّ من 0.900 م`);
 return {ok:m.length?0:1, msgs:m,
  rise:g.rise, tread:g.tread, rule, n:g.n, treads:g.treads};
}
/* ═══ الأوّليات ═══
   خطّ القطع: ما بعده يُرسَم متقطّعاً — الطابق الأعلى لا يظهر مصمَّتاً. */
export function stPrims(st){
 const g=stGeom(st);
 if(!g)return [];
 const L="A-STRS", out=[], h=txtH();
 const P=g.P;
 const cut=(st.cut>0.02)?g.L*st.cut:0;
 const dash=[h*1.5,h*0.9];
 const push=(a,b,beyond)=>out.push(beyond
  ? {t:"line",L,a,b,dash,sid:st.id}
  : {t:"line",L,a,b,sid:st.id});

 /* الجانبان */
 [-g.hw,g.hw].forEach(v=>{
  if(cut){
   push(P(0,v),P(cut,v),0);
   push(P(cut,v),P(g.L,v),1);
  }else push(P(0,v),P(g.L,v),0);
 });
 /* النائمات */
 for(let i=0;i<=g.treads;i++){
  const s=g.tread*i;
  push(P(s,-g.hw),P(s,g.hw), (cut&&s>cut)?1:0);
 }
 /* خطّ القطع: شرطتان مائلتان */
 if(cut){
  const d=g.hw*0.30, k=g.hw*0.34;
  out.push({t:"line",L,sid:st.id,
   a:P(cut-d,-g.hw*1.15), b:P(cut+d,g.hw*1.15)});
  out.push({t:"line",L,sid:st.id,
   a:P(cut-d+k,-g.hw*1.15), b:P(cut+d+k,g.hw*1.15)});
 }
 /* سهم الاتجاه على محور السير */
 const s0=g.tread*0.55;
 const s1=(cut?cut:g.L)-g.tread*0.55;
 if(s1>s0+h){
  const A=(st.up==="up")?P(s0,0):P(s1,0);
  const B=(st.up==="up")?P(s1,0):P(s0,0);
  out.push({t:"line",L,a:A,b:B,sid:st.id});
  const dx=B[0]-A[0], dy=B[1]-A[1], D=Math.hypot(dx,dy)||1;
  const ux=dx/D, uy=dy/D, nx=-uy, ny=ux, k=h*0.62;
  out.push({t:"poly",L,cl:1,sid:st.id,pts:[[B[0],B[1]],
   [R(B[0]-ux*k*1.9+nx*k*0.44),R(B[1]-uy*k*1.9+ny*k*0.44)],
   [R(B[0]-ux*k*1.9-nx*k*0.44),R(B[1]-uy*k*1.9-ny*k*0.44)]]});
  out.push({t:"arc",L,cx:A[0],cy:A[1],r:R(h*0.28),
   a0:0,a1:359.9,sid:st.id});
 }
 /* البطاقة */
 let rot=g.ang;
 if(rot>90.001&&rot<=270)rot=deg(rot+180);
 const m=P(g.L/2, g.hw+h*0.45);
 out.push({t:"text",L,sid:st.id,
  s:ltr(`${g.n} × ${m3(g.rise)} = ${m3(g.H)}`)+` م  `
   +`${st.up==="up"?"صاعد":"هابط"}`,
  x:m[0],y:m[1],h:h*0.9,al:"bc",rot});
 return out;
}
export const stLabel=st=>{
 const c=stCheck(st);
 return `${c.n} قائمة · ق ${m3(c.rise)} · ن ${m3(c.tread)} م`;
};
```

<a id="f-js-core-state-js"></a>

---

## `js/core/units.js`

```javascript
/* ═══ الوحدات · التطبيع · المعرّفات ═══
   الوحدة الداخلية: مليمتر صحيح · الإدخال: متر */

export const D2R=Math.PI/180, R2D=180/Math.PI;
export const clamp=(v,a,b)=>v<a?a:(v>b?b:v);

const AR="٠١٢٣٤٥٦٧٨٩", FA="۰۱۲۳۴۵۶۷۸۹";

export function norm(s){
 s=String(s==null?"":s);
 let o="";
 for(const ch of s){
  let i=AR.indexOf(ch); if(i<0)i=FA.indexOf(ch);
  o+=(i>=0)?String(i):ch;
 }
 return o.trim().toLowerCase()
  .replace(/[\u064B-\u0652\u0670\u0640]/g,"")
  .replace(/[أإآٱ]/g,"ا").replace(/ى/g,"ي")
  .replace(/ؤ/g,"و").replace(/ئ/g,"ي").replace(/ة/g,"ه")
  /* ٫ فاصلةٌ عشرية لا فاصلَ تعداد: تُبدَّل نقطةً لا فاصلة، وإلّا
     قُرِئت «٣٫٥» إحداثيَّ (3,5) في parsePt لا طولاً ٣٫٥ م.
     وplan.js يعالجها بيده بعد norm — فالمعنى كان يفترق. */
  .replace(/٫/g,".")
  .replace(/[،؛]/g,",");
}
/* متر → مليمتر · يقبل m cm mm ومرادفاتها العربية */
/* حدّ الإحداثي: نفس سقف MAXCO في io/dxfin.js (١٠٩ مم ≈ ١٠٠٠كم) —
   ثابتٌ محليٌّ هنا لا استيراد من io/، فلا تُخترَق طبقاتُ الاعتماد.
   بلا هذا الحدّ كان طولٌ مطبوعٌ يدوياً أو عمليةً من مزوّد الذكاء
   يمرّان بلا رفض، فيدخل المشروع رقمٌ لا يُصدَّر ولا يُرسَم بمعنى. */
const MAXCO=1e9;
/* ═══ الصيغة الصارمة ═══
   تعيد null لما لا يُفهَم. تستعملها المُثبِّتات ومسار المزوّد وسطر
   الإدخال. وM المتساهل يبقى للإدخال التفاعلي حيث الحقل الفارغ
   صفرٌ مقصود — فلا يتغيّر سلوكه بحرف. */
export function Mx(v){
 if(v==null||v==="")return null;
 if(typeof v==="number"){
  if(!isFinite(v))return null;
  const mm=Math.round(v*1000);
  return Math.abs(mm)<=MAXCO?mm:null;
 }
 const s=norm(v).replace(/\s+/g,"").replace(/,/g,".");
 const m=/^(-?\d*\.?\d+)(mm|cm|m|مم|سم|م)?$/.exec(s);
 if(!m)return null;
 const n=parseFloat(m[1]);
 if(!isFinite(n))return null;
 const u=m[2]||"m";
 const k=(u==="mm"||u==="مم")?1:((u==="cm"||u==="سم")?10:1000);
 const mm=Math.round(n*k);
 return Math.abs(mm)<=MAXCO?mm:null;
}
export function Nx(v){
 if(typeof v==="number")return isFinite(v)?v:null;
 const s=norm(String(v==null?"":v)).replace(/\s+/g,"")
  .replace(/,/g,".");
 if(!/^-?\d*\.?\d+$/.test(s))return null;
 const n=parseFloat(s);
 return isFinite(n)?n:null;
}
export const M=v=>{const r=Mx(v); return r==null?0:r};
export function isLen(v){
 if(typeof v==="number")return isFinite(v);
 return /^(-?\d*\.?\d+)(mm|cm|m|مم|سم|م)?$/
  .test(norm(String(v)).replace(/\s+/g,"").replace(/,/g,"."));
}
export const mm=v=>(v||0)/1000;
export const m2=v=>((v||0)/1000).toFixed(2);
export const m3=v=>((v||0)/1000).toFixed(3);
export const sqm=v=>((v||0)/1e6).toFixed(2);
/* بلا أصفار زائدة — للعرض في الحقول */
export const mnum=v=>{
 const s=((v||0)/1000).toFixed(3).replace(/0+$/,"").replace(/\.$/,"");
 return (s===""||s==="-0"||s===".")?"0":s;
};
/* ═══ عزل الاتجاه ═══
   المقدار المركَّب (9×14 · 1:100 · 0.5–4.5 · a→b) فيه فاصلٌ محايد
   بين رقمين. وقاعدة يونيكود تُسنِد المحايد إلى اتجاه الفقرة، فينقلب
   الرقمان في سياقٍ عربيّ: «9×14» تُقرأ «14×9».
   LRI…PDI يعزل المقدار في اتجاهٍ يساريّ صريح، ولا يقلب ما حوله.
   محرفان غير مرئيَّين، ويمرّان في textContent وفي fillText معاً. */
const LRI="\u2066", PDI="\u2069";
export const ltr=s=>LRI+String(s==null?"":s)+PDI;

/* المقادير المركَّبة الشائعة — تُستعمل في كل رسالة */
export const rng =(a,b,u)=>ltr(`${a}–${b}`)+(u?" "+u:"");
export const dim2=(a,b,u)=>ltr(`${a}×${b}`)+(u?" "+u:"");
export const scl =k=>ltr(`1:${k}`);
export const pair=(a,b)=>ltr(`${a} , ${b}`);
export const arrow=(a,b)=>ltr(`${a} → ${b}`);

/* مقاديرُ طولٍ مركَّبة — تختصر M+ltr في نداءٍ واحد */
export const rng2=(a,b,u)=>rng(m2(a),m2(b),u);
export const rng3=(a,b,u)=>rng(m3(a),m3(b),u);
export const dm2 =(a,b,u)=>dim2(m2(a),m2(b),u);
export const pt2 =p=>pair(m2(p[0]),m2(p[1]));

let IDC=0;
export const setIdc=v=>{IDC=Math.max(0,v|0)};
export const idc=()=>IDC;
export const bumpIdc=v=>{if((v|0)>IDC)IDC=v|0};
export const newId=p=>`${p}${++IDC}`;
export const idNum=id=>{
 const m=/(\d+)$/.exec(String(id||""));
 return m?parseInt(m[1],10):0;
};
export const deg=a=>((a%360)+360)%360;
```

<a id="f-js-core-walls-js"></a>

---

## `js/core/areas.js`

```javascript
/* ═══ المناطق المخبوزة ═══
   المنطقة كائن صريح: حلقة إحداثيات مخزَّنة، لا استنتاج يُعاد.
   تنقر داخل حلقة مغلقة مرّة، فتُخبَز مضلعاً يُسمّى ويُقاس ويُحرَّر
   بمقابضه. تغيير جدار لا يحرّكها: يجعلها «قديمة» بحدٍّ متقطّع،
   وأنت تحدّثها أو تثبّت بصمتها أو تتركها.

   لا تستورد render.js: الحلقات تُمرَّر إليها وسيطاً، فلا دورة.
   ولا sindex.js: فهرس صناديق الأجسام في walls.js — وهي تستورده
   سلفاً، فلا اتجاهَ يُقلَب. */
import {S,VER,touchView} from "./state.js";
import {newId,clamp,sqm,m2,m3} from "./units.js";
import {pArea,ccw,centroid,perim,pip,bboxOf,bboxHit,
        cleanRing} from "./geom.js";
import {band,wallsIn} from "./walls.js";
import {bump} from "./perf.js";

const R=v=>Math.round(v);
export const MINA=250000;          /* أصغر منطقة مقبولة: ٠٫٢٥ م² */
export const FILLS={none:"بلا",tint:"صبغة",hatch:"هاشور"};
export const areaById=id=>S.areas.find(a=>a.id===id)||null;

/* ═══ المساحة والمحيط ═══
   صافية بين الوجوه الداخلية، لأن الحلقة هي حدّ الفراغ نفسه. */
export const netArea=a=>Math.abs(pArea((a&&a.ring)||[]));
export const netPerim=a=>perim((a&&a.ring)||[]);
export const labelPt=a=>(a.lp?a.lp.slice():centroid(a.ring||[]));

/* ═══ البصمة ═══
   بصمة الجدران المجاورة للحلقة. حسّاسة بقصد: تُنبّه ولا تُصلح.
   لا تعرف الفتحات — الباب لا يغيّر امتداد الغرفة.
   ولا تعرف حالة العرض — إخفاء طبقة ليس تغييراً هندسياً. */
const hash=s=>{
 let h=0x811c9dc5;
 for(let i=0;i<s.length;i++){
  h^=s.charCodeAt(i);
  h=(h*0x01000193)>>>0;
 }
 return h.toString(36);
};
export function stampOf(ring){
 const b=bboxOf(ring);
 if(!b)return "";
 const pad=400;
 const Rc={x0:b.x0-pad,y0:b.y0-pad,x1:b.x1+pad,y1:b.y1+pad};
 const parts=[];
 /* المرشَّحون من فهرس الأجسام: جدارٌ صندوقه لا يلمس الحلقة لا
    يمكن أن يجاورها. والفحص بعده هو الفحص نفسه حرفاً بحرف،
    فالبصمة لا تتبدّل — ولو تبدّلت لصارت كل منطقةٍ محفوظة
    «قديمة» بمجرّد فتح الملفّ. */
 wallsIn(Rc).forEach(w=>{
  const wb=bboxOf(band(w)||[w.a,w.b]);
  if(!wb||!bboxHit(wb,Rc,0))return;
  parts.push(`${w.id}:${w.a[0]},${w.a[1]},${w.b[0]},${w.b[1]},`
   +`${w.t},${w.align},${w.type}`);
 });
 parts.sort();
 return parts.length?hash(parts.join("|")):"—";
}
/* ═══ كاش البصمة ═══
   isStale كان يُنادى مرّتين لكل منطقة في areaPrims، ثم في scene،
   ثم في inspect — أربع مرّاتٍ لكل منطقة في كل إطار.

   والمفتاح شيئان: النسخة الهندسية (تغيّر الجدران) وتوقيعُ الحلقة
   (تحرّك رأسٍ أو نقل المنطقة). والثاني لازم لأن سحب رأس منطقةٍ
   يغيّر جوارها ولا يُقدّم النسخة الهندسية — فمفتاحٌ بها وحدها
   يعرضها قديمةً وهي ليست، أو بالعكس. وتوقيعُ الحلقة رخيصٌ
   (ثمانية أزواج) مقابل stampOf التي تمسح الجوار وتبني band.

   وما يُخزَّن هو البصمة المحسوبة لا نتيجةُ المقارنة: فتثبيتُ
   البصمة (restamp) يقلب الجواب بلا إبطالٍ يدويّ. */
const SC=new Map();
const ringSig=r=>{
 const R2=r||[];
 let s=R2.length+":";
 for(let i=0;i<R2.length;i++)s+=R2[i][0]+","+R2[i][1]+";";
 return s;
};
export function stampNow(a){
 if(!a)return "";
 const rs=ringSig(a.ring);
 const hit=SC.get(a.id);
 if(hit&&hit.g===VER.g&&hit.rs===rs)return hit.sp;
 const sp=stampOf(a.ring);
 bump("stamp");
 if(SC.size>600)SC.clear();      /* لا ينمو بلا حدّ */
 SC.set(a.id,{g:VER.g,rs,sp});
 return sp;
}
export const isStale=a=>!!a&&a.stamp!==stampNow(a);
export const staleAreas=()=>S.areas.filter(isStale);
/* العدّ يمسح S.areas لا الكاش: منطقةٌ محذوفة تبقى في الكاش،
   ولو عُدَّ منه لأُبلِغتَ عن قديمةٍ لا وجود لها. */
export const staleCount=()=>{
 let n=0;
 S.areas.forEach(a=>{if(isStale(a))n++});
 return n;
};
export const stampStats=()=>({n:SC.size});
export const restamp=a=>{a.stamp=stampOf(a.ring); touchView(); return a};

/* ═══ إيجاد الحلقة المحيطة ═══
   حلقاتُ الأجسام تُقرأ بالتناوب — وهو ما يفعله الطلاءُ نفسه
   بـfill("evenodd"): عددٌ فرديٌّ من الحلقات الحاوية يعني صمتاً،
   وزوجيٌّ فراغاً حدُّه أعمقُها. والحلقاتُ الحاويةُ لنقطةٍ واحدة
   متداخلةٌ حتماً — مخرَجُ اتحادٍ لا تتقاطع حلقاتُه — فأصغرُها
   مساحةً هو أعمقُها.

   وكان «الأصغرُ مساحةً» وحدَه ثلاثةَ أعطاب: مركزُ عمودٍ منفردٍ
   يُعيد حلقتَه (٠٫١٦ م²) فترفضها addArea بحدِّ ٠٫٢٥، وجسمُ
   الجدار يُعيد قِشرةَ البناء كلَّها فتُخبَز منطقةٌ بمساحة المبنى،
   ورebake لغرفةٍ قطبُها في الصمت يعيد الخبزَ على القِشرة بلا كلمة. */
export function regionAt(loops,x,y){
 let n=0, best=null, ba=1/0;
 (loops||[]).forEach(lp=>{
  if(!lp||lp.length<3)return;
  if(!pip(lp,x,y))return;
  n++;
  const ar=Math.abs(pArea(lp));
  if(ar<ba){ba=ar; best=lp}
 });
 if(!n||(n&1))return null;          /* لا حلقةَ · أو صمت */
 return best.map(p=>[R(p[0]),R(p[1])]);
}
export const areaAt=(x,y,list)=>{
 let best=null, ba=1/0;
 (list||S.areas).forEach(a=>{
  if(!pip(a.ring,x,y))return;
  const ar=netArea(a);
  if(ar<ba){ba=ar;best=a}
 });
 return best;
};
/* ═══ الخبز ═══ */
export function addArea(ring,name,ex){
 const r=cleanRing(ccw(ring||[]),2);
 if(r.length<3)throw new Error("الحلقة أقلّ من ثلاثة أضلاع");
 const ar=Math.abs(pArea(r));
 if(ar<MINA)
  throw new Error(`المنطقة ${sqm(ar)} م² — الأصغر المقبول `
   +`${sqm(MINA)} م²`);
 const a={id:newId("A"),ring:r,name:String(name||"").slice(0,40),
  stamp:stampOf(r),showArea:1,fill:"tint"};
 if(ex){
  if(ex.showArea===0)a.showArea=0;
  if(FILLS[ex.fill])a.fill=ex.fill;
 }
 S.areas.push(a); touchView();
 return a;
}
export function delArea(a){
 const i=S.areas.indexOf(a);
 if(i<0)return false;
 S.areas.splice(i,1); touchView();
 return true;
}
/* إعادة الخبز من الهندسة الحالية · الاسم والخيارات تبقى.
   القطب المحسوب مرجعُ البحث؛ الموضع الصريح للاسم لا يُمَسّ. */
export function rebake(a,loops){
 const c=centroid(a.ring);
 const r=regionAt(loops,c[0],c[1]);
 if(!r)throw new Error(`${a.id}: لا حلقة مغلقة عند قطبها — `
  +`أغلق الجدران أو حرّك المنطقة`);
 const ar=Math.abs(pArea(r));
 if(ar<MINA)throw new Error(`${a.id}: الحلقة الجديدة ${sqm(ar)} م² فقط`);
 const before=netArea(a);
 a.ring=cleanRing(ccw(r),2);
 a.stamp=stampOf(a.ring);
 touchView();
 return {before,after:netArea(a)};
}
/* ═══ جدول المساحات ═══ */
export function schedule(){
 const rows=S.areas.map(a=>({id:a.id,
  name:a.name||"(بلا اسم)",
  ar:netArea(a), pr:netPerim(a), stale:isStale(a)}));
 rows.sort((x,y)=>y.ar-x.ar);
 return {rows,total:rows.reduce((s,r)=>s+r.ar,0)};
}
/* ═══ الأوّليات ═══
   القديمة: حدّ متقطّع وشارة — لا شيء يُصلَح خلسة.
   وisStale يُنادى مرّةً واحدة هنا فيُقرأ من الكاش. */
export function areaPrims(a,txtH){
 const out=[];
 const st=isStale(a);
 if(a.fill!=="none")
  out.push({t:"fill",L:"A-AREA",ring:a.ring,style:a.fill,aid:a.id});
 out.push({t:"poly",L:"A-AREA",pts:a.ring,cl:1,aid:a.id,
  dash:st?[420,300]:null, warn:st?1:0});
 const h=txtH, c=labelPt(a);
 const two=!!(a.name&&a.showArea);
 if(a.name)
  out.push({t:"text",L:"A-AREA",s:a.name,x:c[0],
   y:R(c[1]+(two?h*0.35:-h*0.5)),h,al:"mc",aid:a.id,
   warn:st?1:0});
 if(a.showArea)
  out.push({t:"text",L:"A-AREA",s:`${sqm(netArea(a))} م²`,
   x:c[0], y:R(c[1]-(two?h*1.35:h*0.5)), h:h*0.82, al:"mc",
   aid:a.id, warn:st?1:0});
 if(st)
  out.push({t:"text",L:"A-AREA",s:"قديمة",
   x:c[0], y:R(c[1]+(a.name?h*1.9:h*1.1)), h:h*0.7, al:"mc",
   aid:a.id, warn:1});
 return out;
}
export const areaLabel=a=>`${a.name||"(بلا اسم)"} · ${sqm(netArea(a))} م²`
 +(isStale(a)?" · قديمة":"");
```

<a id="f-js-core-autodim-js"></a>

---

## `js/core/dims.js`

```javascript
/* ═══ التأشير: الأبعاد والسلاسل والنصوص والمحاور ═══
   البُعد نقطتان صريحتان وموضعُ خطٍّ صريح. لا يرتبط بجدار ولا يزحف:
   القيمة المعروضة تُحسب من نقطتيه المخزَّنتين، فإن تغيّرت الهندسة
   بقي حيث هو وأُبلغتَ أنه «معلَّق».
   السلسلة قيَمٌ مكتوبة لا مطابَقة: تُرسَم كما كتبتها، والمقارنة
   بالهندسة تقريرٌ يُطلَب لا تصحيحٌ يقع. */
import {S,VER,touchView,txtH} from "./state.js";
import {newId,clamp,norm,m2,m3,mnum,deg,D2R,R2D} from "./units.js";
import {dist,nearOnSeg,bboxOf} from "./geom.js";
import {band} from "./walls.js";

const R=v=>Math.round(v);
export const DK={h:"أفقي",v:"رأسي",al:"محاذٍ"};
export const dimById  =id=>S.dims.find(d=>d.id===id)||null;
export const chainById=id=>S.chains.find(c=>c.id===id)||null;
export const annoById =id=>S.anno.find(a=>a.id===id)||null;

/* ═══ القيمة والصيغة ═══ */
export const dimValue=d=>{
 if(!d)return 0;
 if(d.kind==="h")return Math.abs(d.b[0]-d.a[0]);
 if(d.kind==="v")return Math.abs(d.b[1]-d.a[1]);
 return dist(d.a,d.b);
};
export const fmtLen=v=>{
 const n=clamp(parseInt(S.meta.dimDec,10)||0,0,3);
 return ((v||0)/1000).toFixed(n);
};
export const dimText=d=>(d.txt?String(d.txt):fmtLen(dimValue(d)));
export const isOverridden=d=>!!(d&&d.txt);

/* ═══ هندسة البُعد ═══
   pos: للأفقي y خطّ البُعد · للرأسي x · للمحاذي إزاحة عمودية موقَّعة */
export function dimGeom(d){
 if(!d)return null;
 if(d.kind==="h"){
  const y=d.pos;
  return {p1:[d.a[0],y], p2:[d.b[0],y], u:[1,0], n:[0,1], rot:0};
 }
 if(d.kind==="v"){
  const x=d.pos;
  return {p1:[x,d.a[1]], p2:[x,d.b[1]], u:[0,1], n:[-1,0], rot:90};
 }
 const dx=d.b[0]-d.a[0], dy=d.b[1]-d.a[1], L=Math.hypot(dx,dy);
 if(L<1)return null;
 const ux=dx/L, uy=dy/L, nx=-uy, ny=ux, o=d.pos;
 return {p1:[R(d.a[0]+nx*o),R(d.a[1]+ny*o)],
         p2:[R(d.b[0]+nx*o),R(d.b[1]+ny*o)],
         u:[ux,uy], n:[nx,ny], rot:deg(Math.atan2(uy,ux)*R2D)};
}
export const dimMid=d=>{
 const g=dimGeom(d);
 return g?[R((g.p1[0]+g.p2[0])/2),R((g.p1[1]+g.p2[1])/2)]:[0,0];
};
/* موضع خطّ البُعد من نقرة — يُخزَّن إحداثياً لا إزاحةً محسوبة */
export function posFromPt(kind,a,b,p){
 if(kind==="h")return R(p[1]);
 if(kind==="v")return R(p[0]);
 const dx=b[0]-a[0], dy=b[1]-a[1], L=Math.hypot(dx,dy);
 if(L<1)return 0;
 return R(((p[0]-a[0])*(-dy/L))+((p[1]-a[1])*(dx/L)));
}
/* ═══ البُعد المعلَّق ═══
   طرفٌ لا يصادف عقدةً ولا وجهاً — تقريرٌ بصريّ لا تعديل.
   والمراسي من الجدران وحدها، فمفتاحها النسخة الهندسية: كانت على
   النسخة العامّة فتُبنى مع كل إطارٍ أثناء سحب أي شيء. */
let ANC=null, ANCV=-1;
export function anchors(){
 if(ANCV===VER.g&&ANC)return ANC;
 const P=[];
 S.walls.forEach(w=>{
  P.push(w.a,w.b);
  const b=band(w);
  if(b)b.forEach(q=>P.push(q));
 });
 ANC=P; ANCV=VER.g;
 return P;
}
const ACELL=500;
let AG=null, AGV=-1;
function anchorGrid(){
 if(AGV===VER.g&&AG)return AG;
 const g=new Map();
 anchors().forEach(p=>{
  const k=Math.floor(p[0]/ACELL)+","+Math.floor(p[1]/ACELL);
  let a=g.get(k);
  if(!a){a=[]; g.set(k,a)}
  a.push(p);
 });
 AG=g; AGV=VER.g;
 return AG;
}
export function dimLoose(d,tol){
 const T=Math.max(1,tol||30);
 const G=anchorGrid();
 const r=Math.ceil(T/ACELL);
 const near=p=>{
  const cx=Math.floor(p[0]/ACELL), cy=Math.floor(p[1]/ACELL);
  for(let i=-r;i<=r;i++)for(let j=-r;j<=r;j++){
   const a=G.get((cx+i)+","+(cy+j));
   if(!a)continue;
   for(const q of a)
    if(Math.abs(q[0]-p[0])<=T&&Math.abs(q[1]-p[1])<=T)return true;
  }
  return false;
 };
 return !near(d.a)||!near(d.b);
}
export const looseDims=tol=>S.dims.filter(d=>dimLoose(d,tol));

/* ═══ إنشاء البُعد ═══ */
export function addDim(kind,a,b,pos,txt){
 const K=DK[kind]?kind:"h";
 const A=[R(a[0]),R(a[1])], B=[R(b[0]),R(b[1])];
 const d={id:newId("D"),kind:K,a:A,b:B,pos:R(pos||0)};
 const v=dimValue(d);
 if(v<10)throw new Error(
  `المقاس ${fmtLen(v)} م — النقطتان متطابقتان في هذا الاتجاه`);
 if(txt)d.txt=String(txt).slice(0,24);
 S.dims.push(d); touchView();
 return d;
}
export function delDim(d){
 const i=S.dims.indexOf(d);
 if(i<0)return false;
 S.dims.splice(i,1); touchView();
 return true;
}
/* ═══ العلامات ═══ شرطة معمارية أو سهم ═══ */
function tickPrims(L,p,u,n,s,style){
 if(style==="arrow"){
  const a=[p[0]+u[0]*s*1.6, p[1]+u[1]*s*1.6];
  return [
   {t:"line",L,a:[R(p[0]),R(p[1])],
    b:[R(a[0]+n[0]*s*0.4),R(a[1]+n[1]*s*0.4)]},
   {t:"line",L,a:[R(p[0]),R(p[1])],
    b:[R(a[0]-n[0]*s*0.4),R(a[1]-n[1]*s*0.4)]}];
 }
 const d=[(u[0]+n[0])*s, (u[1]+n[1])*s];
 return [{t:"line",L,
  a:[R(p[0]-d[0]),R(p[1]-d[1])], b:[R(p[0]+d[0]),R(p[1]+d[1])]}];
}
export function dimPrims(d){
 const g=dimGeom(d);
 if(!g)return [];
 const L="A-DIMS", h=txtH(), ts=h*0.42;
 const gap=h*0.32, over=h*0.55;
 const st=(S.meta.dimTick==="arrow")?"arrow":"slash";
 const out=[];
 const warn=dimLoose(d,30)?1:0;
 /* خطوط الامتداد: من النقطة الملتقطة إلى خطّ البُعد، بفجوة وتجاوز */
 [[d.a,g.p1],[d.b,g.p2]].forEach(([q,p])=>{
  const dx=p[0]-q[0], dy=p[1]-q[1], L2=Math.hypot(dx,dy);
  if(L2<gap+2)return;
  const ux=dx/L2, uy=dy/L2;
  out.push({t:"line",L,warn,
   a:[R(q[0]+ux*gap),R(q[1]+uy*gap)],
   b:[R(p[0]+ux*over),R(p[1]+uy*over)]});
 });
 out.push({t:"line",L,a:g.p1,b:g.p2,warn});
 tickPrims(L,g.p1,g.u,g.n,ts,st)
  .forEach(x=>out.push(Object.assign(x,{warn})));
 tickPrims(L,g.p2,[-g.u[0],-g.u[1]],g.n,ts,st)
  .forEach(x=>out.push(Object.assign(x,{warn})));
 const m=dimMid(d);
 let rot=g.rot;
 if(rot>90.001&&rot<=270)rot=deg(rot+180);   /* لا يُقرأ مقلوباً */
 const nx=Math.cos((rot+90)*D2R), ny=Math.sin((rot+90)*D2R);
 out.push({t:"text",L,s:dimText(d)+(d.txt?" *":""),
  x:R(m[0]+nx*h*0.42), y:R(m[1]+ny*h*0.42),
  h, al:"bc", rot, warn:warn||(d.txt?1:0)});
 return out;
}
/* ═══ السلاسل: قيَمٌ مكتوبة ═══ */
export const chainVals=c=>(c&&Array.isArray(c.vals))?c.vals:[];
export const chainSum=c=>chainVals(c).reduce((s,v)=>s+(+v||0),0);
export function chainBounds(c){
 const out=[0];
 let s=0;
 chainVals(c).forEach(v=>{s+=(+v||0); out.push(s)});
 return out;
}
/* 3 2.5 4 · أو 1.2*3 للتكرار */
export function parseVals(str){
 const T=norm(String(str||"")).split(/[\s,;+]+/).filter(Boolean);
 const out=[];
 T.forEach(t=>{
  const m=/^(\d*\.?\d+)(?:[x*](\d+))?$/.exec(t);
  if(!m)throw new Error(`«${t}» ليست قيمة — اكتب مثل: 3 2.5 4 أو 3*4`);
  const v=R(parseFloat(m[1])*1000);
  if(v<10)throw new Error(`القيمة «${t}» أصغر من سنتيمتر`);
  const n=m[2]?clamp(parseInt(m[2],10),1,60):1;
  for(let i=0;i<n;i++)out.push(v);
 });
 if(!out.length)throw new Error("لا قيَم — اكتب مثل: 3 2.5 4");
 if(out.length>60)throw new Error("أكثر من 60 قيمة");
 return out;
}
export function addChain(axis,base,pos,vals,total){
 const V=(vals||[]).map(v=>Math.max(10,R(+v||0)));
 if(!V.length)throw new Error("السلسلة بلا قيَم");
 const c={id:newId("C"),axis:(axis==="v")?"v":"h",
  base:[R(base[0]),R(base[1])], pos:R(pos),
  vals:V.slice(0,60), total:total?1:0};
 S.chains.push(c); touchView();
 return c;
}
export function delChain(c){
 const i=S.chains.indexOf(c);
 if(i<0)return false;
 S.chains.splice(i,1); touchView();
 return true;
}
export const chainPt=(c,s)=>(c.axis==="h")
 ? [R(c.base[0]+s), c.pos]
 : [c.pos, R(c.base[1]+s)];

export function chainPrims(c){
 const L="A-DIMS", h=txtH(), ts=h*0.42;
 const st=(S.meta.dimTick==="arrow")?"arrow":"slash";
 const u=(c.axis==="h")?[1,0]:[0,1];
 const n=(c.axis==="h")?[0,1]:[-1,0];
 const B=chainBounds(c), out=[];
 if(B.length<2)return out;
 const p0=chainPt(c,B[0]), pN=chainPt(c,B[B.length-1]);
 out.push({t:"line",L,a:p0,b:pN});
 B.forEach(s=>tickPrims(L,chainPt(c,s),u,n,ts,st)
  .forEach(x=>out.push(x)));
 const rot=(c.axis==="h")?0:90;
 const nx=Math.cos((rot+90)*D2R), ny=Math.sin((rot+90)*D2R);
 const V=chainVals(c);
 for(let i=0;i<V.length;i++){
  const m=chainPt(c,(B[i]+B[i+1])/2);
  out.push({t:"text",L,s:fmtLen(V[i]),
   x:R(m[0]+nx*h*0.42), y:R(m[1]+ny*h*0.42), h, al:"bc", rot});
 }
 if(c.total){
  const off=h*2.1;
  const q1=[R(p0[0]+nx*off),R(p0[1]+ny*off)];
  const q2=[R(pN[0]+nx*off),R(pN[1]+ny*off)];
  out.push({t:"line",L,a:q1,b:q2});
  tickPrims(L,q1,u,n,ts,st).forEach(x=>out.push(x));
  tickPrims(L,q2,[-u[0],-u[1]],n,ts,st).forEach(x=>out.push(x));
  const m=chainPt(c,(B[0]+B[B.length-1])/2);
  out.push({t:"text",L,s:fmtLen(chainSum(c)),
   x:R(m[0]+nx*(off+h*0.42)), y:R(m[1]+ny*(off+h*0.42)),
   h, al:"bc", rot});
 }
 return out;
}
/* ═══ المقارنة بالهندسة — تقرير لا تصحيح ═══
   لكل حدٍّ في السلسلة: أقرب إحداثيّ هندسيّ على المحور وفرقه.
   لا تُعدَّل قيمةٌ واحدة: أنت تقرأ وتقرّر. */
export function chainCompare(c,tol){
 const T=Math.max(1,tol||60);
 const idx=(c.axis==="h")?0:1;
 const uniq=[...new Set(anchors().map(p=>R(p[idx])))];
 const rows=chainBounds(c).map((s,i)=>{
  const q=chainPt(c,s)[idx];
  let best=null,bd=1/0;
  uniq.forEach(v=>{
   const d=Math.abs(v-q);
   if(d<bd){bd=d;best=v}
  });
  return {i,at:q,near:best,d:(best==null)?null:R(best-q),
   ok:(best!=null&&Math.abs(best-q)<=T)};
 });
 return {rows,sum:chainSum(c),off:rows.filter(r=>!r.ok).length};
}
/* ═══ النصوص والقوائد والمناسيب ═══
   مجموعة واحدة بحقل kind — أوفر من ثلاث مجموعات متشابهة. */
export const AK={text:"نصّ",lead:"قائد",level:"منسوب"};
export function addText(p,s,hMul,rot,al){
 const a={id:newId("T"),kind:"text",x:R(p[0]),y:R(p[1]),
  s:String(s==null?"":s).trim().slice(0,120),
  hm:clamp(+hMul||1,0.4,6), rot:deg(+rot||0),
  al:/^(bl|bc|ml|mc)$/.test(al)?al:"bc"};
 if(!a.s)throw new Error("النصّ فارغ");
 S.anno.push(a); touchView();
 return a;
}
export function addLead(pts,s,hMul){
 const P=(pts||[]).map(p=>[R(p[0]),R(p[1])]);
 if(P.length<2)throw new Error("القائد يحتاج نقطتين على الأقلّ");
 const a={id:newId("T"),kind:"lead",pts:P,
  s:String(s==null?"":s).trim().slice(0,120),
  hm:clamp(+hMul||1,0.4,6)};
 if(!a.s)throw new Error("نصّ القائد فارغ");
 S.anno.push(a); touchView();
 return a;
}
export function addLevel(p,z,pre){
 const a={id:newId("T"),kind:"level",x:R(p[0]),y:R(p[1]),
  z:R(z||0), pre:String(pre==null?"":pre).slice(0,8)};
 S.anno.push(a); touchView();
 return a;
}
export function delAnno(a){
 const i=S.anno.indexOf(a);
 if(i<0)return false;
 S.anno.splice(i,1); touchView();
 return true;
}
export const annoPt=a=>{
 if(!a)return [0,0];
 if(a.kind==="lead")return a.pts[a.pts.length-1].slice();
 return [a.x,a.y];
};
export const levelStr=a=>{
 const v=(a.z||0)/1000;
 const s=(v>=0?"+":"−")+Math.abs(v).toFixed(3);
 return (a.pre?a.pre+" ":"")+s;
};
export function annoPrims(a){
 const L="A-ANNO", h=txtH()*(a.hm||1);
 if(a.kind==="text")
  return [{t:"text",L,s:a.s,x:a.x,y:a.y,h,al:a.al||"bc",
   rot:a.rot||0}];
 if(a.kind==="lead"){
  const out=[], P=a.pts;
  for(let i=0;i<P.length-1;i++)
   out.push({t:"line",L,a:P[i],b:P[i+1]});
  /* رأس السهم عند النقطة الأولى — الاتجاه من الثانية إليها */
  const p=P[0], q=P[1];
  const dx=p[0]-q[0], dy=p[1]-q[1], D=Math.hypot(dx,dy)||1;
  const ux=dx/D, uy=dy/D, nx=-uy, ny=ux, s=h*0.5;
  out.push({t:"poly",L,cl:1,pts:[[p[0],p[1]],
   [R(p[0]-ux*s*1.9+nx*s*0.42),R(p[1]-uy*s*1.9+ny*s*0.42)],
   [R(p[0]-ux*s*1.9-nx*s*0.42),R(p[1]-uy*s*1.9-ny*s*0.42)]]});
  const e=P[P.length-1], b=P[P.length-2];
  const right=(e[0]>=b[0]);
  /* خطّ الكتف تحت النصّ */
  out.push({t:"line",L,a:e,
   b:[R(e[0]+(right?h*0.4:-h*0.4)),e[1]]});
  out.push({t:"text",L,s:a.s,
   x:R(e[0]+(right?h*0.5:-h*0.5)), y:R(e[1]+h*0.28),
   h, al:right?"bl":"bc"});
  return out;
 }
 /* المنسوب: مثلّث مفتوح وخطّ أرضية والقيمة */
 const s=txtH()*0.62;
 return [
  {t:"poly",L,cl:0,pts:[[R(a.x-s),R(a.y+s)],[a.x,a.y],
   [R(a.x+s),R(a.y+s)]]},
  {t:"line",L,a:[R(a.x-s*1.7),R(a.y+s)],b:[R(a.x+s*1.7),R(a.y+s)]},
  {t:"text",L,s:levelStr(a),x:a.x,y:R(a.y+s*1.5),
   h:txtH(),al:"bc"}];
}
/* ═══ المحاور ═══
   إحداثيات صريحة في S.grid · حروف للرأسي وأرقام للأفقي. */
const LTR="ABCDEFGHJKLMNPQRSTUVWXYZ";
export const axLabel=(dirv,i)=>(dirv==="x")
 ? (LTR[i%LTR.length]
   +(i>=LTR.length?String(1+Math.floor(i/LTR.length)):""))
 : String(i+1);
export function addAxis(dirv,v){
 const A=(dirv==="y")?S.grid.ys:S.grid.xs;
 const q=R(v);
 if(A.some(x=>Math.abs(x-q)<20))
  throw new Error("يوجد محور على هذا الإحداثي");
 A.push(q);
 A.sort((a,b)=>a-b);
 touchView();
 return q;
}
export function delAxis(dirv,v){
 const A=(dirv==="y")?S.grid.ys:S.grid.xs;
 let bi=-1, bd=1/0;
 A.forEach((x,i)=>{
  const d=Math.abs(x-v);
  if(d<bd){bd=d;bi=i}
 });
 if(bi<0||bd>200)return false;
 A.splice(bi,1); touchView();
 return true;
}
export function gridPrims(bbox){
 const X=S.grid.xs, Y=S.grid.ys;
 if(!X.length&&!Y.length)return [];
 const L="A-GRID", h=txtH(), r=h*1.1;
 let B=bbox;
 if(!B){
  const P=[];
  X.forEach(x=>P.push([x,0]));
  Y.forEach(y=>P.push([0,y]));
  B=bboxOf(P)||{x0:0,y0:0,x1:1000,y1:1000};
 }
 const pad=h*3.2;
 const x0=Math.min(B.x0,...(X.length?X:[B.x0]))-pad;
 const x1=Math.max(B.x1,...(X.length?X:[B.x1]))+pad;
 const y0=Math.min(B.y0,...(Y.length?Y:[B.y0]))-pad;
 const y1=Math.max(B.y1,...(Y.length?Y:[B.y1]))+pad;
 const out=[], dash=[h*1.6,h*0.7,h*0.25,h*0.7];
 X.forEach((x,i)=>{
  out.push({t:"line",L,a:[x,R(y0)],b:[x,R(y1)],dash});
  [[x,R(y1+r)],[x,R(y0-r)]].forEach(c=>{
   out.push({t:"arc",L,cx:c[0],cy:c[1],r:R(r),a0:0,a1:359.9});
   out.push({t:"text",L,s:axLabel("x",i),x:c[0],
    y:R(c[1]-h*0.36),h:h*0.92,al:"bc"});
  });
 });
 Y.forEach((y,i)=>{
  out.push({t:"line",L,a:[R(x0),y],b:[R(x1),y],dash});
  [[R(x0-r),y],[R(x1+r),y]].forEach(c=>{
   out.push({t:"arc",L,cx:c[0],cy:c[1],r:R(r),a0:0,a1:359.9});
   out.push({t:"text",L,s:axLabel("y",i),x:c[0],
    y:R(c[1]-h*0.36),h:h*0.92,al:"bc"});
  });
 });
 return out;
}
```

<a id="f-js-core-elevation-js"></a>

---

## `js/core/autodim.js`

```javascript
/* ═══ الأبعاد التلقائية للغرف ═══
   دوالٌّ خالصة: تقرأ حلقة منطقةٍ (ring بالمليمتر) وتعيد مواصفات
   الأبعاد والملصق — لا تكتب في الحالة. الأداةُ في tools/annotate.js
   هي من تُنشئ addDim/addText داخل edit() واحد.

   القرار: بُعدٌ أفقيّ أسفل الغرفة وبُعدٌ رأسيّ يسارها، بإزاحةٍ ثابتة
   عن حدود الحلقة، وملصقٌ «العرض×الطول» + المساحة في وسط الغرفة. */
import {bboxOf, pArea, centroid} from "./geom.js";
import {dm2} from "./units.js";

const R=v=>Math.round(v);
export function roomDims(ring, opt){
 const O=Object.assign({off:1000}, opt||{});
 const B=bboxOf(ring||[]);
 if(!B)return null;
 const w=B.x1-B.x0, h=B.y1-B.y0;
 if(w<300||h<300)return null;             /* أصغر من أن يُبعَّد */
 const off=Math.max(300, O.off);          /* إزاحة خطّ البُعد (مم) */
 const c=centroid(ring||[]);
 const area=Math.abs(pArea(ring||[]));
 const dims=[
  {kind:"h", a:[R(B.x0),R(B.y0)], b:[R(B.x1),R(B.y0)], pos:R(B.y0-off)},
  {kind:"v", a:[R(B.x0),R(B.y0)], b:[R(B.x0),R(B.y1)], pos:R(B.x0-off)}
 ];
 return {
  w, h, area,
  center:[R(c[0]),R(c[1])],
  dims,
  sizeText:dm2(w,h),                       /* معزولٌ ltr — «5.00×4.00» */
  areaText:(area/1e6).toFixed(2)+" م²"
 };
}
```

<a id="f-js-core-batch-js"></a>

---

