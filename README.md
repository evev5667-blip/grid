# grid
grid
<!doctype html>
<html lang="tr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Grid Painter + Polygon Finder (Final - A-B-C-D)</title>
  <style>
    body{font-family:Arial,Helvetica,sans-serif;margin:16px;background:#fafafa}
    .controls{display:flex;gap:12px;flex-wrap:wrap;margin-bottom:12px;align-items:center}
    label{font-size:14px}
    input{padding:4px}
    button{padding:6px 10px;cursor:pointer}
    #canvasWrap{background:white;border:1px solid #ccc;display:inline-block;margin-bottom:10px}
    #polyMenu{margin-top:10px;padding:10px;border:1px solid #ccc;background:#eee;display:none;max-width:780px}
    .polyBtn{display:block;margin:4px 0;padding:6px;white-space:nowrap;text-align:left}
    #result{font-weight:bold;margin-top:10px}
  </style>
</head>
<body>

<h2>Grid Painter + Çokgen Bulma (Final - A→B→C→D)</h2>

<div class="controls">
  <label>Grid n: <input id="n" type="number" value="4" min="1" style="width:70px" /></label>
  <label>Birim px: <input id="u" type="number" value="60" min="10" style="width:80px" /></label>
  <button id="regen">Yeniden Oluştur</button>
  <button id="red">Kırmızı</button>
  <button id="blue">Mavi</button>
  <button id="erase">Sil</button>
  <button id="clear">Temizle</button>
  <button id="scan">Çokgen Tara</button>
  <button id="menuBtn">Çokgen Gösterim Menüsü</button>
</div>

<div id="canvasWrap"><canvas id="c"></canvas></div>

<div id="result"></div>
<div id="polyMenu"></div>

<script>
/* DOM refs */
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const inputN = document.getElementById('n');
const inputU = document.getElementById('u');
const regenBtn = document.getElementById('regen');
const redBtn = document.getElementById('red');
const blueBtn = document.getElementById('blue');
const eraseBtn = document.getElementById('erase');
const clearBtn = document.getElementById('clear');
const scanBtn = document.getElementById('scan');
const menuBtn = document.getElementById('menuBtn');
const polyMenu = document.getElementById('polyMenu');
const resultDiv = document.getElementById('result');

let mode = 'red';
let segments = [];
let detectedPolygons = { red: [], blue: [] };
let highlighted = new Set(); // edge keys that are currently highlighted
let currentHighlightedKey = null; // canonical cycle key for exclusive highlight

/* Utility helpers */
function edgeKeyFromPoints(ax,ay,bx,by){
  if(ax < bx || (ax === bx && ay <= by)) return `${ax},${ay}-${bx},${by}`;
  return `${bx},${by}-${ax},${ay}`;
}
function edgeKey(s){
  return edgeKeyFromPoints(s.a.x, s.a.y, s.b.x, s.b.y);
}
function toCanvas(p){
  const pad = 10;
  const unit = parseInt(inputU.value,10) || 60;
  return { x: p.x*unit + pad, y: p.y*unit + pad };
}

/* Rebuild grid */
function rebuild(){
  const n = parseInt(inputN.value,10) || 4;
  const unit = parseInt(inputU.value,10) || 60;
  const pad = 10;
  canvas.width = n*unit + pad*2;
  canvas.height = n*unit + pad*2;

  segments = [];
  detectedPolygons = { red: [], blue: [] };
  highlighted.clear();
  currentHighlightedKey = null;
  polyMenu.innerHTML = "";
  polyMenu.style.display = "none";
  resultDiv.textContent = "";

  for(let r=0;r<n;r++){
    for(let c=0;c<n;c++){
      const x = c, y = r;
      const base = [
        { a: {x:x,   y:y  }, b: {x:x+1, y:y  } }, // üst
        { a: {x:x+1, y:y  }, b: {x:x+1, y:y+1} }, // sağ
        { a: {x:x,   y:y+1}, b: {x:x+1, y:y+1} }, // alt
        { a: {x:x,   y:y  }, b: {x:x,   y:y+1} }, // sol
        { a: {x:x,   y:y  }, b: {x:x+1, y:y+1} }, // çapraz 1
        { a: {x:x+1, y:y  }, b: {x:x,   y:y+1} }  // çapraz 2
      ];
      base.forEach(s => segments.push({...s, color: null}));
    }
  }
  mergeSegments();
  draw();
}

function mergeSegments(){
  const map = new Map();
  for(const s of segments){
    const k = edgeKey(s);
    if(!map.has(k)) map.set(k, {...s});
  }
  segments = [...map.values()];
}

/* Drawing */
function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  for(const s of segments){
    const A = toCanvas(s.a), B = toCanvas(s.b);
    ctx.beginPath();
    ctx.moveTo(A.x, A.y);
    ctx.lineTo(B.x, B.y);
    const k = edgeKey(s);
    if(highlighted.has(k)){
      ctx.lineWidth = 7;
      ctx.strokeStyle = 'yellow';
    } else {
      ctx.lineWidth = 3;
      ctx.strokeStyle = s.color ? (s.color === 'red' ? 'red' : 'blue') : '#555';
    }
    ctx.stroke();
  }
}

/* Mouse click to color segments */
function dist(px,py,Ax,Ay,Bx,By){
  const vx = Bx-Ax, vy = By-Ay;
  const wx = px-Ax, wy = py-Ay;
  const c1 = vx*wx + vy*wy;
  const c2 = vx*vx + vy*vy;
  let t = c2 ? c1/c2 : 0;
  t = Math.max(0, Math.min(1, t));
  const X = Ax + vx*t, Y = Ay + vy*t;
  return Math.hypot(px-X, py-Y);
}
canvas.addEventListener('click', (e) => {
  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  let best = null, bestd = Infinity;
  for(const s of segments){
    const A = toCanvas(s.a), B = toCanvas(s.b);
    const d = dist(x,y,A.x,A.y,B.x,B.y);
    if(d < bestd){ bestd = d; best = s; }
  }
  if(best && bestd < 15){
    best.color = (mode === 'erase') ? null : mode;
    // If coloring changed, any current highlight that depends on that edge should be removed
    highlighted.delete(edgeKey(best));
    currentHighlightedKey = null;
    draw();
  }
});

/* Buttons */
regenBtn.addEventListener('click', rebuild);
redBtn.addEventListener('click', ()=> mode = 'red');
blueBtn.addEventListener('click', ()=> mode = 'blue');
eraseBtn.addEventListener('click', ()=> mode = 'erase');
clearBtn.addEventListener('click', ()=> {
  segments.forEach(s=> s.color = null);
  highlighted.clear();
  currentHighlightedKey = null;
  polyMenu.style.display = 'none';
  polyMenu.innerHTML = '';
  resultDiv.textContent = '';
  draw();
});
menuBtn.addEventListener('click', ()=> {
  polyMenu.style.display = polyMenu.style.display === 'none' ? 'block' : 'none';
});

/* Polygon scanning and menu creation
   -- REPLACED WITH A->B->C->D NODE-BASED CYCLE FINDER
*/
scanBtn.addEventListener('click', ()=> {
  resultDiv.textContent = 'Taranıyor...';

  // Build adjacency using segment endpoints only (A-B-C-D method)
  const adj = { red: {}, blue: {} };
  function nk(p){ return `${p.x},${p.y}`; }
  function addAdj(color,a,b){
    if(!adj[color][a]) adj[color][a] = new Set();
    if(!adj[color][b]) adj[color][b] = new Set();
    adj[color][a].add(b);
    adj[color][b].add(a);
  }

  for(const s of segments){
    if(!s.color) continue;
    const A = nk(s.a), B = nk(s.b);
    addAdj(s.color, A, B);
  }

  detectedPolygons = { red: [], blue: [] };
  const counts = { red: {}, blue: {} };

  // canonicalization for cycle (rotate + reverse)
  function canonicalCycle(arr){
    const a = arr.slice(0,-1); // drop duplicated last
    const n = a.length;
    const rots = [];
    for(let i=0;i<n;i++) rots.push(a.slice(i).concat(a.slice(0,i)));
    const rev = a.slice().reverse();
    for(let i=0;i<n;i++) rots.push(rev.slice(i).concat(rev.slice(0,i)));
    return rots.map(r=>r.join('|')).sort()[0];
  }

  // DFS-based cycle enumeration on node graph (no splitting at intersections)
  function findCyclesForColor(color){
    const g = adj[color];
    const nodes = Object.keys(g).sort();
    const found = new Set();

    for(const start of nodes){
      const stack = [{ node: start, path: [start] }];

      while(stack.length){
        const { node, path } = stack.pop();

        for(const nb of g[node]){
          if(nb === start && path.length >= 3){
            const cycle = path.concat([start]);
            const can = canonicalCycle(cycle);
            if(!found.has(can)){
              found.add(can);
              detectedPolygons[color].push(cycle);
              const sides = path.length;
              counts[color][sides] = (counts[color][sides] || 0) + 1;
            }
            continue;
          }
          if(path.includes(nb)) continue;
          // pruning: ensure monotonic order to avoid duplicates (only extend to nodes >= start lexicographically)
          if(nb < start) continue;
          stack.push({ node: nb, path: path.concat([nb]) });
        }
      }
    }
  }

  findCyclesForColor('red');
  findCyclesForColor('blue');

  // Menu creation: red first then blue, grouped by side count
  polyMenu.innerHTML = '';
  const grouped = { red: {}, blue: {} };

  for(const color of ['red','blue']){
    for(const cyc of detectedPolygons[color]){
      const sides = cyc.length - 1;
      if(!grouped[color][sides]) grouped[color][sides] = [];
      grouped[color][sides].push(cyc);
    }
  }

  const CNAME = { red: 'kırmızı', blue: 'mavi' };

  let any = false;
  for(const color of ['red','blue']){
    const sideCounts = Object.keys(grouped[color]).map(Number).sort((a,b)=>a-b);
    for(const sides of sideCounts){
      const list = grouped[color][sides];
      for(let i=0;i<list.length;i++){
        any = true;
        const idx = i+1;
        const type = sides===3 ? 'üçgen' : sides===4 ? 'dörtgen' : `${sides}-gen`;
        const btn = document.createElement('button');
        btn.className = 'polyBtn';
        btn.textContent = `${idx}. ${CNAME[color]} ${type}`;
        // create stable canonical key for this cycle
        const canKey = canonicalCycle(list[i]);

        btn.addEventListener('click', () => {
          toggleExclusiveByCanonical(color, canKey, list[i]);
        });

        polyMenu.appendChild(btn);
      }
    }
  }

  polyMenu.style.display = any ? 'block' : 'none';

  const parts = [];
  for(const color of ['red','blue']){
    const keys = Object.keys(counts[color]).map(Number).sort((a,b)=>a-b);
    for(const k of keys){
      const cnt = counts[color][k];
      const cname = CNAME[color];
      const t = k===3 ? 'üçgen' : k===4 ? 'dörtgen' : `${k}-gen`;
      parts.push(`${cnt} ${cname} ${t}`);
    }
  }
  resultDiv.textContent = parts.length ? parts.join(', ') : 'Hiç çokgen yok';
  draw();

  // toggle exclusive highlight by canonical key
  function toggleExclusiveByCanonical(color, canKey, cycle){
    // If same cycle clicked again -> turn off
    if(currentHighlightedKey === canKey){
      highlighted.clear();
      currentHighlightedKey = null;
      draw();
      return;
    }

    // Clear previous highlight
    highlighted.clear();

    // Add edges of this cycle if the segment exists and has matching color
    for(let i=0;i<cycle.length-1;i++){
      const [ax,ay] = cycle[i].split(',').map(Number);
      const [bx,by] = cycle[i+1].split(',').map(Number);
      const k = edgeKeyFromPoints(ax,ay,bx,by);
      // verify segment exists and has correct color
      const seg = segments.find(s => edgeKeyFromPoints(s.a.x,s.a.y,s.b.x,s.b.y) === k);
      if(seg && seg.color === color){
        highlighted.add(k);
      }
    }

    currentHighlightedKey = canKey;
    draw();
  }
});

/* İlk yüklemede grid kur */
rebuild();
</script>

</body>
</html>
>
