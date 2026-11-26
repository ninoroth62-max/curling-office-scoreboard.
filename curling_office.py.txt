#!/usr/bin/env python3
import json, os, socket
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse

PORT = 8765
DATA_FILE = os.path.join(os.path.dirname(__file__), "spielstand.json")
DEFAULT = {
 "mode":"", "ends":8, "current":1,
 "team1":"TEAM 1 / SKIP 1", "team2":"TEAM 2 / SKIP 2",
 "color1":"#15377e", "color2":"#f0eb00",
 "club":"Curling Wallisellen", "rink":"Rink 3",
 "scores1":[None]*8, "scores2":[None]*8, "finished":False, "forfeit":0
}
def load():
 try:
  with open(DATA_FILE,encoding="utf-8") as f:
   d=DEFAULT.copy(); d.update(json.load(f)); return d
 except (OSError,ValueError): return DEFAULT.copy()
STATE=load()
def save():
 with open(DATA_FILE,"w",encoding="utf-8") as f: json.dump(STATE,f,ensure_ascii=False,indent=2)
def ip():
 s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
 try: s.connect(("192.0.2.1",80)); return s.getsockname()[0]
 except OSError: return "127.0.0.1"
 finally: s.close()

HTML = r'''<!doctype html><html lang="de"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Curling Office</title><style>
*{box-sizing:border-box}body{margin:0;background:#eef1f5;color:#172033;font-family:Arial}header{background:#142f70;color:white;padding:15px 20px;display:flex;justify-content:space-between;gap:10px;flex-wrap:wrap}h1{font-size:22px;margin:0}main{max-width:1100px;margin:18px auto;padding:0 14px}nav,.actions{display:flex;gap:8px;flex-wrap:wrap;margin-bottom:14px}button{border:0;border-radius:8px;padding:10px 14px;background:#183b85;color:white;font-weight:bold;cursor:pointer}button.alt{background:#e2e7ef;color:#172033}button.red{background:#b42318}.panel{background:white;border:1px solid #d7dce5;border-radius:12px;padding:16px;margin-bottom:14px}.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(190px,1fr));gap:12px}label{display:block;font-size:13px;font-weight:bold;margin:0 0 5px}input,select{width:100%;padding:10px;border:1px solid #cbd2df;border-radius:7px;font:inherit}.stats{display:grid;grid-template-columns:repeat(4,1fr);gap:8px}.stat{background:#f0f3f8;border-radius:8px;text-align:center;padding:12px}.stat b{display:block;font-size:24px;margin-top:4px}.modegrid{display:grid;grid-template-columns:repeat(auto-fit,minmax(145px,1fr));gap:8px}.mode{background:#f5f7fb;color:#172033;border:2px solid #d7dce5;text-align:left}.mode.selected{border-color:#16834c;background:#e6f7ed}.ends{display:grid;grid-template-columns:70px 1fr 1fr;gap:7px;align-items:center;margin:7px 0}.ends input{text-align:center;font-size:18px;font-weight:bold}.muted{color:#647084}.scoregrid{display:grid;gap:2px;background:#222}.cell{background:#fafafa;color:#080808;min-width:0;min-height:55px;display:flex;align-items:center;justify-content:center;font-size:clamp(16px,2.5vw,29px);font-weight:900}.team{grid-column:1/-1;padding:9px 4px;text-align:center;font-weight:900;overflow-wrap:anywhere}.tv{background:#050505;color:white;border-radius:10px;overflow:hidden;border:3px solid #333}.tvtitle{text-align:center;padding:14px;font-weight:900;font-size:clamp(18px,3vw,32px)}.total{font-size:clamp(12px,2vw,21px)}.tvfoot{text-align:center;padding:10px;background:#111;font-size:13px}.hidden{display:none!important}.tvpage{max-width:1500px;margin:auto;padding:10px}#notice{padding:10px;color:#14532d;font-weight:bold}@media(max-width:600px){.stats{grid-template-columns:repeat(2,1fr)}.ends{grid-template-columns:55px 1fr 1fr}.cell{min-height:43px}}
</style></head><body><header><h1>🥌 CURLING OFFICE</h1><span id="status">Verbunden</span></header>
<main id="officeMain"><nav><button class="tab active" data-tab="office">Curl Office</button><button class="tab" data-tab="terminal">Eingabe-Terminal</button><button id="openTV">TV-Anzeige ↗</button></nav>
<section id="office"><div class="panel"><h2>Spielsteuerung</h2><div class="stats"><div class="stat">Spielmodus<b id="modeStat">Keiner</b></div><div class="stat">Aktuelles End<b id="endStat">–</b></div><div class="stat">Team 1 Total<b id="total1">0</b></div><div class="stat">Team 2 Total<b id="total2">0</b></div></div><div class="actions"><button id="prev">← Vorheriges End</button><button id="next">Nächstes End →</button><button class="red" id="finish">Spiel beenden</button><button class="alt" id="newGame">Neues Spiel</button></div></div>
<div class="panel"><h2>Teams & Anzeige</h2><div class="grid"><div><label>Team 1 / Skip 1</label><input id="team1"></div><div><label>Team 2 / Skip 2</label><input id="team2"></div><div><label>Farbe Team 1</label><input type="color" id="color1"></div><div><label>Farbe Team 2</label><input type="color" id="color2"></div><div><label>Clubname</label><input id="club"></div><div><label>Rink / Bahn</label><input id="rink"></div></div><div class="actions"><button id="saveDesign">Änderungen übernehmen</button></div></div>
<div class="panel"><h2>Spielstand bearbeiten</h2><div id="officeScores"></div></div><div class="panel"><h2>TV-Vorschau</h2><div id="preview"></div><p class="muted">Die TV-Seite aktualisiert sich automatisch über das lokale Netzwerk.</p></div></section>
<section id="terminal" class="hidden"><div class="panel"><h2>Spielmenü – Spielmodus auswählen</h2><div class="modegrid">
<button class="mode" data-mode="4">4<br><small>4 Ends</small></button><button class="mode" data-mode="6">6<br><small>6 Ends</small></button><button class="mode" data-mode="8">8<br><small>8 Ends</small></button><button class="mode" data-mode="8+">8+<br><small>bis 8 Ends</small></button><button class="mode" data-mode="MD">MD<br><small>Mixed Doubles · max. 6</small></button><button class="mode" data-mode="Skin 6">6+ Skinsgame<br><small>bis 6 Ends</small></button><button class="mode" data-mode="Skin 8">8+ Skinsgame<br><small>bis 8 Ends</small></button></div><div style="margin-top:12px"><label>Endanzahl</label><select id="endCount"></select></div><div class="actions"><button id="start">Spiel starten</button></div></div>
<div class="panel"><h2>Aktuelles End</h2><div class="actions" style="justify-content:center"><button class="alt" id="tPrev">←</button><h2 id="currentEnd">1</h2><button class="alt" id="tNext">→</button></div><div class="grid"><div><label id="label1">TEAM 1</label><input type="number" min="0" max="8" id="score1" value="0" style="font-size:26px;text-align:center;font-weight:bold"></div><div><label id="label2">TEAM 2</label><input type="number" min="0" max="8" id="score2" value="0" style="font-size:26px;text-align:center;font-weight:bold"></div></div><div class="actions"><button id="saveEnd">Steine speichern</button><button class="red" id="concede">X – Spiel aufgeben</button></div><p class="muted">Mixed Doubles maximal 6 Ends; andere Modi maximal 8 Ends. Stein-Eingabe 0–8.</p></div></section><div id="notice">Bereit.</div></main>
<section id="tvPage" class="tvpage hidden"><div class="actions"><button class="alt" id="backOffice">← Zurück zum Curl Office</button><button id="full">Vollbild</button></div><div id="tvFull"></div></section>
<script>
const $=id=>document.getElementById(id);let S=null,selectedMode='';
async function api(path,data){let r=await fetch(path,{method:data?'POST':'GET',headers:{'Content-Type':'application/json'},body:data?JSON.stringify(data):undefined});if(!r.ok)throw Error(await r.text());return r.json()}
function msg(t){$('notice').textContent=t}function total(a){return a.reduce((s,v)=>s+(Number(v)||0),0)}
function esc(x){return String(x??'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]))}
function contrast(h){let c=h.slice(1),r=parseInt(c.slice(0,2),16),g=parseInt(c.slice(2,4),16),b=parseInt(c.slice(4,6),16);return (r*299+g*587+b*114)/1000>145?'#111':'white'}
function tv(){if(!S.mode)return `<div style="background:white;color:#111;border-radius:10px;text-align:center;padding:35px"><div style="font-size:clamp(25px,5vw,48px);color:#3285bb">${esc(S.club)}</div><div style="font-size:clamp(45px,9vw,85px);margin-top:30px">${esc(S.rink)}</div></div>`;
let n=S.ends,h=`<div class="tv"><div class="tvtitle" style="background:${S.color1}">${esc(S.team1)} / ${esc(S.team2)}</div><div class="scoregrid" style="grid-template-columns:repeat(${n},minmax(24px,1fr)) minmax(55px,1.2fr)">`;
for(let i=0;i<n;i++)h+=`<div class="cell">${i+1}</div>`;h+='<div class="cell total">Total</div>';
for(let t=1;t<=2;t++){let name=t===1?S.team1:S.team2,color=t===1?S.color1:S.color2,a=t===1?S.scores1:S.scores2;h+=`<div class="team" style="background:${color};color:${contrast(color)}">${esc(name)}</div>`;for(let i=0;i<n;i++)h+=`<div class="cell">${a[i]===null?'':a[i]}</div>`;h+=`<div class="cell total">${total(a.slice(0,n))}</div>`}
return h+`</div><div class="tvfoot">${esc(S.mode)} · END ${S.current}/${S.ends}${S.finished?' · SPIEL BEENDET':''}${S.forfeit?' · AUFGEGEBEN':''}</div></div>`}
function render(){if(!S)return;$('modeStat').textContent=S.mode||'Keiner';$('endStat').textContent=S.mode?`${S.current}/${S.ends}`:'–';$('total1').textContent=total(S.scores1);$('total2').textContent=total(S.scores2);$('currentEnd').textContent=S.current;
for(let k of ['team1','team2','color1','color2','club','rink'])$(k).value=S[k];$('label1').textContent=S.team1;$('label2').textContent=S.team2;$('score1').value=S.scores1[S.current-1]??0;$('score2').value=S.scores2[S.current-1]??0;
$('officeScores').innerHTML=S.mode?Array.from({length:S.ends},(_,i)=>`<div class="ends"><button class="alt" data-end="${i+1}">End ${i+1}</button><input type="number" min="0" max="8" data-a="${i}" value="${S.scores1[i]??''}" placeholder="Team 1"><input type="number" min="0" max="8" data-b="${i}" value="${S.scores2[i]??''}" placeholder="Team 2"></div>`).join(''):'<p class="muted">Bitte zuerst einen Spielmodus auswählen.</p>';
$('officeScores').querySelectorAll('[data-end]').forEach(b=>b.onclick=()=>update({current:+b.dataset.end}));
$('officeScores').querySelectorAll('[data-a],[data-b]').forEach(i=>i.onchange=()=>{let x=+(i.dataset.a??i.dataset.b),k=i.dataset.a!==undefined?'scores1':'scores2',a=S[k].slice();a[x]=val(i.value);update({[k]:a})});
$('preview').innerHTML=tv();$('tvFull').innerHTML=tv()}
function val(x){return x===''?null:Math.max(0,Math.min(8,Math.floor(Number(x)||0)))}
async function update(d){try{S=await api('/api/state',d);render()}catch(e){msg('Fehler: '+e.message)}}
async function refresh(){try{let n=await api('/api/state');if(JSON.stringify(n)!==JSON.stringify(S)){S=n;render()}$('status').textContent='● Verbunden'}catch(e){$('status').textContent='Verbindung unterbrochen'}}
document.querySelectorAll('.tab').forEach(b=>b.onclick=()=>{document.querySelectorAll('.tab').forEach(x=>x.classList.toggle('active',x===b));$('office').classList.toggle('hidden',b.dataset.tab!=='office');$('terminal').classList.toggle('hidden',b.dataset.tab!=='terminal')});
$('openTV').onclick=()=>{$('officeMain').classList.add('hidden');$('tvPage').classList.remove('hidden');render()};$('backOffice').onclick=()=>{$('tvPage').classList.add('hidden');$('officeMain').classList.remove('hidden')};$('full').onclick=()=>document.documentElement.requestFullscreen?.();
$('saveDesign').onclick=()=>{let d={};for(let k of ['team1','team2','color1','color2','club','rink'])d[k]=$(k).value;update(d).then(()=>msg('Einstellungen gespeichert.'))};
document.querySelectorAll('[data-mode]').forEach(b=>b.onclick=()=>{selectedMode=b.dataset.mode;document.querySelectorAll('[data-mode]').forEach(x=>x.classList.toggle('selected',x===b));let max=selectedMode==='MD'?6:8,s=selectedMode==='4'?4:['6','Skin 6','MD'].includes(selectedMode)?6:8;$('endCount').innerHTML=Array.from({length:max},(_,i)=>`<option value="${i+1}">${i+1} Ends</option>`).join('');$('endCount').value=s});
$('start').onclick=()=>{if(!selectedMode)return msg('Bitte Spielmodus auswählen.');let n=+$('endCount').value;if(n>(selectedMode==='MD'?6:8))return msg('Endanzahl zu hoch.');update({mode:selectedMode,ends:n,current:1,scores1:Array(n).fill(null),scores2:Array(n).fill(null),finished:false,forfeit:0}).then(()=>msg('Spiel gestartet.'))};
async function move(d){if(S.mode)await update({current:Math.max(1,Math.min(S.ends,S.current+d))})}
$('prev').onclick=$('tPrev').onclick=()=>move(-1);$('next').onclick=$('tNext').onclick=()=>move(1);
$('saveEnd').onclick=()=>{let a=S.scores1.slice(),b=S.scores2.slice();a[S.current-1]=val($('score1').value);b[S.current-1]=val($('score2').value);update({scores1:a,scores2:b}).then(()=>msg('End gespeichert.'))};
$('finish').onclick=()=>update({finished:true}).then(()=>msg('Spiel beendet.'));
$('newGame').onclick=()=>{if(confirm('Neues Spiel starten und Spielstand zurücksetzen?')){selectedMode='';update({mode:'',current:1,ends:8,scores1:Array(8).fill(null),scores2:Array(8).fill(null),finished:false,forfeit:0})}};
$('concede').onclick=()=>{if(!S.mode)return msg('Kein aktives Spiel.');let t=confirm('Team 1 gibt auf? OK = Team 1, Abbrechen = Team 2.');update({forfeit:t?1:2,finished:true}).then(()=>msg('Spiel aufgegeben.'))};
refresh();setInterval(refresh,1000);
</script></body></html>'''

class Handler(BaseHTTPRequestHandler):
 def send(self,code,body,typ="application/json; charset=utf-8"):
  b=body.encode("utf-8"); self.send_response(code); self.send_header("Content-Type",typ); self.send_header("Content-Length",str(len(b))); self.send_header("Cache-Control","no-store"); self.end_headers(); self.wfile.write(b)
 def do_GET(self):
  p=urlparse(self.path).path
  if p=="/": self.send(200,HTML,"text/html; charset=utf-8")
  elif p=="/tv":
   page=HTML.replace('<main id="officeMain">','<main id="officeMain" class="hidden">').replace('<section id="tvPage" class="tvpage hidden">','<section id="tvPage" class="tvpage">')
   self.send(200,page,"text/html; charset=utf-8")
  elif p=="/api/state": self.send(200,json.dumps(STATE,ensure_ascii=False))
  else: self.send(404,'{"error":"not found"}')
 def do_POST(self):
  global STATE
  if urlparse(self.path).path!="/api/state": return self.send(404,'{"error":"not found"}')
  try:
   data=json.loads(self.rfile.read(int(self.headers.get("Content-Length","0"))) or b"{}")
   for k,v in data.items():
    if k in DEFAULT: STATE[k]=v
   STATE["ends"]=max(1,min(8,int(STATE.get("ends",8))))
   if STATE.get("mode")=="MD": STATE["ends"]=min(6,STATE["ends"])
   STATE["current"]=max(1,min(STATE["ends"],int(STATE.get("current",1))))
   for k in ("scores1","scores2"):
    a=list(STATE.get(k,[]))[:STATE["ends"]]; a += [None]*(STATE["ends"]-len(a))
    STATE[k]=[None if x is None else max(0,min(8,int(x))) for x in a]
   save(); self.send(200,json.dumps(STATE,ensure_ascii=False))
  except Exception as e: self.send(400,json.dumps({"error":str(e)}))
 def log_message(self,fmt,*args): print(fmt%args)

if __name__=="__main__":
 print("\\nCURLING OFFICE")
 print("Laptop: http://localhost:%s"%PORT)
 print("TV:     http://%s:%s/tv"%(ip(),PORT))
 print("Laptop und TV muessen im gleichen WLAN/LAN sein.")
 print("Dieses Fenster offen lassen. Beenden mit STRG+C.\\n")
 ThreadingHTTPServer(("0.0.0.0",PORT),Handler).serve_forever()
