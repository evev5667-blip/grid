<!doctype html>
<html lang="tr">
<head>
<meta charset="utf-8"/>
<title>Grid Painter + Çokgen Bulma</title>
<style>
body{font-family:Arial;margin:16px;background:#fafafa}
.controls{display:flex;gap:10px;flex-wrap:wrap;margin-bottom:12px}
button{padding:6px 10px}
#canvasWrap{background:white;border:1px solid #ccc;display:inline-block}
#polyMenu{margin-top:10px;padding:10px;border:1px solid #ccc;background:#eee;display:none}
.polyBtn{display:block;margin:4px 0;padding:6px;text-align:left}
#infoBox{display:none;margin-top:10px;padding:10px;border:1px solid #aaa;background:#fff}
</style>
</head>
<body>

<h2>Grid Painter + Çokgen Bulma</h2>

<div class="controls">
  <label>Grid n <input id="n" type="number" value="4"></label>
  <label>Birim <input id="u" type="number" value="60"></label>

  <button id="regen">Yeniden Oluştur</button>
  <button id="gridToggle">Grid Türü</button>
  <button id="paintToggle">Boyama Türü: Kenar</button>

  <button id="red">Kırmızı</button>
  <button id="blue">Mavi</button>
  <button id="erase">Sil</button>
  <button id="clear">Temizle</button>

  <button id="scan">Çokgen Tara</button>
  <button id="menuBtn">Menü</button>
  <button id="infoBtn">Program Açıklaması</button>
</div>

<div id="canvasWrap"><canvas id="c"></canvas></div>
<div id="polyMenu"></div>

<div id="infoBox">
<b>Program Nasıl Çalışır?</b><br><br>
• Grid üzerindeki çizgiler tıklanarak boyanır.<br>
• Aynı renkte kapalı çevrimler DFS ile bulunur.<br>
• Aynı doğrultudaki ardışık kenarlar tek kenar sayılır.<br>
• Menüden seçilen çokgenin gerçek kenarları vurgulanır.<br>
• Boyama türü kenar / doğru arasında değiştirilebilir.
</div>

<script>
const canvas=document.getElementById('c');
const ctx=canvas.getContext('2d');
const nInput=document.getElementById('n');
const uInput=document.getElementById('u');
const polyMenu=document.getElementById('polyMenu');
const infoBox=document.getElementById('infoBox');

let mode='red';
let paintType='edge';
let segments=[];
let highlighted=new Set();
let gridMode=1;

function edgeKey(s){
  const a=s.a,b=s.b;
  return a.x<b.x||a.x===b.x&&a.y<=b.y
    ?`${a.x},${a.y}-${b.x},${b.y}`
    :`${b.x},${b.y}-${a.x},${a.y}`;
}
function toCanvas(p){
  const u=+uInput.value||60;
  return {x:p.x*u+10,y:p.y*u+10};
}
function collinear(a,b,c){
  return (b.x-a.x)*(c.y-a.y)===(b.y-a.y)*(c.x-a.x);
}
function simplifyCycle(cycle){
  let pts=cycle.slice(0,-1).map(p=>{
    const[x,y]=p.split(',').map(Number);
    return{x,y};
  });
  let changed=true;
  while(changed){
    changed=false;
    for(let i=0;i<pts.length;i++){
      const A=pts[(i-1+pts.length)%pts.length];
      const B=pts[i];
      const C=pts[(i+1)%pts.length];
      if(collinear(A,B,C)){
        pts.splice(i,1);
        changed=true;
        break;
      }
    }
  }
  pts.push(pts[0]);
  return pts.map(p=>`${p.x},${p.y}`);
}

function rebuild(){
  const n=+nInput.value||4;
  const u=+uInput.value||60;
  canvas.width=n*u+20;
  canvas.height=n*u+20;

  segments=[];
  highlighted.clear();
  polyMenu.innerHTML='';

  for(let r=0;r<n;r++)for(let c=0;c<n;c++){
    const x=c,y=r;
    const edges=[
      [{x,y},{x:x+1,y}],
      [{x:x+1,y},{x:x+1,y:y+1}],
      [{x,y:y+1},{x:x+1,y:y+1}],
      [{x,y},{x,y:y+1}]
    ];
    if(gridMode){
      edges.push([{x,y},{x:x+1,y:y+1}]);
      edges.push([{x:x+1,y},{x,y:y+1}]);
    }
    edges.forEach(e=>segments.push({a:e[0],b:e[1],color:null}));
  }

  const m=new Map();
  segments.forEach(s=>{
    const k=edgeKey(s);
    if(!m.has(k))m.set(k,s);
  });
  segments=[...m.values()];
  draw();
}

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

canvas.onclick=e=>{
  const r=canvas.getBoundingClientRect();
  let best=null,d=1e9;
  segments.forEach(s=>{
    const A=toCanvas(s.a),B=toCanvas(s.b);
    const t=((e.clientX-r.left-A.x)*(B.x-A.x)+(e.clientY-r.top-A.y)*(B.y-A.y))
      /((B.x-A.x)**2+(B.y-A.y)**2||1);
    const cl=Math.max(0,Math.min(1,t));
    const dx=A.x+cl*(B.x-A.x)-(e.clientX-r.left);
    const dy=A.y+cl*(B.y-A.y)-(e.clientY-r.top);
    const dist=Math.hypot(dx,dy);
    if(dist<d){d=dist;best=s;}
  });
  if(!best||d>15)return;

  if(paintType==='edge'){
    best.color=mode==='erase'?null:mode;
  }else{
    segments.forEach(s=>{
      if(collinear(best.a,best.b,s.a)&&collinear(best.a,best.b,s.b)){
        s.color=mode==='erase'?null:mode;
      }
    });
  }
  highlighted.clear();
  draw();
};

regen.onclick=rebuild;
gridToggle.onclick=()=>{gridMode=1-gridMode;rebuild();};
red.onclick=()=>mode='red';
blue.onclick=()=>mode='blue';
erase.onclick=()=>mode='erase';
clear.onclick=()=>{segments.forEach(s=>s.color=null);highlighted.clear();draw();};

paintToggle.onclick=()=>{
  paintType=paintType==='edge'?'line':'edge';
  paintToggle.textContent=
    paintType==='edge'?'Boyama Türü: Kenar':'Boyama Türü: Doğru';
};

menuBtn.onclick=()=>polyMenu.style.display=
  polyMenu.style.display==='none'?'block':'none';
infoBtn.onclick=()=>infoBox.style.display=
  infoBox.style.display==='none'?'block':'none';

/* === ÇOKGEN TARAMA === */
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

  const found=new Set();
  polyMenu.innerHTML='';

  function canonical(c){
    const a=c.slice(0,-1);
    const r=[];
    for(let i=0;i<a.length;i++)r.push(a.slice(i).concat(a.slice(0,i)));
    const rev=a.slice().reverse();
    for(let i=0;i<rev.length;i++)r.push(rev.slice(i).concat(rev.slice(0,i)));
    return r.map(x=>x.join('|')).sort()[0];
  }

  ['red','blue'].forEach(color=>{
    const g=adj[color];
    Object.keys(g).forEach(start=>{
      const stack=[[start,[start]]];
      while(stack.length){
        const [n,p]=stack.pop();
        g[n].forEach(nb=>{
          if(nb===start&&p.length>=3){
            const raw=p.concat([start]);
            const simple=simplifyCycle(raw);
            if(simple.length<4)return;

            const key=canonical(simple);
            if(found.has(key))return;
            found.add(key);

            const kenar=simple.length-1;
            const renk=color==='red'?'kırmızı':'mavi';

            const b=document.createElement('button');
            b.className='polyBtn';
            b.textContent=`${found.size}. ${renk} ${kenar}-gen`;

            b.onclick=()=>{
              highlighted.clear();
              for(let i=0;i<simple.length-1;i++){
                const [x1,y1]=simple[i].split(',').map(Number);
                const [x2,y2]=simple[i+1].split(',').map(Number);
                segments.forEach(s=>{
                  if(
                    (x2-x1)*(s.a.y-y1)!==(y2-y1)*(s.a.x-x1)||
                    (x2-x1)*(s.b.y-y1)!==(y2-y1)*(s.b.x-x1)
                  )return;
                  const minx=Math.min(x1,x2),maxx=Math.max(x1,x2);
                  const miny=Math.min(y1,y2),maxy=Math.max(y1,y2);
                  if(
                    s.a.x>=minx&&s.a.x<=maxx&&
                    s.a.y>=miny&&s.a.y<=maxy&&
                    s.b.x>=minx&&s.b.x<=maxx&&
                    s.b.y>=miny&&s.b.y<=maxy
                  ) highlighted.add(edgeKey(s));
                });
              }
              draw();
            };
            polyMenu.appendChild(b);
          }
          if(!p.includes(nb)&&nb>=start)
            stack.push([nb,p.concat(nb)]);
        });
      }
    });
  });

  if(found.size===0){
    polyMenu.textContent='çokgen bulunamadı.';
  }

  polyMenu.style.display='block';
};

rebuild();
</script>
</body>
</html>
