<!doctype html>
<html lang="tr">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Grid Painter + Polygon Finder</title>
<style>
body{font-family:Arial;margin:16px;background:#fafafa}
.controls{display:flex;gap:12px;flex-wrap:wrap;margin-bottom:12px}
button{padding:6px 10px}
#canvasWrap{background:white;border:1px solid #ccc;display:inline-block}
#polyMenu{margin-top:10px;padding:10px;border:1px solid #ccc;background:#eee;display:none}
.polyBtn{display:block;margin:4px 0;padding:6px;text-align:left}
</style>
</head>
<body>

<h2>Grid Painter + Çokgen Bulma</h2>

<div class="controls">
  <label>Grid n <input id="n" type="number" value="4"></label>
  <label>Birim <input id="u" type="number" value="60"></label>
  <button id="regen">Yeniden</button>
  <button id="red">Kırmızı</button>
  <button id="blue">Mavi</button>
  <button id="erase">Sil</button>
  <button id="clear">Temizle</button>
  <button id="scan">Çokgen Tara</button>
  <button id="menuBtn">Menü</button>
</div>

<div id="canvasWrap"><canvas id="c"></canvas></div>
<div id="result"></div>
<div id="polyMenu"></div>

<script>
const canvas=document.getElementById('c');
const ctx=canvas.getContext('2d');
const inputN=document.getElementById('n');
const inputU=document.getElementById('u');
const polyMenu=document.getElementById('polyMenu');
const resultDiv=document.getElementById('result');

let mode='red';
let segments=[];
let detectedPolygons={red:[],blue:[]};
let highlighted=new Set();
let currentHighlightedKey=null;

function edgeKeyFromPoints(ax,ay,bx,by){
  return ax<bx||ax===bx&&ay<=by?`${ax},${ay}-${bx},${by}`:`${bx},${by}-${ax},${ay}`;
}
function edgeKey(s){return edgeKeyFromPoints(s.a.x,s.a.y,s.b.x,s.b.y);}
function toCanvas(p){
  const u=+inputU.value||60;
  return {x:p.x*u+10,y:p.y*u+10};
}

/* ---------- GRID ---------- */
function rebuild(){
  const n=+inputN.value||4;
  const u=+inputU.value||60;
  canvas.width=n*u+20;
  canvas.height=n*u+20;
  segments=[];
  detectedPolygons={red:[],blue:[]};
  highlighted.clear();
  polyMenu.innerHTML='';
  resultDiv.textContent='';
  for(let r=0;r<n;r++)for(let c=0;c<n;c++){
    let x=c,y=r;
    [
      [{x,y},{x:x+1,y}],
      [{x:x+1,y},{x:x+1,y:y+1}],
      [{x,y:y+1},{x:x+1,y:y+1}],
      [{x,y},{x,y:y+1}],
      [{x,y},{x:x+1,y:y+1}],
      [{x:x+1,y},{x,y:y+1}]
    ].forEach(e=>segments.push({a:e[0],b:e[1],color:null}));
  }
  mergeSegments();
  draw();
}
function mergeSegments(){
  const m=new Map();
  segments.forEach(s=>{const k=edgeKey(s);if(!m.has(k))m.set(k,s)});
  segments=[...m.values()];
}

/* ---------- DRAW ---------- */
function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  segments.forEach(s=>{
    const A=toCanvas(s.a),B=toCanvas(s.b);
    ctx.beginPath();
    ctx.moveTo(A.x,A.y);
    ctx.lineTo(B.x,B.y);
    ctx.lineWidth=highlighted.has(edgeKey(s))?7:3;
    ctx.strokeStyle=highlighted.has(edgeKey(s))?'yellow':s.color||'#555';
    ctx.stroke();
  });
}

/* ---------- CLICK ---------- */
canvas.onclick=e=>{
  const r=canvas.getBoundingClientRect();
  let best=null,d=1e9;
  segments.forEach(s=>{
    const A=toCanvas(s.a),B=toCanvas(s.b);
    const t=((e.clientX-r.left-A.x)*(B.x-A.x)+(e.clientY-r.top-A.y)*(B.y-A.y))/((B.x-A.x)**2+(B.y-A.y)**2||1);
    const cl=Math.max(0,Math.min(1,t));
    const dx=A.x+cl*(B.x-A.x)- (e.clientX-r.left);
    const dy=A.y+cl*(B.y-A.y)- (e.clientY-r.top);
    const dd=Math.hypot(dx,dy);
    if(dd<d){d=dd;best=s;}
  });
  if(best&&d<15){
    best.color=mode==='erase'?null:mode;
    highlighted.clear();
    draw();
  }
};

/* ---------- CONTROLS ---------- */
regen.onclick=rebuild;
red.onclick=()=>mode='red';
blue.onclick=()=>mode='blue';
erase.onclick=()=>mode='erase';
clear.onclick=()=>{segments.forEach(s=>s.color=null);highlighted.clear();draw();}
menuBtn.onclick=()=>polyMenu.style.display=polyMenu.style.display==='none'?'block':'none';

/* ===========================================================
   ===  EKLENEN GEOMETRİK FİLTRELER (KOD BOZULMADI)         ===
   =========================================================== */

function collinear(a,b,c){
  return (b.x-a.x)*(c.y-a.y)===(b.y-a.y)*(c.x-a.x);
}

function simplifyCycle(cycle){
  let pts=cycle.slice(0,-1).map(p=>{
    const[x,y]=p.split(',').map(Number);return{x,y};
  });
  let changed=true;
  while(changed){
    changed=false;
    for(let i=0;i<pts.length;i++){
      let A=pts[(i-1+pts.length)%pts.length];
      let B=pts[i];
      let C=pts[(i+1)%pts.length];
      if(collinear(A,B,C)){pts.splice(i,1);changed=true;break;}
    }
  }
  pts.push(pts[0]);
  return pts.map(p=>`${p.x},${p.y}`);
}

function intersect(p1,p2,p3,p4){
  function o(a,b,c){return (b.x-a.x)*(c.y-a.y)-(b.y-a.y)*(c.x-a.x);}
  let o1=o(p1,p2,p3),o2=o(p1,p2,p4),o3=o(p3,p4,p1),o4=o(p3,p4,p2);
  return o1*o2<0&&o3*o4<0;
}

function selfIntersecting(cycle){
  let p=cycle.slice(0,-1).map(s=>{const[x,y]=s.split(',');return{x:+x,y:+y}});
  for(let i=0;i<p.length;i++)for(let j=i+2;j<p.length;j++){
    if((j+1)%p.length===i)continue;
    if(intersect(p[i],p[(i+1)%p.length],p[j],p[(j+1)%p.length]))return true;
  }
  return false;
}

/* ---------- SCAN ---------- */
scan.onclick=()=>{
  const adj={red:{},blue:{}};
  const nk=p=>p.x+','+p.y;
  segments.forEach(s=>{
    if(!s.color)return;
    const A=nk(s.a),B=nk(s.b);
    adj[s.color][A]??=new Set();
    adj[s.color][B]??=new Set();
    adj[s.color][A].add(B);
    adj[s.color][B].add(A);
  });

  detectedPolygons={red:[],blue:[]};
  const found=new Set();

  function canonical(c){
    const a=c.slice(0,-1);
    const r=[];
    for(let i=0;i<a.length;i++)r.push(a.slice(i).concat(a.slice(0,i)));
    const rev=a.slice().reverse();
    for(let i=0;i<rev.length;i++)r.push(rev.slice(i).concat(rev.slice(0,i)));
    return r.map(x=>x.join('|')).sort()[0];
  }

  for(const color of ['red','blue']){
    const g=adj[color];
    Object.keys(g).forEach(start=>{
      const stack=[[start,[start]]];
      while(stack.length){
        const [n,p]=stack.pop();
        g[n].forEach(nb=>{
          if(nb===start&&p.length>=3){
            let raw=p.concat([start]);
            let simple=simplifyCycle(raw);
            if(simple.length<4||selfIntersecting(simple))return;
            const k=canonical(simple);
            if(!found.has(k)){found.add(k);detectedPolygons[color].push(simple);}
          }
          if(!p.includes(nb)&&nb>=start)stack.push([nb,p.concat(nb)]);
        });
      }
    });
  }

  polyMenu.innerHTML='';
  detectedPolygons.red.concat(detectedPolygons.blue).forEach((c,i)=>{
    let b=document.createElement('button');
    b.className='polyBtn';
    b.textContent=`${c.length-1}-gen`;
b.onclick=()=>{
  highlighted.clear();

  for(let i=0;i<c.length-1;i++){
    const [x1,y1]=c[i].split(',').map(Number);
    const [x2,y2]=c[i+1].split(',').map(Number);

    segments.forEach(s=>{
      // s, bu birleşik doğrunun üzerinde mi?
      const ax=s.a.x, ay=s.a.y, bx=s.b.x, by=s.b.y;

      // kollinear mi
      if((x2-x1)*(ay-y1)===(y2-y1)*(ax-x1) &&
         (x2-x1)*(by-y1)===(y2-y1)*(bx-x1)){

        // iki nokta da doğru parçası üzerinde mi
        const minx=Math.min(x1,x2), maxx=Math.max(x1,x2);
        const miny=Math.min(y1,y2), maxy=Math.max(y1,y2);

        if(ax>=minx&&ax<=maxx&&ay>=miny&&ay<=maxy &&
           bx>=minx&&bx<=maxx&&by>=miny&&by<=maxy){
          highlighted.add(edgeKey(s));
        }
      }
    });
  }

  draw();
};

    polyMenu.appendChild(b);
  });

  polyMenu.style.display='block';
  draw();
};

rebuild();
</script>
</body>
</html>
