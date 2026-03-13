<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover,user-scalable=no">
<meta name="theme-color" content="#020912">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="OrbitWatch">
<meta name="description" content="Live ISS tracker, airspace anomaly monitor, emergency radio scanner. Created by @MrCodeBlood.">
<title>ORBIT WATCH — @MrCodeBlood</title>

<!-- Inline PWA Manifest as data URI -->
<script>
(function(){
  const manifest = {
    name:"ORBIT WATCH by @MrCodeBlood",
    short_name:"OrbitWatch",
    description:"Live ISS tracker, airspace anomaly monitor & emergency radio scanner",
    start_url:"./",
    display:"standalone",
    background_color:"#020912",
    theme_color:"#020912",
    orientation:"any",
    icons:[
      {src:"data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 512 512'%3E%3Crect width='512' height='512' fill='%23020912'/%3E%3Ccircle cx='256' cy='256' r='120' fill='none' stroke='%2300e5ff' stroke-width='6'/%3E%3Ccircle cx='256' cy='256' r='30' fill='%2300e5ff'/%3E%3Ccircle cx='376' cy='136' r='14' fill='%2300ff88'/%3E%3Ctext x='256' y='420' text-anchor='middle' fill='%2300e5ff' font-family='monospace' font-size='40' font-weight='bold'%3EORBIT%3C/text%3E%3C/svg%3E",sizes:"512x512",type:"image/svg+xml",purpose:"any maskable"}
    ]
  };
  const blob = new Blob([JSON.stringify(manifest)],{type:'application/manifest+json'});
  const link = document.createElement('link');
  link.rel='manifest'; link.href=URL.createObjectURL(blob);
  document.head.appendChild(link);
})();
</script>

<!-- Apple touch icon inline SVG -->
<link rel="apple-touch-icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 180 180'%3E%3Crect width='180' height='180' rx='40' fill='%23020912'/%3E%3Ccircle cx='90' cy='90' r='50' fill='none' stroke='%2300e5ff' stroke-width='4'/%3E%3Ccircle cx='90' cy='90' r='14' fill='%2300e5ff'/%3E%3Ccircle cx='140' cy='40' r='8' fill='%2300ff88'/%3E%3C/svg%3E">

<!-- Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Share+Tech+Mono&display=swap" rel="stylesheet">

<!-- External libs -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js" defer></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js" defer></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js" defer></script>

<style>
/* ═══════════════════════════════
   DESIGN TOKENS
═══════════════════════════════ */
:root{
  --bg:#020912; --panel:#070f1d; --panel2:#0a1525;
  --border:#0b3d6a; --border2:#0f4d86;
  --cyan:#00e5ff; --cyan2:#00bcd4;
  --green:#00ff88; --amber:#ffaa00;
  --red:#ff3366; --purple:#a855f7;
  --text:#94bdd6; --dim:#3d6680; --bright:#c8e8f8;
  --safe-top:env(safe-area-inset-top,0px);
  --safe-bot:env(safe-area-inset-bottom,0px);
  --safe-l:env(safe-area-inset-left,0px);
  --safe-r:env(safe-area-inset-right,0px);
}

/* ═══════════════════════════════
   RESET & BASE
═══════════════════════════════ */
*{margin:0;padding:0;box-sizing:border-box;-webkit-tap-highlight-color:transparent}
html{height:-webkit-fill-available}
body{
  background:var(--bg); color:var(--text);
  font-family:'Share Tech Mono',monospace;
  min-height:100vh; min-height:-webkit-fill-available;
  overflow-x:hidden;
  padding-top:var(--safe-top);
  padding-left:var(--safe-l);
  padding-right:var(--safe-r);
  padding-bottom:var(--safe-bot);
}
/* Scanlines overlay */
body::after{
  content:''; position:fixed; inset:0; pointer-events:none; z-index:9990;
  background:repeating-linear-gradient(0deg,transparent,transparent 3px,rgba(0,0,0,.04) 3px,rgba(0,0,0,.04) 4px);
}

/* ═══════════════════════════════
   ANIMATIONS
═══════════════════════════════ */
@keyframes blink{0%,100%{opacity:1}50%{opacity:.2}}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.5}}
@keyframes alertPulse{0%,100%{box-shadow:0 0 10px rgba(0,255,136,.2)}50%{box-shadow:0 0 25px rgba(0,255,136,.5)}}
@keyframes slideIn{from{opacity:0;transform:translateX(-8px)}to{opacity:1;transform:translateX(0)}}
@keyframes fadeIn{from{opacity:0;transform:translateY(4px)}to{opacity:1;transform:translateY(0)}}
@keyframes ebsScroll{from{transform:translateX(0)}to{transform:translateX(-100%)}}

/* ═══════════════════════════════
   STARS CANVAS
═══════════════════════════════ */
#stars-canvas{position:fixed;inset:0;pointer-events:none;z-index:0}

/* ═══════════════════════════════
   EBS TICKER
═══════════════════════════════ */
#ebs-bar{
  position:fixed; top:0; left:0; right:0; z-index:10000;
  height:34px; display:flex; align-items:stretch;
  font-family:'Share Tech Mono',monospace;
  pointer-events:none;
  transform:translateY(-100%); transition:transform .4s ease;
}
#ebs-bar.visible{transform:translateY(0)}
#ebs-label{
  flex-shrink:0; display:flex; align-items:center; padding:0 12px; gap:6px;
  font-family:'Orbitron',sans-serif; font-size:9px; letter-spacing:2px; font-weight:700;
  white-space:nowrap; pointer-events:auto;
}
#ebs-bar.ebs-alert{background:#8b0000}
#ebs-bar.ebs-warn{background:#7a4a00}
#ebs-bar.ebs-info{background:#003366}
#ebs-bar.ebs-alert #ebs-label{color:#fff;border-right:2px solid #f33}
#ebs-bar.ebs-warn  #ebs-label{color:#ffcc00;border-right:2px solid var(--amber)}
#ebs-bar.ebs-info  #ebs-label{color:#6cf;border-right:2px solid #08f}
#ebs-ticker-wrap{flex:1;overflow:hidden;display:flex;align-items:center;background:rgba(0,0,0,.6);backdrop-filter:blur(4px);border-bottom:1px solid rgba(255,255,255,.1)}
#ebs-ticker-text{white-space:nowrap;font-size:11px;letter-spacing:.5px;animation:ebsScroll 0s linear infinite;padding-left:100%}
#ebs-bar.ebs-alert #ebs-ticker-text{color:#faaa}
#ebs-bar.ebs-warn  #ebs-ticker-text{color:#ffe066}
#ebs-bar.ebs-info  #ebs-ticker-text{color:#adf}
#ebs-dismiss{
  flex-shrink:0; padding:0 12px; cursor:pointer; pointer-events:auto;
  display:flex; align-items:center; font-size:15px; opacity:.7;
  background:rgba(0,0,0,.4); border:none; color:inherit;
  min-width:40px; min-height:34px; /* touch target */
}
#ebs-dismiss:hover{opacity:1}
body.ebs-active{padding-top:calc(34px + var(--safe-top))}

/* ═══════════════════════════════
   SOUND TOGGLE (touch-friendly)
═══════════════════════════════ */
#sound-toggle{
  position:fixed; bottom:calc(18px + var(--safe-bot)); right:calc(18px + var(--safe-r)); z-index:9998;
  width:50px; height:50px; border-radius:50%;
  border:2px solid var(--border2); background:var(--panel);
  color:var(--cyan); font-size:20px; cursor:pointer;
  display:flex; align-items:center; justify-content:center;
  transition:all .2s; box-shadow:0 0 20px rgba(0,229,255,.2);
  touch-action:manipulation;
}
#sound-toggle:hover{background:var(--panel2);box-shadow:0 0 30px rgba(0,229,255,.4)}
#sound-toggle.muted{border-color:var(--dim);color:var(--dim);box-shadow:none}

/* Install banner */
#install-banner{
  position:fixed; bottom:calc(80px + var(--safe-bot)); right:calc(14px + var(--safe-r));
  z-index:9997; background:var(--panel); border:1px solid var(--cyan);
  padding:10px 14px; font-size:10px; letter-spacing:1px;
  box-shadow:0 0 20px rgba(0,229,255,.3); max-width:220px;
  display:none; flex-direction:column; gap:8px;
}
#install-banner b{color:var(--cyan)}
.install-btns{display:flex;gap:8px}
.install-btn{
  flex:1; padding:6px; border:1px solid; background:transparent;
  font-family:'Share Tech Mono',monospace; font-size:9px; letter-spacing:1px;
  cursor:pointer; touch-action:manipulation;
}
.install-btn.yes{border-color:var(--cyan);color:var(--cyan)}
.install-btn.no{border-color:var(--dim);color:var(--dim)}

/* ═══════════════════════════════
   LAYOUT
═══════════════════════════════ */
.app{
  position:relative; z-index:1;
  max-width:1500px; margin:0 auto;
  padding:8px; display:grid; gap:8px;
}

/* ═══════════════════════════════
   HEADER
═══════════════════════════════ */
.header{
  display:grid; grid-template-columns:auto 1fr auto;
  align-items:center; gap:16px; padding:12px 18px;
  background:var(--panel); border:1px solid var(--border);
  border-bottom:2px solid var(--cyan); position:relative;
}
.header::before{
  content:'◆ OPEN NOTIFY API — LIVE ◆';
  position:absolute; top:-1px; left:50%; transform:translateX(-50%);
  font-size:8px; letter-spacing:3px; color:var(--cyan);
  background:var(--panel); padding:0 8px;
}
.logo{
  font-family:'Orbitron',sans-serif; font-size:clamp(18px,3vw,26px);
  font-weight:900; color:var(--cyan);
  text-shadow:0 0 30px rgba(0,229,255,.6);
  letter-spacing:4px; white-space:nowrap;
}
.logo em{color:var(--green);font-style:normal}
.header-stats{display:flex;gap:0;align-items:stretch;border:1px solid var(--border);overflow-x:auto;-webkit-overflow-scrolling:touch}
.hstat{padding:7px 14px;border-right:1px solid var(--border);text-align:center;flex-shrink:0}
.hstat:last-child{border-right:none}
.hstat-label{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:3px;white-space:nowrap}
.hstat-val{font-family:'Orbitron',sans-serif;font-size:clamp(13px,1.8vw,17px);color:var(--cyan)}
.hstat-val.green{color:var(--green)} .hstat-val.amber{color:var(--amber)}
.header-right{display:flex;flex-direction:column;gap:6px;align-items:flex-end}
#alert-box{
  padding:6px 12px; border:1px solid var(--dim);
  font-size:9px; letter-spacing:1.5px; color:var(--dim); transition:all .4s;
  white-space:nowrap;
}
#alert-box.active{border-color:var(--green);color:var(--green);background:rgba(0,255,136,.08);box-shadow:0 0 20px rgba(0,255,136,.2);animation:alertPulse 1.2s infinite}
#alert-box.nearby{border-color:var(--amber);color:var(--amber);background:rgba(255,170,0,.08);animation:pulse 2s infinite}
.creator-tag{font-size:8px;letter-spacing:2px;color:var(--dim);display:flex;align-items:center;gap:4px}
.creator-tag span{color:var(--cyan)}

/* ═══════════════════════════════
   PANEL
═══════════════════════════════ */
.panel{background:var(--panel);border:1px solid var(--border);display:flex;flex-direction:column;position:relative;overflow:hidden}
.panel-head{
  display:flex;align-items:center;justify-content:space-between;
  padding:8px 12px; border-bottom:1px solid var(--border);
  font-size:8px; letter-spacing:2px; color:var(--cyan); flex-shrink:0;
}
.live-dot{width:7px;height:7px;border-radius:50%;background:var(--green);box-shadow:0 0 10px var(--green);animation:blink 1.4s infinite;flex-shrink:0}
.live-dot.amber{background:var(--amber);box-shadow:0 0 10px var(--amber)}
.live-dot.cyan{background:var(--cyan);box-shadow:0 0 10px var(--cyan)}
.live-dot.purple{background:var(--purple);box-shadow:0 0 10px var(--purple)}
.live-dot.red{background:var(--red);box-shadow:0 0 10px var(--red);animation:blink .8s infinite}
.loading-msg{padding:20px;text-align:center;font-size:9px;letter-spacing:2px;color:var(--dim);animation:pulse 1.5s infinite}

/* ═══════════════════════════════
   MAIN GRID (Map + Telemetry)
═══════════════════════════════ */
.main-grid{display:grid;grid-template-columns:1fr 300px;gap:8px}
#map-svg-wrap{flex:1;min-height:300px;position:relative}
#world-svg{width:100%;height:100%;display:block}
#world-svg .land{fill:#081f10;stroke:#00ff8818;stroke-width:.4}
#world-svg .graticule{fill:none;stroke:rgba(0,229,255,.05);stroke-width:.5}
#world-svg .equator{fill:none;stroke:rgba(0,229,255,.2);stroke-width:.8}
#world-svg .tropic{fill:none;stroke:rgba(255,170,0,.12);stroke-width:.5;stroke-dasharray:4,6}
#world-svg .orbit-track{fill:none;stroke:rgba(0,229,255,.22);stroke-width:1.2;stroke-dasharray:5,5}
.map-footer{padding:8px 12px;border-top:1px solid var(--border);display:flex;gap:16px;flex-shrink:0;font-size:9px;color:var(--dim);flex-wrap:wrap}
.map-footer b{color:var(--cyan)}
.right-col{display:flex;flex-direction:column;gap:8px}

/* Telemetry */
.tele-grid{display:grid;grid-template-columns:1fr 1fr;gap:1px;background:var(--border)}
.tele-cell{background:var(--panel);padding:11px 12px;display:flex;flex-direction:column;justify-content:space-between}
.tele-lbl{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:4px}
.tele-val{font-family:'Orbitron',sans-serif;font-size:clamp(12px,1.5vw,16px);color:var(--green);line-height:1}
.tele-val.c{color:var(--cyan)} .tele-val.a{color:var(--amber)}
.tele-val small{font-size:8px;color:var(--dim);font-family:'Share Tech Mono',monospace;margin-left:2px}
.orbit-prog-wrap{padding:10px 12px;border-top:1px solid var(--border);flex-shrink:0}
.prog-lbl{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:5px;display:flex;justify-content:space-between}
.prog-track{height:3px;background:rgba(0,229,255,.1)}
.prog-fill{height:100%;background:linear-gradient(90deg,var(--cyan2),var(--cyan));box-shadow:0 0 8px var(--cyan);transition:width 1s linear}

/* Crew */
.crew-scroll{overflow-y:auto;-webkit-overflow-scrolling:touch;max-height:260px}
.crew-item{display:flex;align-items:center;gap:10px;padding:9px 12px;border-bottom:1px solid rgba(11,61,106,.5);transition:background .2s}
.crew-item:hover{background:rgba(0,229,255,.03)}
.crew-item:last-child{border-bottom:none}
.crew-avatar{width:28px;height:28px;border-radius:50%;border:1px solid var(--cyan2);background:rgba(0,229,255,.1);display:flex;align-items:center;justify-content:center;font-size:13px;flex-shrink:0}
.crew-info{flex:1;min-width:0}
.crew-name{font-size:11px;color:var(--bright)}
.crew-craft{font-size:8px;color:var(--dim);letter-spacing:1px;margin-top:2px}
.crew-badge{font-size:7px;letter-spacing:1px;padding:2px 6px;border:1px solid var(--border2);color:var(--cyan2);flex-shrink:0}

/* ═══════════════════════════════
   3D ROW
═══════════════════════════════ */
.three-row{display:grid;grid-template-columns:1fr 320px;gap:8px}
#three-wrap{position:relative;min-height:400px;flex:1}
#three-canvas{position:absolute;inset:0;width:100%!important;height:100%!important;display:block;cursor:grab;touch-action:none}
#three-canvas:active{cursor:grabbing}
.three-hud{position:absolute;inset:0;pointer-events:none;padding:12px;display:flex;flex-direction:column;justify-content:space-between}
.hud-top{display:flex;justify-content:space-between;align-items:flex-start}
.hud-tag{font-size:8px;letter-spacing:2px;color:rgba(0,229,255,.5);border:1px solid rgba(0,229,255,.15);padding:3px 7px}
.hud-iss-pos{text-align:right;font-size:8px;color:rgba(0,229,255,.6);line-height:1.6}
.hud-bottom{display:flex;gap:14px}
.hud-stat{font-size:8px;color:rgba(0,229,255,.4);letter-spacing:1px}
.hud-stat b{color:rgba(0,229,255,.8)}

/* Orbital Mechanics */
.mech-panel{padding:14px;display:flex;flex-direction:column;gap:12px}
.mech-label{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:4px}
.mech-val{font-family:'Orbitron',sans-serif;font-size:clamp(16px,2vw,20px);color:var(--cyan)}
.mech-val.green{color:var(--green)} .mech-val.amber{color:var(--amber)}
.mech-val small{font-size:9px;color:var(--dim);margin-left:3px;font-family:'Share Tech Mono',monospace}
.mech-bar{margin-top:5px;height:2px;background:rgba(0,229,255,.1)}
.mech-bar-fill{height:100%;background:var(--cyan)}

/* ═══════════════════════════════
   INFO STRIP
═══════════════════════════════ */
.info-strip{display:grid;grid-template-columns:repeat(4,1fr);gap:1px;background:var(--border);border:1px solid var(--border)}
.istrip-cell{background:var(--panel2);padding:12px 14px;text-align:center}
.istrip-lbl{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:5px}
.istrip-val{font-family:'Orbitron',sans-serif;font-size:clamp(15px,2vw,20px);color:var(--cyan)}
.istrip-val.green{color:var(--green)}

/* ═══════════════════════════════
   ANOMALY MONITOR
═══════════════════════════════ */
.anomaly-section{display:grid;grid-template-columns:200px 1fr 230px;gap:8px}
.radar-canvas-wrap{flex:1;min-height:200px;display:flex;align-items:center;justify-content:center;background:radial-gradient(circle,#030e08 60%,var(--bg) 100%);overflow:hidden}
#radar-canvas{display:block}
.anomaly-stats{display:grid;grid-template-columns:1fr 1fr;gap:1px;background:var(--border);border-top:1px solid var(--border);flex-shrink:0}
.a-stat{background:var(--panel);padding:9px 10px;text-align:center}
.a-stat-lbl{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:3px}
.a-stat-val{font-family:'Orbitron',sans-serif;font-size:18px;color:var(--cyan)}
.a-stat-val.red{color:var(--red);text-shadow:0 0 12px rgba(255,51,102,.5)}
.a-stat-val.amber{color:var(--amber);text-shadow:0 0 12px rgba(255,170,0,.4)}
.a-stat-val.green{color:var(--green)}
.anomaly-feed{flex:1;overflow-y:auto;-webkit-overflow-scrolling:touch;max-height:320px}
.anomaly-feed::-webkit-scrollbar,.crew-scroll::-webkit-scrollbar,.station-list-wrap::-webkit-scrollbar{width:3px}
.anomaly-feed::-webkit-scrollbar-thumb,.crew-scroll::-webkit-scrollbar-thumb,.station-list-wrap::-webkit-scrollbar-thumb{background:var(--border2)}
.a-event{display:grid;grid-template-columns:auto 1fr auto;gap:8px;padding:9px 12px;border-bottom:1px solid rgba(11,61,106,.4);align-items:start;animation:slideIn .35s ease;transition:background .2s}
.a-event:hover{background:rgba(255,51,102,.04)}
.a-sev{width:8px;height:8px;border-radius:50%;margin-top:3px;flex-shrink:0}
.sev-critical{background:var(--red);box-shadow:0 0 8px var(--red);animation:blink .6s infinite}
.sev-high{background:#f60;box-shadow:0 0 8px #f60;animation:blink .9s infinite}
.sev-medium{background:var(--amber);box-shadow:0 0 6px var(--amber)}
.sev-low{background:var(--cyan);box-shadow:0 0 4px var(--cyan);opacity:.7}
.a-type{font-size:8px;letter-spacing:2px;color:var(--red);margin-bottom:2px}
.a-type.high{color:#f60} .a-type.medium{color:var(--amber)} .a-type.low{color:var(--cyan2)}
.a-flight{font-size:11px;color:var(--bright)}
.a-detail{font-size:8px;color:var(--dim);margin-top:2px;line-height:1.5}
.a-time{font-size:8px;color:var(--dim);white-space:nowrap}
.squawk-badge{display:inline-block;font-family:'Orbitron',sans-serif;font-size:9px;font-weight:700;padding:2px 6px;border:1px solid;margin-left:5px;vertical-align:middle}
.sq-7500{border-color:var(--red);color:var(--red);background:rgba(255,51,102,.1);animation:blink .7s infinite}
.sq-7600{border-color:#f60;color:#f60;background:rgba(255,102,0,.1)}
.sq-7700{border-color:var(--amber);color:var(--amber);background:rgba(255,170,0,.1)}
.anomaly-detail{display:flex;flex-direction:column;gap:0}
.a-info-block{padding:10px 12px;border-bottom:1px solid var(--border);flex-shrink:0}
.a-info-lbl{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:4px}
.a-info-val{font-family:'Orbitron',sans-serif;font-size:15px;color:var(--cyan)}
.a-info-val.red{color:var(--red)} .a-info-val.amber{color:var(--amber)}
.threat-tape{height:3px;background:rgba(255,51,102,.1);margin-top:5px;overflow:hidden}
.threat-fill{height:100%;background:linear-gradient(90deg,var(--amber),var(--red));box-shadow:0 0 8px var(--red);transition:width .6s ease}
.scan-status{padding:7px 12px;border-top:1px solid var(--border);font-size:8px;letter-spacing:2px;color:var(--dim);display:flex;justify-content:space-between;flex-shrink:0}
.scan-pulse{width:6px;height:6px;border-radius:50%;background:var(--red);box-shadow:0 0 6px var(--red);animation:blink .8s infinite;display:inline-block;margin-right:5px}
.scan-pulse.ok{background:var(--green);box-shadow:0 0 6px var(--green);animation:blink 2s infinite}
.anomaly-badge{display:none;align-items:center;justify-content:center;width:18px;height:18px;border-radius:50%;background:var(--red);color:#fff;font-size:9px;font-weight:bold;font-family:'Orbitron',sans-serif;margin-left:6px;box-shadow:0 0 10px rgba(255,51,102,.6);animation:blink 1s infinite}

/* ═══════════════════════════════
   RADIO PANEL
═══════════════════════════════ */
.radio-section{display:grid;grid-template-columns:1fr 1fr;gap:8px}
.radio-player{display:flex;flex-direction:column}
.now-playing{display:flex;align-items:center;gap:14px;padding:14px;border-bottom:1px solid var(--border);flex-shrink:0;background:linear-gradient(135deg,rgba(0,229,255,.04),transparent)}
.np-icon{width:44px;height:44px;border-radius:4px;border:1px solid var(--border2);background:rgba(0,229,255,.08);display:flex;align-items:center;justify-content:center;font-size:22px;flex-shrink:0;position:relative;overflow:hidden}
.np-icon.playing::after{content:'';position:absolute;inset:0;background:rgba(0,229,255,.15);animation:pulse 1s infinite}
.np-info{flex:1;min-width:0}
.np-station{font-size:12px;color:var(--bright);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.np-freq{font-family:'Orbitron',sans-serif;font-size:16px;color:var(--cyan);margin-top:2px}
.np-tag{font-size:8px;color:var(--dim);margin-top:2px;letter-spacing:1px}
.np-controls{display:flex;gap:7px;align-items:center;flex-shrink:0}
.radio-btn{
  width:38px;height:38px;border-radius:50%;
  border:1px solid var(--border2);background:transparent;
  color:var(--cyan);cursor:pointer;font-size:13px;
  display:flex;align-items:center;justify-content:center;transition:all .2s;
  touch-action:manipulation; /* prevent double-tap zoom */
  -webkit-touch-callout:none;
}
.radio-btn:hover{background:rgba(0,229,255,.1)}
.radio-btn.play-btn{width:44px;height:44px;font-size:17px;border-color:var(--cyan);box-shadow:0 0 14px rgba(0,229,255,.2)}
.radio-btn.play-btn.active{background:rgba(0,229,255,.15);box-shadow:0 0 20px rgba(0,229,255,.4);animation:pulse 1.5s infinite}
.vol-bar{display:flex;align-items:center;gap:8px;padding:9px 14px;border-bottom:1px solid var(--border);flex-shrink:0}
.vol-bar label{font-size:7px;letter-spacing:2px;color:var(--dim);white-space:nowrap}
#vol-slider{flex:1;-webkit-appearance:none;height:4px;background:rgba(0,229,255,.15);outline:none;border-radius:2px;cursor:pointer}
#vol-slider::-webkit-slider-thumb{-webkit-appearance:none;width:16px;height:16px;border-radius:50%;background:var(--cyan);cursor:pointer;box-shadow:0 0 8px var(--cyan)}
.station-list-wrap{flex:1;overflow-y:auto;-webkit-overflow-scrolling:touch;max-height:300px}
.station-item{display:flex;align-items:center;gap:10px;padding:10px 12px;border-bottom:1px solid rgba(11,61,106,.4);cursor:pointer;transition:background .15s;touch-action:manipulation}
.station-item:hover,.station-item:active{background:rgba(0,229,255,.04)}
.station-item.active{background:rgba(0,229,255,.08);border-left:2px solid var(--cyan)}
.station-item.emergency{border-left:2px solid var(--red)!important}
.station-item.emergency.active{background:rgba(255,51,102,.06)}
.si-icon{font-size:15px;width:20px;text-align:center;flex-shrink:0}
.si-info{flex:1;min-width:0}
.si-name{font-size:10px;color:var(--bright);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.si-meta{font-size:8px;color:var(--dim);margin-top:1px}
.si-freq{font-family:'Orbitron',sans-serif;font-size:10px;color:var(--cyan);flex-shrink:0}
.si-freq.red{color:var(--red)}
.freq-viz{height:30px;padding:6px 12px;border-top:1px solid var(--border);display:flex;align-items:flex-end;gap:2px;flex-shrink:0}
.freq-bar{flex:1;background:var(--cyan);opacity:.5;border-radius:1px 1px 0 0;transition:height .1s}
.emerg-badge{font-size:7px;padding:1px 4px;border:1px solid var(--red);color:var(--red);margin-left:4px;letter-spacing:1px}
.uptime-bar{display:flex;align-items:center;gap:10px;padding:6px 12px;background:rgba(0,255,136,.04);border-top:1px solid rgba(0,255,136,.12);flex-shrink:0;font-size:8px;letter-spacing:1.5px;color:var(--dim);flex-wrap:wrap}
.uptime-bar b{color:var(--green)}

/* Emerg freq reference */
.freq-ref-grid{display:grid;grid-template-columns:1fr 1fr;gap:1px;background:var(--border);flex-shrink:0}
.freq-ref-cell{background:var(--panel);padding:9px 10px}
.freq-ref-label{font-size:7px;letter-spacing:2px;color:var(--dim);margin-bottom:3px}
.freq-ref-val{font-family:'Orbitron',sans-serif;font-size:clamp(13px,1.5vw,16px)}
.freq-ref-note{font-size:8px;color:var(--dim);margin-top:1px}

/* ═══════════════════════════════
   FOOTER
═══════════════════════════════ */
.footer{text-align:center;padding:12px;font-size:8px;letter-spacing:2px;color:var(--dim);border-top:1px solid var(--border)}

/* ═══════════════════════════════
   RESPONSIVE — TABLET (< 1100px)
═══════════════════════════════ */
@media(max-width:1100px){
  .anomaly-section{grid-template-columns:190px 1fr}
  .anomaly-detail{display:none}
  .three-row{grid-template-columns:1fr}
  #three-wrap{min-height:320px}
}

/* ═══════════════════════════════
   RESPONSIVE — MOBILE (< 768px)
═══════════════════════════════ */
@media(max-width:768px){
  .app{padding:6px;gap:6px}
  .header{grid-template-columns:1fr;gap:8px;padding:10px 14px}
  .header::before{display:none}
  .header-stats{overflow-x:auto;-webkit-overflow-scrolling:touch}
  .main-grid,.three-row,.radio-section{grid-template-columns:1fr}
  .anomaly-section{grid-template-columns:1fr}
  .anomaly-detail{display:none}
  .right-col{gap:6px}
  #map-svg-wrap{min-height:220px}
  #three-wrap{min-height:280px}
  .info-strip{grid-template-columns:repeat(2,1fr)}
  .crew-scroll{max-height:200px}
  .anomaly-feed{max-height:260px}
  .station-list-wrap{max-height:220px}
  .mech-panel{padding:10px;gap:10px}
  #sound-toggle{width:44px;height:44px;font-size:18px}
  .hstat{padding:6px 10px}
}

/* ═══════════════════════════════
   RESPONSIVE — SMALL (< 480px)
═══════════════════════════════ */
@media(max-width:480px){
  .logo{font-size:20px;letter-spacing:3px}
  .hstat-label{font-size:6px}
  .hstat-val{font-size:13px}
  .header-right{flex-direction:row;align-items:center;justify-content:space-between;width:100%}
  .now-playing{gap:10px;padding:10px}
  .np-freq{font-size:14px}
  .istrip-val{font-size:14px}
  .panel-head{font-size:7px;letter-spacing:1.5px;padding:7px 10px}
  .a-event{padding:7px 10px}
  .station-item{padding:8px 10px}
  #ebs-label{font-size:8px;padding:0 8px}
}
</style>
</head>
<body>

<!-- EBS BAR -->
<div id="ebs-bar" class="ebs-info">
  <div id="ebs-label">
    <span id="ebs-dot" style="width:7px;height:7px;border-radius:50%;background:#fff;animation:blink .7s infinite;display:inline-block;flex-shrink:0"></span>
    <span id="ebs-type">EBS</span>
  </div>
  <div id="ebs-ticker-wrap">
    <div id="ebs-ticker-text">INITIALIZING ORBIT WATCH — FETCHING ALERTS...</div>
  </div>
  <button id="ebs-dismiss" aria-label="Dismiss">✕</button>
</div>

<!-- SOUND TOGGLE -->
<button id="sound-toggle" aria-label="Toggle sound">🔊</button>

<!-- INSTALL BANNER -->
<div id="install-banner">
  <div><b>📲 ADD TO HOME SCREEN</b><br>Install ORBIT WATCH as an app</div>
  <div class="install-btns">
    <button class="install-btn yes" id="install-yes">INSTALL</button>
    <button class="install-btn no" id="install-no">LATER</button>
  </div>
</div>

<canvas id="stars-canvas"></canvas>

<div class="app">

  <!-- HEADER -->
  <div class="header">
    <div class="logo">ORBIT<em>WATCH</em></div>
    <div class="header-stats">
      <div class="hstat"><div class="hstat-label">PEOPLE IN SPACE</div><div class="hstat-val green" id="h-people">—</div></div>
      <div class="hstat"><div class="hstat-label">ISS ALTITUDE</div><div class="hstat-val">408<span style="font-size:9px;color:var(--dim)"> KM</span></div></div>
      <div class="hstat"><div class="hstat-label">VELOCITY</div><div class="hstat-val">7.66<span style="font-size:9px;color:var(--dim)"> KM/S</span></div></div>
      <div class="hstat"><div class="hstat-label">PERIOD</div><div class="hstat-val">92.9<span style="font-size:9px;color:var(--dim)"> MIN</span></div></div>
      <div class="hstat"><div class="hstat-label">INCLINATION</div><div class="hstat-val">51.6<span style="font-size:9px;color:var(--dim)">°</span></div></div>
      <div class="hstat" style="border-right:none"><div class="hstat-label">UTC</div><div class="hstat-val amber" id="h-clock">--:--:--</div></div>
    </div>
    <div class="header-right">
      <div id="alert-box">● MONITORING ISS</div>
      <div style="display:flex;gap:14px;align-items:center">
        <div id="clock-display" style="font-family:'Orbitron',sans-serif;font-size:11px;color:var(--text);letter-spacing:2px">v4.0</div>
        <div class="creator-tag">BY <span>@MrCodeBlood</span></div>
      </div>
    </div>
  </div>

  <!-- MAP + TELEMETRY -->
  <div class="main-grid">
    <div class="panel">
      <div class="panel-head"><span>LIVE ISS TRACKING MAP</span><div class="live-dot"></div></div>
      <div id="map-svg-wrap"><svg id="world-svg"></svg></div>
      <div class="map-footer">
        <span>LAT <b id="f-lat">—</b></span>
        <span>LON <b id="f-lon">—</b></span>
        <span>YOU <b id="f-you">LOCATING…</b></span>
        <span style="margin-left:auto;font-size:8px">◉ ISS &nbsp;◉ YOU</span>
      </div>
    </div>
    <div class="right-col">
      <div class="panel">
        <div class="panel-head"><span>ISS TELEMETRY</span><div class="live-dot cyan"></div></div>
        <div class="tele-grid">
          <div class="tele-cell"><div class="tele-lbl">LATITUDE</div><div class="tele-val" id="t-lat">—</div></div>
          <div class="tele-cell"><div class="tele-lbl">LONGITUDE</div><div class="tele-val" id="t-lon">—</div></div>
          <div class="tele-cell"><div class="tele-lbl">ALTITUDE</div><div class="tele-val c">408<small>km</small></div></div>
          <div class="tele-cell"><div class="tele-lbl">VELOCITY</div><div class="tele-val c">27,600<small>km/h</small></div></div>
          <div class="tele-cell"><div class="tele-lbl">DIST FROM YOU</div><div class="tele-val a" id="t-dist">—</div></div>
          <div class="tele-cell"><div class="tele-lbl">TIME</div><div class="tele-val" style="font-size:11px" id="t-ts">—</div></div>
        </div>
        <div class="orbit-prog-wrap">
          <div class="prog-lbl"><span>ORBITAL PROGRESS</span><span id="prog-pct">0%</span></div>
          <div class="prog-track"><div class="prog-fill" id="prog-fill" style="width:0%"></div></div>
        </div>
      </div>
      <div class="panel" style="flex:1">
        <div class="panel-head"><span>CREW IN SPACE</span><div class="live-dot amber"></div></div>
        <div class="crew-scroll" id="crew-list"><div class="loading-msg">FETCHING CREW…</div></div>
      </div>
    </div>
  </div>

  <!-- 3D + MECHANICS -->
  <div class="three-row">
    <div class="panel">
      <div class="panel-head"><span>3D ORBITAL VIEW — DRAG · PINCH · SCROLL</span><div class="live-dot purple"></div></div>
      <div id="three-wrap">
        <canvas id="three-canvas"></canvas>
        <div class="three-hud">
          <div class="hud-top">
            <div class="hud-tag">INTERNATIONAL SPACE STATION</div>
            <div class="hud-iss-pos">
              <div id="hud-lat">LAT —</div><div id="hud-lon">LON —</div><div>ALT 408 km</div>
            </div>
          </div>
          <div class="hud-bottom">
            <div class="hud-stat">ORBIT <b id="hud-orbits">—</b></div>
            <div class="hud-stat">PERIOD <b>92.9 MIN</b></div>
            <div class="hud-stat">INCL <b>51.6°</b></div>
          </div>
        </div>
      </div>
    </div>
    <div class="panel">
      <div class="panel-head"><span>ORBITAL MECHANICS</span><div class="live-dot cyan"></div></div>
      <div class="mech-panel">
        <div><div class="mech-label">SEMI-MAJOR AXIS</div><div class="mech-val">6,779 <small>km</small></div><div class="mech-bar"><div class="mech-bar-fill" style="width:62%"></div></div></div>
        <div><div class="mech-label">ECCENTRICITY</div><div class="mech-val green">0.0006</div><div class="mech-bar"><div class="mech-bar-fill" style="width:4%;background:var(--green)"></div></div></div>
        <div><div class="mech-label">APOGEE</div><div class="mech-val">415 <small>km</small></div></div>
        <div><div class="mech-label">PERIGEE</div><div class="mech-val">402 <small>km</small></div></div>
        <div><div class="mech-label">ORBITS TODAY</div><div class="mech-val amber" id="m-orbits">—</div></div>
        <div><div class="mech-label">REVOLUTIONS SINCE LAUNCH</div><div class="mech-val" style="font-size:15px">~145,000+</div></div>
      </div>
    </div>
  </div>

  <!-- INFO STRIP -->
  <div class="info-strip">
    <div class="istrip-cell"><div class="istrip-lbl">ISS LAUNCH</div><div class="istrip-val" style="font-size:12px">NOV 20, 1998</div></div>
    <div class="istrip-cell"><div class="istrip-lbl">MASS</div><div class="istrip-val">420,000 <span style="font-size:9px;color:var(--dim)">KG</span></div></div>
    <div class="istrip-cell"><div class="istrip-lbl">SOLAR SPAN</div><div class="istrip-val">73 <span style="font-size:9px;color:var(--dim)">M</span></div></div>
    <div class="istrip-cell"><div class="istrip-lbl">DAYS IN ORBIT</div><div class="istrip-val green" id="days-orbit">—</div></div>
  </div>

  <!-- ANOMALY MONITOR -->
  <div class="panel" style="border-color:rgba(255,51,102,.3);border-top:2px solid var(--red)">
    <div class="panel-head" style="border-bottom-color:rgba(255,51,102,.25)">
      <span style="color:var(--red)">⚡ AIRSPACE ANOMALY MONITOR <span class="anomaly-badge" id="anomaly-badge">0</span></span>
      <div style="display:flex;gap:12px;align-items:center">
        <span style="font-size:8px;color:var(--dim)" id="scan-region">GLOBAL</span>
        <div class="live-dot red"></div>
      </div>
    </div>
    <div class="anomaly-section">
      <!-- Radar -->
      <div class="panel" style="border:none">
        <div class="panel-head" style="font-size:7px;border-bottom-color:rgba(255,51,102,.2);color:var(--red)">
          <span>RADAR SWEEP</span><span id="radar-aircraft-count" style="color:var(--dim)">—</span>
        </div>
        <div class="radar-canvas-wrap"><canvas id="radar-canvas" width="190" height="190"></canvas></div>
        <div class="anomaly-stats">
          <div class="a-stat"><div class="a-stat-lbl">CRITICAL</div><div class="a-stat-val red" id="cnt-critical">0</div></div>
          <div class="a-stat"><div class="a-stat-lbl">HIGH</div><div class="a-stat-val amber" id="cnt-high">0</div></div>
          <div class="a-stat"><div class="a-stat-lbl">MEDIUM</div><div class="a-stat-val" id="cnt-medium">0</div></div>
          <div class="a-stat"><div class="a-stat-lbl">SCANNED</div><div class="a-stat-val green" id="cnt-scanned">—</div></div>
        </div>
      </div>
      <!-- Feed -->
      <div class="panel" style="border:none;display:flex;flex-direction:column">
        <div class="panel-head" style="font-size:7px;border-bottom-color:rgba(255,51,102,.2)">
          <span style="color:var(--dim)">LIVE ANOMALY LOG</span>
          <span style="color:var(--dim);font-size:7px" id="feed-count">0 EVENTS</span>
        </div>
        <div class="anomaly-feed" id="anomaly-feed"><div class="loading-msg" style="color:var(--dim)">INITIALIZING SCANNER…</div></div>
        <div class="scan-status">
          <span><span class="scan-pulse ok" id="scan-dot"></span><span id="scan-label">SCANNING</span></span>
          <span id="last-scan">—</span>
        </div>
      </div>
      <!-- Reference (hidden on mobile) -->
      <div class="panel anomaly-detail" style="border:none">
        <div class="panel-head" style="font-size:7px;color:var(--red);border-bottom-color:rgba(255,51,102,.2)"><span>SQUAWK REFERENCE</span></div>
        <div class="a-info-block"><div class="a-info-lbl">7500 — HIJACK</div><div class="a-info-val red">CRITICAL</div><div class="threat-tape"><div class="threat-fill" style="width:100%"></div></div><div style="font-size:8px;color:var(--dim);margin-top:5px">Unlawful interference. Silent distress.</div></div>
        <div class="a-info-block"><div class="a-info-lbl">7600 — RADIO FAILURE</div><div class="a-info-val" style="color:#f60">HIGH</div><div class="threat-tape"><div class="threat-fill" style="width:70%;background:linear-gradient(90deg,#f60,#f90)"></div></div><div style="font-size:8px;color:var(--dim);margin-top:5px">Loss of communication.</div></div>
        <div class="a-info-block"><div class="a-info-lbl">7700 — EMERGENCY</div><div class="a-info-val amber">HIGH</div><div class="threat-tape"><div class="threat-fill" style="width:75%;background:linear-gradient(90deg,var(--amber),#f80)"></div></div><div style="font-size:8px;color:var(--dim);margin-top:5px">General emergency. Engine/medical/fuel.</div></div>
        <div class="a-info-block"><div class="a-info-lbl">RAPID DESCENT</div><div class="a-info-val amber">MEDIUM</div><div class="threat-tape"><div class="threat-fill" style="width:50%;background:linear-gradient(90deg,var(--cyan2),var(--amber))"></div></div><div style="font-size:8px;color:var(--dim);margin-top:5px">Vert rate &lt;−15 m/s (&gt;3,000 ft/min).</div></div>
        <div class="a-info-block" style="border-bottom:none"><div class="a-info-lbl">SPI IDENT / SPEED</div><div class="a-info-val" style="color:var(--cyan)">LOW</div><div style="font-size:8px;color:var(--dim);margin-top:5px">ATC ident signal or extreme low-alt speed.</div></div>
      </div>
    </div>
  </div>

  <!-- RADIO -->
  <div class="radio-section">
    <div class="panel radio-player">
      <div class="panel-head"><span>📻 EMERGENCY RADIO SCANNER</span><div class="live-dot cyan"></div></div>
      <div class="now-playing">
        <div class="np-icon" id="np-icon">📡</div>
        <div class="np-info">
          <div class="np-station" id="np-name">SELECT A STATION</div>
          <div class="np-freq" id="np-freq">— MHz</div>
          <div class="np-tag" id="np-tags">EMERGENCY · NEWS · SCANNER</div>
        </div>
        <div class="np-controls">
          <button class="radio-btn" id="prev-btn" aria-label="Previous">⏮</button>
          <button class="radio-btn play-btn" id="play-btn" aria-label="Play">▶</button>
          <button class="radio-btn" id="next-btn" aria-label="Next">⏭</button>
        </div>
      </div>
      <div class="vol-bar">
        <label for="vol-slider">VOL</label>
        <input type="range" id="vol-slider" min="0" max="1" step="0.01" value="0.7">
        <label id="vol-pct" style="width:28px;text-align:right">70%</label>
      </div>
      <div class="freq-viz" id="freq-viz"></div>
      <div class="station-list-wrap"><div id="station-list"><div class="loading-msg">SCANNING FREQUENCIES…</div></div></div>
      <div class="uptime-bar">
        <span style="width:6px;height:6px;border-radius:50%;background:var(--green);box-shadow:0 0 6px var(--green);animation:blink 2s infinite;display:inline-block;flex-shrink:0"></span>
        UPTIME: <b id="uptime-display">00:00:00</b>
        &nbsp;·&nbsp; KA: <b id="keepalive-status" style="color:var(--green)">ACTIVE</b>
        &nbsp;·&nbsp; 40HR MODE
      </div>
    </div>

    <div class="panel" style="display:flex;flex-direction:column">
      <div class="panel-head" style="color:var(--red);border-bottom-color:rgba(255,51,102,.25)"><span>⚡ EMERGENCY FREQUENCIES</span><div class="live-dot red"></div></div>
      <div class="freq-ref-grid">
        <div class="freq-ref-cell"><div class="freq-ref-label">CIVIL AIR DISTRESS</div><div class="freq-ref-val" style="color:var(--red)">121.5 <span style="font-size:9px;color:var(--dim)">MHz</span></div><div class="freq-ref-note">Int'l Mayday</div></div>
        <div class="freq-ref-cell"><div class="freq-ref-label">MARITIME VHF CH16</div><div class="freq-ref-val" style="color:var(--amber)">156.8 <span style="font-size:9px;color:var(--dim)">MHz</span></div><div class="freq-ref-note">Coast Guard</div></div>
        <div class="freq-ref-cell"><div class="freq-ref-label">NOAA WEATHER</div><div class="freq-ref-val" style="color:var(--cyan)">162.4 <span style="font-size:9px;color:var(--dim)">MHz</span></div><div class="freq-ref-note">NWS Alerts</div></div>
        <div class="freq-ref-cell"><div class="freq-ref-label">MIL AIR COMMAND</div><div class="freq-ref-val" style="color:var(--purple)">243.0 <span style="font-size:9px;color:var(--dim)">MHz</span></div><div class="freq-ref-note">NATO Guard</div></div>
        <div class="freq-ref-cell"><div class="freq-ref-label">POLICE/FIRE</div><div class="freq-ref-val" style="color:var(--green)">155.3 <span style="font-size:9px;color:var(--dim)">MHz</span></div><div class="freq-ref-note">Public Safety</div></div>
        <div class="freq-ref-cell"><div class="freq-ref-label">EAS PRIMARY AM</div><div class="freq-ref-val" style="color:var(--bright)">640 <span style="font-size:9px;color:var(--dim)">kHz</span></div><div class="freq-ref-note">Broadcast Entry</div></div>
      </div>

      <div class="panel-head" style="color:var(--amber);border-bottom-color:rgba(255,170,0,.2);border-top:1px solid var(--border)">
        <span>🌩 LOCAL NWS ALERTS</span><div class="live-dot amber"></div>
      </div>
      <div id="nws-alert-list" style="flex:1;overflow-y:auto;-webkit-overflow-scrolling:touch;max-height:200px">
        <div class="loading-msg">FETCHING LOCAL ALERTS…</div>
      </div>
      <audio id="radio-audio" preload="none" crossorigin="anonymous" style="display:none"></audio>
    </div>
  </div>

  <div class="footer">
    OPEN NOTIFY · OPENSKY NETWORK · NWS WEATHER · RADIO BROWSER &nbsp;·&nbsp;
    <span style="color:var(--cyan);letter-spacing:2px">@MrCodeBlood</span> &nbsp;·&nbsp;
    ORBIT WATCH v4.0 © 2025 &nbsp;·&nbsp; PWA · 40HR MODE
  </div>

</div><!-- /app -->

<script>
'use strict';
// ══════════════════════════════════════════════════════
//  SERVICE WORKER — Offline cache via Blob URL
// ══════════════════════════════════════════════════════
if('serviceWorker' in navigator){
  const swCode=`
const CACHE='orbitwatch-v4';
const PRECACHE=[
  'https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Share+Tech+Mono&display=swap',
  'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js',
  'https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js',
  'https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js',
  'https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json'
];
self.addEventListener('install',e=>{
  e.waitUntil(caches.open(CACHE).then(c=>c.addAll(PRECACHE).catch(()=>{})));
  self.skipWaiting();
});
self.addEventListener('activate',e=>{
  e.waitUntil(caches.keys().then(ks=>Promise.all(ks.filter(k=>k!==CACHE).map(k=>caches.delete(k)))));
  self.clients.claim();
});
self.addEventListener('fetch',e=>{
  const url=new URL(e.request.url);
  // Cache-first for static assets
  if(['fonts.googleapis.com','fonts.gstatic.com','cdnjs.cloudflare.com','cdn.jsdelivr.net'].some(h=>url.hostname.includes(h))){
    e.respondWith(caches.match(e.request).then(r=>r||fetch(e.request).then(nr=>{caches.open(CACHE).then(c=>c.put(e.request,nr.clone()));return nr;})));
    return;
  }
  // Network-first for APIs
  if(['api.open-notify.org','opensky-network.org','api.weather.gov','radio-browser.info'].some(h=>url.hostname.includes(h))){
    e.respondWith(fetch(e.request).catch(()=>caches.match(e.request)));
    return;
  }
});`;
  const blob=new Blob([swCode],{type:'application/javascript'});
  navigator.serviceWorker.register(URL.createObjectURL(blob)).catch(()=>{});
}

// ══════════════════════════════════════════════════════
//  CONFIG & STATE
// ══════════════════════════════════════════════════════
const CFG={REFRESH_MS:5000,ALERT_KM:1500,EARTH_R:6371,ISS_ALT:408};
const STATE={
  iss:{lat:0,lon:0,ts:0},
  user:{lat:null,lon:null},
  issNearby:false,
  map:{ready:false,proj:null,path:null,svg:null,issG:null,userG:null,orbitPath:null},
};

// ══════════════════════════════════════════════════════
//  UTILITIES
// ══════════════════════════════════════════════════════
const el=id=>document.getElementById(id);
function haversine(la1,lo1,la2,lo2){
  const R=CFG.EARTH_R,r=Math.PI/180;
  const dLa=(la2-la1)*r,dLo=(lo2-lo1)*r;
  const a=Math.sin(dLa/2)**2+Math.cos(la1*r)*Math.cos(la2*r)*Math.sin(dLo/2)**2;
  return R*2*Math.atan2(Math.sqrt(a),Math.sqrt(1-a));
}
const latStr=v=>Math.abs(v).toFixed(3)+'° '+(v>=0?'N':'S');
const lonStr=v=>Math.abs(v).toFixed(3)+'° '+(v>=0?'E':'W');

// ══════════════════════════════════════════════════════
//  OPEN NOTIFY API
// ══════════════════════════════════════════════════════
const API={
  BASE:'https://api.open-notify.org',
  async _get(ep){
    try{const r=await fetch(this.BASE+ep,{signal:AbortSignal.timeout(10000)});if(!r.ok)throw 0;return r.json();}
    catch{const r=await fetch('https://api.allorigins.win/get?url='+encodeURIComponent(this.BASE+ep));const w=await r.json();return JSON.parse(w.contents);}
  },
  async issPosition(){const d=await this._get('/iss-now.json');return{lat:+d.iss_position.latitude,lon:+d.iss_position.longitude,ts:d.timestamp};},
  async astronauts(){const d=await this._get('/astros.json');return{count:d.number,people:d.people};},
};

// ══════════════════════════════════════════════════════
//  OPENSKY API
// ══════════════════════════════════════════════════════
const SKYAPI={
  BASE:'https://opensky-network.org/api',
  async _get(ep){
    const url=this.BASE+ep;
    const proxies=['','https://api.allorigins.win/get?url=','https://corsproxy.io/?'];
    for(const p of proxies){
      try{
        const r=await fetch(p?p+encodeURIComponent(url):url,{signal:AbortSignal.timeout(15000)});
        if(!r.ok)continue;
        if(p==='https://api.allorigins.win/get?url='){const w=await r.json();return JSON.parse(w.contents);}
        return r.json();
      }catch{}
    }
    return null;
  },
  async allStates(){
    const d=await this._get('/states/all');
    if(!d?.states)return[];
    return d.states.map(s=>({
      icao24:s[0],callsign:(s[1]||'').trim()||null,country:s[2],
      lon:s[5],lat:s[6],alt:s[7],onGround:s[8],velocity:s[9],
      track:s[10],vertRate:s[11],geoAlt:s[13],squawk:s[14],spi:s[15],
    }));
  },
};

// ══════════════════════════════════════════════════════
//  SOUND SYSTEM
// ══════════════════════════════════════════════════════
const SOUND={
  ctx:null,muted:false,
  _ctx(){
    if(!this.ctx)this.ctx=new(window.AudioContext||window.webkitAudioContext)();
    if(this.ctx.state==='suspended')this.ctx.resume();
    return this.ctx;
  },
  _tone(f,type,g,dur,t){
    const ctx=this._ctx(),o=ctx.createOscillator(),gn=ctx.createGain();
    o.connect(gn);gn.connect(ctx.destination);
    o.type=type;o.frequency.setValueAtTime(f,t);
    gn.gain.setValueAtTime(g,t);gn.gain.exponentialRampToValueAtTime(.001,t+dur);
    o.start(t);o.stop(t+dur);
  },
  critical(){
    if(this.muted)return;
    const ctx=this._ctx(),now=ctx.currentTime;
    for(let i=0;i<4;i++){
      const t=now+i*.35,o=ctx.createOscillator(),gn=ctx.createGain();
      o.connect(gn);gn.connect(ctx.destination);o.type='sawtooth';
      o.frequency.setValueAtTime(440,t);o.frequency.linearRampToValueAtTime(1100,t+.28);
      gn.gain.setValueAtTime(.28,t);gn.gain.exponentialRampToValueAtTime(.001,t+.3);
      o.start(t);o.stop(t+.31);
    }
  },
  high(){if(this.muted)return;const ctx=this._ctx(),now=ctx.currentTime;[880,1100,880].forEach((f,i)=>this._tone(f,'square',.15,.18,now+i*.2));},
  medium(){if(this.muted)return;const ctx=this._ctx(),now=ctx.currentTime;[660,660].forEach((f,i)=>this._tone(f,'sine',.12,.14,now+i*.22));},
  low(){if(this.muted)return;this._tone(520,'sine',.08,.12,this._ctx().currentTime);},
  ebs(){
    if(this.muted)return;
    const ctx=this._ctx(),now=ctx.currentTime;
    [now,now+1].forEach(t=>{this._tone(853,'sine',.2,.9,t);this._tone(960,'sine',.2,.9,t);});
  },
  iss(){
    if(this.muted)return;
    const ctx=this._ctx(),now=ctx.currentTime;
    [[1047,.18,.6,0],[1319,.14,.5,.25],[1568,.10,.4,.5]].forEach(([f,g,d,dt])=>this._tone(f,'sine',g,d,now+dt));
  },
  init(){
    const btn=el('sound-toggle');
    btn.addEventListener('click',()=>{
      this.muted=!this.muted;
      btn.textContent=this.muted?'🔇':'🔊';
      btn.id='sound-toggle';
      btn.className=this.muted?'muted':'';
      if(!this.muted)this.low();
    });
    document.addEventListener('click',()=>this._ctx(),{once:true});
  },
};

// ══════════════════════════════════════════════════════
//  EBS TICKER
// ══════════════════════════════════════════════════════
const EBS={
  alerts:[],current:0,timer:null,dismissed:false,
  SEV:{
    Extreme:{cls:'ebs-alert',label:'⚡ EXTREME'},
    Severe:{cls:'ebs-alert',label:'🚨 SEVERE'},
    Moderate:{cls:'ebs-warn',label:'⚠ WARNING'},
    Minor:{cls:'ebs-info',label:'ℹ ADVISORY'},
    Unknown:{cls:'ebs-info',label:'📡 BROADCAST'},
  },
  _set(txt,ms){
    const t=el('ebs-ticker-text');if(!t)return;
    t.textContent=txt;t.style.animationDuration=Math.max(12000,ms)+'ms';
  },
  async fetch(lat,lon){
    try{
      const r=await fetch(`https://api.weather.gov/alerts/active?point=${lat.toFixed(4)},${lon.toFixed(4)}`,
        {headers:{'User-Agent':'OrbitWatch/4.0 (@MrCodeBlood)'},signal:AbortSignal.timeout(10000)});
      if(!r.ok)throw 0;
      const d=await r.json();
      this.alerts=(d.features||[]).map(f=>({
        headline:f.properties.headline||f.properties.event||'Alert',
        severity:f.properties.severity||'Unknown',
        area:f.properties.areaDesc||'Area',
        desc:(f.properties.description||'').slice(0,160),
      }));
      this._renderList();
      if(this.alerts.length){
        this.show();
        const s=this.alerts[0].severity;
        if(s==='Extreme'||s==='Severe'){SOUND.ebs();setTimeout(()=>SOUND.critical(),2200);}
        else SOUND.medium();
      }else this._showClear();
    }catch{this._showFallback();this._renderList();}
  },
  show(){if(!el('ebs-bar')||this.dismissed)return;this.current=0;this._rotate();el('ebs-bar').classList.add('visible');document.body.classList.add('ebs-active');},
  _showClear(){
    if(!el('ebs-bar'))return;
    el('ebs-bar').className='ebs-info visible';el('ebs-type').textContent='✓ CLEAR';
    this._set('  ✓ NO ACTIVE ALERTS IN YOUR AREA — @MrCodeBlood  ',18000);
    document.body.classList.add('ebs-active');
  },
  _showFallback(){
    if(!el('ebs-bar')||this.dismissed)return;
    el('ebs-bar').className='ebs-info visible';el('ebs-type').textContent='📡 ORBIT WATCH';
    const msgs=[
      'ORBIT WATCH v4.0 — BY @MrCodeBlood — ISS LIVE TRACKING · AIRSPACE ANOMALY MONITOR · EMERGENCY RADIO SCANNER — ALL SYSTEMS ONLINE',
      'SQUAWK MONITORING: 7500 HIJACK · 7600 RADIO FAIL · 7700 EMERGENCY — OPENSKY NETWORK LIVE · 40-HOUR OPERATION MODE',
      'NOAA WEATHER ALERTS · NWS EMERGENCY BROADCAST · RADIO BROWSER API — PWA INSTALLABLE — ADD TO HOME SCREEN',
    ];
    let i=0;
    const roll=()=>this._set('  '+msgs[i++%msgs.length]+'  ',msgs[(i-1)%msgs.length].length*80);
    roll();clearInterval(this.timer);this.timer=setInterval(roll,20000);
    document.body.classList.add('ebs-active');el('ebs-bar').classList.add('visible');
  },
  _rotate(){
    if(!this.alerts.length)return;
    const a=this.alerts[this.current%this.alerts.length];
    const sev=this.SEV[a.severity]||this.SEV.Unknown;
    el('ebs-bar').className=sev.cls+' visible';el('ebs-type').textContent=sev.label;
    const txt=`  ⚡ ${a.headline}  ·  ${a.area}  ·  ${a.desc}  `;
    this._set(txt,txt.length*75);
    this.current=(this.current+1)%this.alerts.length;
    clearInterval(this.timer);this.timer=setInterval(()=>this._rotate(),Math.max(15000,txt.length*75));
  },
  _renderList(){
    const list=el('nws-alert-list');if(!list)return;
    if(!this.alerts.length){list.innerHTML='<div class="loading-msg" style="color:var(--green)">✓ NO ACTIVE ALERTS IN YOUR AREA</div>';return;}
    list.innerHTML=this.alerts.map(a=>{
      const s=this.SEV[a.severity]||this.SEV.Unknown;
      const c=s.cls==='ebs-alert'?'var(--red)':s.cls==='ebs-warn'?'var(--amber)':'var(--cyan2)';
      return`<div style="padding:9px 12px;border-bottom:1px solid rgba(11,61,106,.4)">
        <div style="font-size:8px;letter-spacing:2px;color:${c};margin-bottom:2px">${s.label} · ${a.severity.toUpperCase()}</div>
        <div style="font-size:11px;color:var(--bright)">${a.headline}</div>
        <div style="font-size:8px;color:var(--dim);margin-top:2px">${a.area}</div>
      </div>`;
    }).join('');
  },
  init(){
    el('ebs-dismiss').addEventListener('click',()=>{
      this.dismissed=true;el('ebs-bar').classList.remove('visible');
      document.body.classList.remove('ebs-active');clearInterval(this.timer);
    });
    this._showFallback();
  },
};

// ══════════════════════════════════════════════════════
//  RADIO SCANNER
// ══════════════════════════════════════════════════════
const RADIO={
  stations:[],idx:-1,playing:false,vizBars:[],vizTimer:null,
  KW:['emergency','alert','weather','noaa','news','scanner','police','fire','eas'],
  async fetch(){
    const res=await Promise.allSettled(['news','emergency','scanner','weather'].map(t=>this._tag(t)));
    const seen=new Set(),all=[];
    res.forEach(r=>{if(r.status==='fulfilled'&&Array.isArray(r.value))r.value.forEach(s=>{if(!seen.has(s.stationuuid)&&s.url_resolved){seen.add(s.stationuuid);all.push(s);}});});
    all.sort((a,b)=>(this._em(b)?1:0)-(this._em(a)?1:0)||(b.votes||0)-(a.votes||0));
    this.stations=all.slice(0,60);this._render();
  },
  async _tag(tag){
    for(const srv of['https://de1.api.radio-browser.info','https://nl1.api.radio-browser.info','https://at1.api.radio-browser.info']){
      try{const r=await fetch(`${srv}/json/stations/search?tag=${encodeURIComponent(tag)}&limit=30&hidebroken=true&order=votes&reverse=true`,{signal:AbortSignal.timeout(8000)});if(r.ok)return r.json();}catch{}
    }
    return[];
  },
  _em(s){const t=((s.name||'')+' '+(s.tags||'')).toLowerCase();return this.KW.some(k=>t.includes(k));},
  _freq(s){const h=[...(s.name||'')].reduce((a,c)=>a+c.charCodeAt(0),0);return h%5===0?((h%136)*10+540).toLocaleString()+' kHz AM':(88.1+(h%900)/10).toFixed(1)+' MHz FM';},
  _render(){
    const list=el('station-list');
    if(!this.stations.length){list.innerHTML='<div class="loading-msg">NO STATIONS — CHECK CONNECTION</div>';return;}
    list.innerHTML=this.stations.map((s,i)=>{
      const em=this._em(s);
      return`<div class="station-item${em?' emergency':''}${i===this.idx?' active':''}" onclick="RADIO.select(${i})">
        <div class="si-icon">${em?'⚡':'📻'}</div>
        <div class="si-info">
          <div class="si-name">${s.name||'Unknown'}${em?'<span class="emerg-badge">EMERG</span>':''}</div>
          <div class="si-meta">${s.country||''} · ${(s.tags||'').split(',').slice(0,3).join(' · ')}</div>
        </div>
        <div class="si-freq${em?' red':''}">${this._freq(s)}</div>
      </div>`;
    }).join('');
  },
  select(i){
    this.idx=i;const s=this.stations[i];if(!s)return;
    el('np-name').textContent=s.name||'Unknown';
    el('np-freq').textContent=this._freq(s);
    el('np-tags').textContent=(s.tags||'news').split(',').slice(0,4).join(' · ').toUpperCase();
    el('np-icon').textContent=this._em(s)?'⚡':'📡';
    document.querySelectorAll('.station-item').forEach((e,j)=>e.classList.toggle('active',j===i));
    this.play();
  },
  play(){
    const s=this.stations[this.idx];if(!s)return;
    const a=el('radio-audio');a.src=s.url_resolved;a.volume=+el('vol-slider').value;
    a.play().then(()=>{this.playing=true;el('play-btn').textContent='⏸';el('play-btn').classList.add('active');el('np-icon').classList.add('playing');this._startViz();})
           .catch(()=>{el('np-name').textContent='⚠ STREAM UNAVAILABLE';el('play-btn').classList.remove('active');});
  },
  pause(){el('radio-audio').pause();this.playing=false;el('play-btn').textContent='▶';el('play-btn').classList.remove('active');el('np-icon').classList.remove('playing');this._stopViz();},
  toggle(){if(this.playing){this.pause();return;}if(this.idx<0&&this.stations.length){this.select(0);return;}this.play();},
  prev(){this.select(((this.idx||0)-1+this.stations.length)%this.stations.length);},
  next(){this.select(((this.idx||0)+1)%this.stations.length);},
  _startViz(){
    clearInterval(this.vizTimer);
    this.vizTimer=setInterval(()=>{
      this.vizBars.forEach(b=>{
        const h=Math.max(2,Math.random()*24);
        b.style.height=h+'px';b.style.opacity=.3+(h/24)*.7;
        b.style.background=h>17?'var(--red)':h>10?'var(--amber)':'var(--cyan)';
      });
    },80);
  },
  _stopViz(){clearInterval(this.vizTimer);this.vizBars.forEach(b=>{b.style.height='2px';b.style.opacity='.15';b.style.background='var(--cyan)';});},
  init(){
    el('play-btn').addEventListener('click',()=>this.toggle());
    el('prev-btn').addEventListener('click',()=>this.prev());
    el('next-btn').addEventListener('click',()=>this.next());
    el('vol-slider').addEventListener('input',e=>{const v=+e.target.value;el('radio-audio').volume=v;el('vol-pct').textContent=Math.round(v*100)+'%';});
    el('radio-audio').addEventListener('error',()=>{if(this.playing)setTimeout(()=>this.next(),2000);});
    el('radio-audio').addEventListener('ended',()=>this.next());
    const wrap=el('freq-viz');
    for(let i=0;i<26;i++){const b=document.createElement('div');b.className='freq-bar';b.style.height='2px';b.style.opacity='.15';wrap.appendChild(b);this.vizBars.push(b);}
    this.fetch();
  },
};

// ══════════════════════════════════════════════════════
//  ANOMALY ENGINE
// ══════════════════════════════════════════════════════
const ANOMALY={
  events:[],max:60,counts:{critical:0,high:0,medium:0,low:0},radarDots:[],scanned:0,
  SQUAWKS:{
    '7500':{label:'SQUAWK 7500 — HIJACK',sev:'critical',sevClass:'sev-critical',typeClass:''},
    '7600':{label:'SQUAWK 7600 — RADIO FAIL',sev:'high',sevClass:'sev-high',typeClass:'high'},
    '7700':{label:'SQUAWK 7700 — EMERGENCY',sev:'high',sevClass:'sev-high',typeClass:'high'},
  },
  classify(ac){
    const f=[];
    if(ac.squawk&&this.SQUAWKS[ac.squawk])f.push({...this.SQUAWKS[ac.squawk],squawk:ac.squawk});
    if(!ac.onGround&&ac.vertRate<-15&&ac.alt>300&&ac.alt<12000){
      const fpm=Math.round(ac.vertRate*196.85);
      f.push({label:`RAPID DESCENT — ${fpm.toLocaleString()} ft/min`,sev:'medium',sevClass:'sev-medium',typeClass:'medium',squawk:null});
    }
    if(!ac.onGround&&ac.velocity>350&&ac.alt<3000)
      f.push({label:`EXTREME SPEED — ${Math.round(ac.velocity*1.944)} kts low alt`,sev:'low',sevClass:'sev-low',typeClass:'low',squawk:null});
    if(ac.spi&&!ac.onGround)
      f.push({label:'SPI IDENT ACTIVE',sev:'low',sevClass:'sev-low',typeClass:'low',squawk:null});
    return f;
  },
  ingest(aircraft){
    const timeStr=new Date().toUTCString().slice(17,25);
    this.scanned=aircraft.length;this.counts={critical:0,high:0,medium:0,low:0};this.radarDots=[];
    const newEvts=[],seen=new Set(this.events.slice(0,20).map(e=>e.key));
    aircraft.forEach(ac=>{
      this.classify(ac).forEach(flag=>{
        this.counts[flag.sev]++;
        const key=(ac.icao24||'')+flag.sev+(flag.squawk||'x');
        if(!seen.has(key)){newEvts.push({...flag,ac,time:timeStr,key});seen.add(key);}
        const ang=Math.random()*Math.PI*2,dist=Math.random()*.75+.15;
        this.radarDots.push({x:.5+dist*Math.cos(ang)*.5,y:.5+dist*Math.sin(ang)*.5,sev:flag.sev,phase:Math.random()*Math.PI*2});
      });
    });
    this.events=[...newEvts,...this.events].slice(0,this.max);
    return newEvts.length;
  },
  render(){
    el('cnt-critical').textContent=this.counts.critical;
    el('cnt-high').textContent=this.counts.high;
    el('cnt-medium').textContent=this.counts.medium;
    el('cnt-scanned').textContent=this.scanned.toLocaleString();
    el('feed-count').textContent=this.events.length+' EVENTS';
    const total=this.counts.critical+this.counts.high+this.counts.medium;
    const badge=el('anomaly-badge');
    badge.style.display=total>0?'inline-flex':'none';
    if(total>0)badge.textContent=total>99?'99+':total;
    const feed=el('anomaly-feed');
    if(!this.events.length){feed.innerHTML='<div class="loading-msg" style="color:var(--green)">✓ AIRSPACE CLEAR</div>';return;}
    feed.innerHTML=this.events.map(ev=>{
      const ac=ev.ac,cs=ac.callsign||ac.icao24||'UNKNOWN';
      const alt=ac.alt?Math.round(ac.alt*3.281).toLocaleString()+' ft':'—';
      const spd=ac.velocity?Math.round(ac.velocity*1.944).toLocaleString()+' kts':'—';
      const sq=ev.squawk?`<span class="squawk-badge sq-${ev.squawk}">${ev.squawk}</span>`:'';
      return`<div class="a-event">
        <div class="a-sev ${ev.sevClass}"></div>
        <div>
          <div class="a-type ${ev.typeClass}">${ev.label}</div>
          <div class="a-flight">${cs}${sq}</div>
          <div class="a-detail">${ac.country||'UNKNOWN'} · ${alt} · ${spd}${ac.lat&&ac.lon?` · ${ac.lat.toFixed(2)}°,${ac.lon.toFixed(2)}°`:''}</div>
        </div>
        <div class="a-time">${ev.time}</div>
      </div>`;
    }).join('');
  },
};

// ══════════════════════════════════════════════════════
//  RADAR ANIMATION
// ══════════════════════════════════════════════════════
function initRadar(){
  const canvas=el('radar-canvas'),ctx=canvas.getContext('2d');
  const W=canvas.width,H=canvas.height,cx=W/2,cy=H/2,R=W/2-6;
  let angle=0;
  const SEV_COLORS={critical:'#ff3366',high:'#ff6600',medium:'#ffaa00',low:'#00e5ff'};
  const trail=document.createElement('canvas');trail.width=W;trail.height=H;
  const tc=trail.getContext('2d');
  (function tick(){
    requestAnimationFrame(tick);angle+=.025;
    ctx.clearRect(0,0,W,H);
    ctx.fillStyle='#030e08';ctx.beginPath();ctx.arc(cx,cy,R,0,Math.PI*2);ctx.fill();
    tc.fillStyle='rgba(3,14,8,.04)';tc.fillRect(0,0,W,H);
    tc.save();tc.translate(cx,cy);tc.rotate(angle);
    const g=tc.createLinearGradient(0,0,R,0);
    g.addColorStop(0,'rgba(0,255,80,.35)');g.addColorStop(1,'rgba(0,255,80,0)');
    tc.beginPath();tc.moveTo(0,0);tc.arc(0,0,R,-.3,0);tc.closePath();tc.fillStyle=g;tc.fill();tc.restore();
    ctx.drawImage(trail,0,0);
    [.25,.5,.75,1].forEach(f=>{ctx.beginPath();ctx.arc(cx,cy,R*f,0,Math.PI*2);ctx.strokeStyle=`rgba(0,229,255,${f===1?.25:.1})`;ctx.lineWidth=.7;ctx.stroke();});
    ctx.strokeStyle='rgba(0,229,255,.12)';ctx.lineWidth=.5;
    ctx.beginPath();ctx.moveTo(cx-R,cy);ctx.lineTo(cx+R,cy);ctx.stroke();
    ctx.beginPath();ctx.moveTo(cx,cy-R);ctx.lineTo(cx,cy+R);ctx.stroke();
    ctx.save();ctx.translate(cx,cy);ctx.rotate(angle);
    ctx.beginPath();ctx.moveTo(0,0);ctx.lineTo(R,0);ctx.strokeStyle='rgba(0,255,80,.8)';ctx.lineWidth=1.2;ctx.stroke();ctx.restore();
    ANOMALY.radarDots.forEach(d=>{
      d.phase+=.04;const x=d.x*W,y=d.y*H,col=SEV_COLORS[d.sev]||'#00e5ff';
      const p=.5+.5*Math.sin(d.phase);
      ctx.beginPath();ctx.arc(x,y,2+p*1.5,0,Math.PI*2);ctx.fillStyle=col;ctx.fill();
    });
    ctx.beginPath();ctx.arc(cx,cy,3,0,Math.PI*2);ctx.fillStyle='#0f5';ctx.fill();
  })();
}

// ══════════════════════════════════════════════════════
//  STARS
// ══════════════════════════════════════════════════════
function initStars(){
  const c=el('stars-canvas'),ctx=c.getContext('2d');
  const resize=()=>{c.width=innerWidth;c.height=innerHeight;};
  resize();window.addEventListener('resize',resize);
  const stars=Array.from({length:240},()=>({x:Math.random(),y:Math.random(),r:Math.random()*1.3+.2,ph:Math.random()*Math.PI*2,sp:Math.random()*.008+.002}));
  (function tick(){
    ctx.clearRect(0,0,c.width,c.height);
    stars.forEach(s=>{s.ph+=s.sp;ctx.beginPath();ctx.arc(s.x*c.width,s.y*c.height,s.r,0,Math.PI*2);ctx.fillStyle=`rgba(180,220,255,${.2+.6*Math.abs(Math.sin(s.ph))})`;ctx.fill();});
    requestAnimationFrame(tick);
  })();
}

// ══════════════════════════════════════════════════════
//  MAP
// ══════════════════════════════════════════════════════
async function initMap(){
  const wrap=el('map-svg-wrap');
  const W=wrap.clientWidth||window.innerWidth-32;
  const H=Math.max(220,wrap.clientHeight||300);
  const svg=d3.select('#world-svg').attr('viewBox',`0 0 ${W} ${H}`);
  STATE.map.svg=svg;
  const proj=d3.geoEquirectangular().scale(W/(2*Math.PI)).translate([W/2,H/2]);
  const path=d3.geoPath().projection(proj);
  STATE.map.proj=proj;STATE.map.path=path;
  svg.append('rect').attr('width',W).attr('height',H).attr('fill','#020912');
  svg.append('path').datum(d3.geoGraticule()()).attr('class','graticule').attr('d',path);
  try{
    const world=await d3.json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
    svg.append('g').selectAll('path').data(topojson.feature(world,world.objects.countries).features).enter().append('path').attr('class','land').attr('d',path);
  }catch{}
  [{lat:0},{lat:23.5},{lat:-23.5}].forEach(({lat},i)=>{
    const cls=i===0?'equator':'tropic';
    svg.append('path').datum({type:'Feature',geometry:{type:'LineString',coordinates:d3.range(-180,181,2).map(x=>[x,lat])}}).attr('class',cls).attr('d',path);
  });
  STATE.map.orbitPath=svg.append('path').attr('class','orbit-track');
  const ug=svg.append('g').attr('opacity',0);
  ug.append('circle').attr('r',8).attr('fill','rgba(255,170,0,.15)').attr('stroke','#fa0').attr('stroke-width',1.5);
  ug.append('circle').attr('r',3).attr('fill','#fa0');
  ug.append('text').attr('x',10).attr('y',-8).attr('fill','#fa0').attr('font-family','Share Tech Mono,monospace').attr('font-size','9px').text('YOU');
  STATE.map.userG=ug;
  const ig=svg.append('g');
  ig.append('circle').attr('r',16).attr('fill','none').attr('stroke','rgba(0,229,255,.15)').attr('stroke-width',1.5).style('animation','pulse 2s infinite');
  ig.append('circle').attr('r',6).attr('fill','rgba(0,229,255,.2)').attr('stroke','var(--cyan)').attr('stroke-width',1);
  ig.append('circle').attr('r',2.5).attr('fill','#00e5ff');
  [[-13,0,13,0],[0,-13,0,13]].forEach(([x1,y1,x2,y2])=>ig.append('line').attr('x1',x1).attr('y1',y1).attr('x2',x2).attr('y2',y2).attr('stroke','rgba(0,229,255,.35)').attr('stroke-width',.8));
  ig.append('text').attr('x',11).attr('y',-8).attr('fill','#00e5ff').attr('font-family','Share Tech Mono,monospace').attr('font-size','9px').attr('letter-spacing','1px').text('ISS');
  STATE.map.issG=ig;
  STATE.map.ready=true;
  if(STATE.user.lat!==null)_placeUser();
}
function _placeUser(){
  if(!STATE.map.ready||STATE.user.lat===null)return;
  const[x,y]=STATE.map.proj([STATE.user.lon,STATE.user.lat]);
  STATE.map.userG.attr('transform',`translate(${x},${y})`).attr('opacity',1);
}
function updateMapISS(lat,lon){
  if(!STATE.map.ready)return;
  const[x,y]=STATE.map.proj([lon,lat]);
  STATE.map.issG.attr('transform',`translate(${x},${y})`);
  const inc=51.6*Math.PI/180,pts=[];
  for(let t=-50;t<=50;t++){
    const ang=t*Math.PI/180*4.5,oLat=Math.asin(Math.sin(inc)*Math.sin(ang))*180/Math.PI;
    let oLon=lon+t*4.5;while(oLon>180)oLon-=360;while(oLon<-180)oLon+=360;
    pts.push([oLon,oLat]);
  }
  try{STATE.map.orbitPath.datum({type:'Feature',geometry:{type:'LineString',coordinates:pts}}).attr('d',STATE.map.path);}catch{}
}

// ══════════════════════════════════════════════════════
//  3D GLOBE
// ══════════════════════════════════════════════════════
function init3D(){
  const wrap=el('three-wrap'),canvas=el('three-canvas');
  let W=wrap.clientWidth,H=wrap.clientHeight;
  const renderer=new THREE.WebGLRenderer({canvas,antialias:true,alpha:true,powerPreference:'low-power'});
  renderer.setSize(W,H);renderer.setPixelRatio(Math.min(devicePixelRatio,2));
  const scene=new THREE.Scene();
  const cam=new THREE.PerspectiveCamera(45,W/H,.1,500);
  cam.position.set(0,0,3.8);
  new ResizeObserver(()=>{W=wrap.clientWidth;H=wrap.clientHeight;renderer.setSize(W,H);cam.aspect=W/H;cam.updateProjectionMatrix();}).observe(wrap);

  // Stars
  const sv=[];for(let i=0;i<3000;i++){const t=Math.random()*Math.PI*2,p=Math.acos(2*Math.random()-1),r=60+Math.random()*40;sv.push(r*Math.sin(p)*Math.cos(t),r*Math.sin(p)*Math.sin(t),r*Math.cos(p));}
  const sg=new THREE.BufferGeometry();sg.setAttribute('position',new THREE.Float32BufferAttribute(sv,3));
  scene.add(new THREE.Points(sg,new THREE.PointsMaterial({color:0xffffff,size:.06,transparent:true,opacity:.8})));

  // Earth texture (procedural)
  const tc=document.createElement('canvas');tc.width=1024;tc.height=512;
  const tx=tc.getContext('2d');
  const og=tx.createLinearGradient(0,0,0,512);og.addColorStop(0,'#020d1a');og.addColorStop(1,'#030f20');
  tx.fillStyle=og;tx.fillRect(0,0,1024,512);
  tx.strokeStyle='rgba(0,229,255,.04)';tx.lineWidth=.5;
  for(let i=0;i<=8;i++){tx.beginPath();tx.moveTo(0,i*64);tx.lineTo(1024,i*64);tx.stroke();}
  for(let i=0;i<=16;i++){tx.beginPath();tx.moveTo(i*64,0);tx.lineTo(i*64,512);tx.stroke();}
  tx.fillStyle='#081f10';
  [{cx:180,cy:165,rx:80,ry:85,rot:-.15},{cx:240,cy:300,rx:48,ry:90,rot:.12},{cx:510,cy:150,rx:52,ry:55,rot:0},{cx:515,cy:280,rx:58,ry:100,rot:.08},{cx:710,cy:155,rx:160,ry:90,rot:0},{cx:800,cy:340,rx:52,ry:38,rot:.1},{cx:512,cy:490,rx:200,ry:22,rot:0},{cx:290,cy:90,rx:35,ry:28,rot:0}]
    .forEach(({cx,cy,rx,ry,rot})=>{tx.save();tx.translate(cx,cy);tx.rotate(rot);tx.beginPath();tx.ellipse(0,0,rx,ry,0,0,Math.PI*2);tx.fill();tx.restore();});
  tx.strokeStyle='rgba(0,229,255,.15)';tx.lineWidth=1.5;tx.beginPath();tx.moveTo(0,256);tx.lineTo(1024,256);tx.stroke();

  const earth=new THREE.Mesh(
    new THREE.SphereGeometry(1,48,48),
    new THREE.MeshPhongMaterial({map:new THREE.CanvasTexture(tc),color:0x0a3f6b,emissive:0x001a33,specular:0x224466,shininess:12})
  );
  scene.add(earth);
  [[1.02,0x0055ff,.06],[1.05,0x003399,.04],[1.08,0x0022aa,.025]].forEach(([r,c,o])=>scene.add(new THREE.Mesh(new THREE.SphereGeometry(r,24,24),new THREE.MeshPhongMaterial({color:c,transparent:true,opacity:o,side:THREE.FrontSide,depthWrite:false}))));

  const ORB_R=(CFG.EARTH_R+CFG.ISS_ALT)/CFG.EARTH_R;
  const orbitRing=new THREE.Mesh(new THREE.TorusGeometry(ORB_R,.0025,8,160),new THREE.MeshBasicMaterial({color:0x00e5ff,transparent:true,opacity:.35}));
  orbitRing.rotation.x=Math.PI/2;
  const orbitPlane=new THREE.Group();orbitPlane.rotation.z=51.6*Math.PI/180;
  orbitPlane.add(orbitRing);scene.add(orbitPlane);

  const issG=new THREE.Group();scene.add(issG);
  issG.add(new THREE.Mesh(new THREE.BoxGeometry(.04,.012,.012),new THREE.MeshPhongMaterial({color:0xccddee,emissive:0x002244,shininess:60})));
  const pm=new THREE.MeshPhongMaterial({color:0x00aaff,emissive:0x001133,transparent:true,opacity:.9});
  [-.06,.06].forEach(px=>{const p=new THREE.Mesh(new THREE.BoxGeometry(.025,.045,.002),pm);p.position.x=px;issG.add(p);});
  const glow=new THREE.Mesh(new THREE.SphereGeometry(.018,12,12),new THREE.MeshBasicMaterial({color:0x00e5ff,transparent:true,opacity:.6}));
  issG.add(glow);

  const trailPts=[],trailGeo=new THREE.BufferGeometry();
  scene.add(new THREE.Line(trailGeo,new THREE.LineBasicMaterial({color:0x00e5ff,transparent:true,opacity:.25})));
  scene.add(new THREE.AmbientLight(0x112244,.9));
  const sun=new THREE.DirectionalLight(0xffffff,1.4);sun.position.set(5,3,5);scene.add(sun);

  // Input (mouse + touch)
  let drag=false,ax=.25,ay=0,tx2=.25,ty2=0,px=0,py=0,auto=true;
  canvas.addEventListener('mousedown',e=>{drag=true;auto=false;px=e.clientX;py=e.clientY;});
  window.addEventListener('mouseup',()=>drag=false);
  canvas.addEventListener('mousemove',e=>{if(!drag)return;ty2+=(e.clientX-px)*.012;tx2+=(e.clientY-py)*.008;tx2=Math.max(-.7,Math.min(.7,tx2));px=e.clientX;py=e.clientY;});
  canvas.addEventListener('wheel',e=>{cam.position.z=Math.max(2,Math.min(7,cam.position.z+e.deltaY*.004));},{passive:true});
  let pt=null;
  canvas.addEventListener('touchstart',e=>{drag=true;auto=false;pt={x:e.touches[0].clientX,y:e.touches[0].clientY};e.preventDefault();},{passive:false});
  canvas.addEventListener('touchend',()=>{drag=false;pt=null;},{passive:true});
  canvas.addEventListener('touchmove',e=>{
    if(!pt)return;
    if(e.touches.length===2){
      // Pinch to zoom
      const dx=e.touches[0].clientX-e.touches[1].clientX,dy=e.touches[0].clientY-e.touches[1].clientY;
      const dist=Math.sqrt(dx*dx+dy*dy);
      if(pt.pinchDist)cam.position.z=Math.max(2,Math.min(7,cam.position.z+(pt.pinchDist-dist)*.01));
      pt.pinchDist=dist;
    }else{
      ty2+=(e.touches[0].clientX-pt.x)*.012;tx2+=(e.touches[0].clientY-pt.y)*.008;
      pt={x:e.touches[0].clientX,y:e.touches[0].clientY};
    }
    e.preventDefault();
  },{passive:false});

  let gph=0;
  (function tick(){
    requestAnimationFrame(tick);gph+=.04;
    if(auto)ty2+=.004;ax+=(tx2-ax)*.08;ay+=(ty2-ay)*.08;
    earth.rotation.x=ax;earth.rotation.y=ay;
    orbitPlane.rotation.x=ax;orbitPlane.rotation.y=ay;
    const{lat,lon}=STATE.iss;
    const phi=(90-lat)*Math.PI/180,theta=(lon+180)*Math.PI/180,r=ORB_R;
    const ix=-r*Math.sin(phi)*Math.cos(theta),iy=r*Math.cos(phi),iz=r*Math.sin(phi)*Math.sin(theta);
    const rm=new THREE.Matrix4().makeRotationY(ay).multiply(new THREE.Matrix4().makeRotationX(ax));
    const ipos=new THREE.Vector3(ix,iy,iz).applyMatrix4(rm);
    issG.position.copy(ipos);issG.lookAt(new THREE.Vector3(0,0,0).applyMatrix4(rm));
    glow.material.opacity=.4+.3*Math.sin(gph);
    if(!trailPts.length||ipos.distanceTo(trailPts[trailPts.length-1])>.001){
      trailPts.push(ipos.clone());if(trailPts.length>120)trailPts.shift();
      if(trailPts.length>1)trailGeo.setFromPoints(trailPts);
    }
    renderer.render(scene,cam);
  })();
}

// ══════════════════════════════════════════════════════
//  40-HOUR KEEPALIVE
// ══════════════════════════════════════════════════════
const KEEPALIVE={
  start:Date.now(),lock:null,
  async grab(){
    try{
      if('wakeLock' in navigator){
        this.lock=await navigator.wakeLock.request('screen');
        this.lock.addEventListener('release',()=>{el('keepalive-status').textContent='RE-ACQUIRING';el('keepalive-status').style.color='var(--amber)';setTimeout(()=>this.grab(),3000);});
        el('keepalive-status').textContent='SCREEN LOCK ✓';el('keepalive-status').style.color='var(--green)';
      }
    }catch{el('keepalive-status').textContent='AUDIO MODE';el('keepalive-status').style.color='var(--amber)';}
  },
  silent(){
    try{const c=SOUND._ctx(),b=c.createBuffer(1,c.sampleRate*3,c.sampleRate),s=c.createBufferSource(),g=c.createGain();s.buffer=b;s.loop=true;g.gain.value=.00001;s.connect(g);g.connect(c.destination);s.start();}catch{}
  },
  uptime(){
    const e=Math.floor((Date.now()-this.start)/1000);
    const h=String(Math.floor(e/3600)).padStart(2,'0'),m=String(Math.floor(e%3600/60)).padStart(2,'0'),s=String(e%60).padStart(2,'0');
    const d=el('uptime-display');if(d){d.textContent=`${h}:${m}:${s}`;if(e>39*3600)d.style.color='var(--amber)';}
  },
  async init(){
    await this.grab();
    document.addEventListener('visibilitychange',()=>{if(document.visibilityState==='visible')this.grab();});
    setInterval(()=>this.uptime(),1000);
    document.addEventListener('click',()=>this.silent(),{once:true});
    setTimeout(()=>{setInterval(()=>{fetchISS();fetchCrew();},30*60*1000);},60000);
  },
};

// ══════════════════════════════════════════════════════
//  PWA INSTALL PROMPT
// ══════════════════════════════════════════════════════
let deferredPrompt=null;
window.addEventListener('beforeinstallprompt',e=>{
  e.preventDefault();deferredPrompt=e;
  const banner=el('install-banner');banner.style.display='flex';
  el('install-yes').addEventListener('click',async()=>{
    banner.style.display='none';
    deferredPrompt.prompt();
    const{outcome}=await deferredPrompt.userChoice;
    if(outcome==='accepted')SOUND.iss();
    deferredPrompt=null;
  });
  el('install-no').addEventListener('click',()=>{banner.style.display='none';});
});
// iOS instructions via long-press on logo
document.querySelector('.logo')?.addEventListener('contextmenu',e=>{
  e.preventDefault();
  if(/iPhone|iPad|iPod/.test(navigator.userAgent)){
    alert('📲 To install on iOS:\nTap the Share button (□↑) at the bottom of Safari, then tap "Add to Home Screen".');
  }
});

// ══════════════════════════════════════════════════════
//  CLOCK
// ══════════════════════════════════════════════════════
function startClock(){
  (function tick(){
    const t=new Date().toUTCString().slice(17,25);
    el('h-clock').textContent=t;el('clock-display').textContent='UTC '+t;
    setTimeout(tick,1000);
  })();
}

// ══════════════════════════════════════════════════════
//  GEO
// ══════════════════════════════════════════════════════
function initGeo(){
  if(!navigator.geolocation){el('f-you').textContent='UNAVAILABLE';return;}
  navigator.geolocation.getCurrentPosition(pos=>{
    STATE.user.lat=pos.coords.latitude;STATE.user.lon=pos.coords.longitude;
    el('f-you').textContent=`${pos.coords.latitude.toFixed(2)}°, ${pos.coords.longitude.toFixed(2)}°`;
    if(STATE.map.ready)_placeUser();
  },()=>el('f-you').textContent='DENIED');
}

// ══════════════════════════════════════════════════════
//  ISS ALERT
// ══════════════════════════════════════════════════════
function checkAlert(lat,lon){
  const box=el('alert-box');
  if(STATE.user.lat===null){box.textContent='● MONITORING ISS';return;}
  const dist=Math.round(haversine(STATE.user.lat,STATE.user.lon,lat,lon));
  el('t-dist').textContent=dist.toLocaleString()+' km';
  if(dist<CFG.ALERT_KM){
    box.textContent='⚡ ISS OVERHEAD NOW!';box.className='active';
    if(!STATE.issNearby){SOUND.iss();STATE.issNearby=true;}
    Notification?.permission==='granted'&&new Notification('🛸 ISS Overhead!',{body:`Only ${dist.toLocaleString()} km away!`});
  }else if(dist<4000){
    box.textContent=`▲ ISS NEARBY — ${dist.toLocaleString()} km`;box.className='nearby';STATE.issNearby=false;
  }else{
    box.textContent=`● ISS — ${dist.toLocaleString()} km`;box.className='';STATE.issNearby=false;
  }
}

// ══════════════════════════════════════════════════════
//  UI UPDATE
// ══════════════════════════════════════════════════════
function updateUI(lat,lon,ts){
  const ls=latStr(lat),ln=lonStr(lon);
  el('t-lat').textContent=ls;el('t-lon').textContent=ln;
  el('f-lat').textContent=ls;el('f-lon').textContent=ln;
  el('hud-lat').textContent='LAT '+lat.toFixed(3)+'°';el('hud-lon').textContent='LON '+lon.toFixed(3)+'°';
  el('t-ts').textContent=new Date(ts*1000).toUTCString().slice(17,25);
  const P=92.9*60*1000,pct=((Date.now()%P)/P*100).toFixed(1);
  el('prog-fill').style.width=pct+'%';el('prog-pct').textContent=pct+'%';
  const today=Date.now()-new Date().setUTCHours(0,0,0,0);
  el('m-orbits').textContent=Math.floor(today/P);
  el('hud-orbits').textContent='#'+Math.floor(today/P);
  el('days-orbit').textContent=Math.floor((Date.now()-new Date('1998-11-20').getTime())/86400000).toLocaleString();
}

function renderCrew(people,count){
  el('h-people').textContent=count;
  const icons={ISS:'🛸',CSS:'🚀',Shenzhou:'🚀',Tiangong:'🔴'};
  el('crew-list').innerHTML=people.map((p,i)=>`
    <div class="crew-item" style="animation:fadeIn .3s ease ${i*.07}s both">
      <div class="crew-avatar">${icons[p.craft]||'👨‍🚀'}</div>
      <div class="crew-info"><div class="crew-name">${p.name}</div><div class="crew-craft">${p.craft}</div></div>
      <div class="crew-badge">${p.craft.toUpperCase()}</div>
    </div>`).join('');
}

// ══════════════════════════════════════════════════════
//  FETCH FUNCTIONS
// ══════════════════════════════════════════════════════
async function fetchISS(){
  try{const pos=await API.issPosition();STATE.iss=pos;updateUI(pos.lat,pos.lon,pos.ts);updateMapISS(pos.lat,pos.lon);checkAlert(pos.lat,pos.lon);}
  catch(e){console.warn('ISS:',e);}
}
async function fetchCrew(){
  try{const{count,people}=await API.astronauts();renderCrew(people,count);}
  catch{el('crew-list').innerHTML='<div class="loading-msg">DATA UNAVAILABLE</div>';el('h-people').textContent='?';}
}
async function fetchAnomalies(){
  const dot=el('scan-dot'),lbl=el('scan-label'),last=el('last-scan');
  dot.className='scan-pulse amber';lbl.textContent='SCANNING…';
  try{
    const aircraft=await SKYAPI.allStates();
    if(aircraft.length){
      el('radar-aircraft-count').textContent=aircraft.length.toLocaleString();
      const prevC=ANOMALY.counts.critical,prevH=ANOMALY.counts.high;
      const n=ANOMALY.ingest(aircraft);ANOMALY.render();
      dot.className='scan-pulse ok';lbl.textContent='SCANNED '+aircraft.length.toLocaleString()+' AIRCRAFT';
      last.textContent='LAST: '+new Date().toUTCString().slice(17,25);
      if(n>0){
        if(ANOMALY.counts.critical>prevC)SOUND.critical();
        else if(ANOMALY.counts.high>prevH)SOUND.high();
        else if(ANOMALY.counts.medium>0)SOUND.medium();
        else SOUND.low();
      }
      if(ANOMALY.counts.critical>0&&Notification?.permission==='granted')
        new Notification('🚨 Critical Squawk!',{body:`${ANOMALY.counts.critical} critical aircraft anomalies detected.`});
    }
  }catch(e){
    console.warn('OpenSky:',e);dot.className='scan-pulse';lbl.textContent='OPENSKY UNAVAILABLE — RETRY IN 60s';
    last.textContent='ERR '+new Date().toUTCString().slice(17,25);ANOMALY.render();
  }
}

// ══════════════════════════════════════════════════════
//  BOOT
// ══════════════════════════════════════════════════════
async function boot(){
  Notification?.requestPermission?.();
  SOUND.init();EBS.init();
  initStars();startClock();initGeo();initRadar();

  // Wait for deferred scripts to load
  await new Promise(r=>window.d3?r():window.addEventListener('load',r,{once:true}));

  await initMap();init3D();
  await fetchISS();await fetchCrew();
  RADIO.init();await KEEPALIVE.init();
  fetchAnomalies();

  // NWS alerts (wait for geo)
  const tryNWS=()=>STATE.user.lat!==null?EBS.fetch(STATE.user.lat,STATE.user.lon):setTimeout(tryNWS,2000);
  setTimeout(tryNWS,3000);

  setInterval(fetchISS,CFG.REFRESH_MS);
  setInterval(fetchCrew,5*60*1000);
  setInterval(fetchAnomalies,60*1000);
  setInterval(()=>{if(STATE.user.lat)EBS.fetch(STATE.user.lat,STATE.user.lon);},15*60*1000);
}

window.addEventListener('load',boot);
</script>
</body>
</html>
