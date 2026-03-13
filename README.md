<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ORBIT WATCH — Live Satellite Tracker</title>
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Share+Tech+Mono&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js"></script>
<style>
:root {
  --bg: #020912;
  --panel: #070f1d;
  --panel2: #0a1525;
  --border: #0b3d6a;
  --border2: #0f4d86;
  --cyan: #00e5ff;
  --cyan2: #00bcd4;
  --green: #00ff88;
  --amber: #ffaa00;
  --red: #ff3366;
  --purple: #a855f7;
  --text: #94bdd6;
  --dim: #3d6680;
  --bright: #c8e8f8;
}
* { margin:0; padding:0; box-sizing:border-box; }
html,body { width:100%; height:100%; }
body {
  background: var(--bg);
  color: var(--text);
  font-family: 'Share Tech Mono', monospace;
  min-height:100vh;
  overflow-x:hidden;
}

/* ── SCANLINES ── */
body::after {
  content:'';
  position:fixed; inset:0;
  background:repeating-linear-gradient(0deg,transparent,transparent 3px,rgba(0,0,0,0.04) 3px,rgba(0,0,0,0.04) 4px);
  pointer-events:none;
  z-index:9999;
}

/* ── STARS ── */
#stars-canvas {
  position:fixed; inset:0;
  pointer-events:none;
  z-index:0;
}

/* ── LAYOUT ── */
.app {
  position:relative;
  z-index:1;
  max-width:1500px;
  margin:0 auto;
  padding:12px;
  display:grid;
  gap:10px;
}

/* ── HEADER ── */
.header {
  display:grid;
  grid-template-columns:auto 1fr auto;
  align-items:center;
  gap:24px;
  padding:14px 24px;
  background:var(--panel);
  border:1px solid var(--border);
  border-bottom:2px solid var(--cyan);
  position:relative;
}
.header::before {
  content:'◆ OPEN NOTIFY API — LIVE ◆';
  position:absolute; top:-1px; left:50%; transform:translateX(-50%);
  font-size:9px; letter-spacing:4px; color:var(--cyan);
  background:var(--panel); padding:0 10px;
}
.logo {
  font-family:'Orbitron',sans-serif;
  font-size:26px; font-weight:900;
  color:var(--cyan);
  text-shadow:0 0 30px rgba(0,229,255,0.6);
  letter-spacing:5px;
  white-space:nowrap;
}
.logo em { color:var(--green); font-style:normal; }

.header-stats {
  display:flex; gap:0; align-items:stretch;
  border:1px solid var(--border);
}
.hstat {
  padding:8px 20px;
  border-right:1px solid var(--border);
  text-align:center;
}
.hstat:last-child { border-right:none; }
.hstat-label { font-size:8px; letter-spacing:3px; color:var(--dim); margin-bottom:4px; }
.hstat-val {
  font-family:'Orbitron',sans-serif;
  font-size:17px; color:var(--cyan);
}
.hstat-val.green { color:var(--green); }
.hstat-val.amber { color:var(--amber); }

.header-right { display:flex; flex-direction:column; gap:8px; align-items:flex-end; }

#alert-box {
  padding:7px 16px;
  border:1px solid var(--dim);
  font-size:10px; letter-spacing:2px; color:var(--dim);
  transition:all .4s ease;
}
#alert-box.active {
  border-color:var(--green); color:var(--green);
  background:rgba(0,255,136,0.08);
  box-shadow:0 0 20px rgba(0,255,136,0.2);
  animation:alertPulse 1.2s infinite;
}
#alert-box.nearby {
  border-color:var(--amber); color:var(--amber);
  background:rgba(255,170,0,0.08);
  animation:pulse 2s infinite;
}

#clock-display {
  font-family:'Orbitron',sans-serif;
  font-size:13px; color:var(--text); letter-spacing:2px;
}

/* ── MAIN GRID ── */
.main-grid {
  display:grid;
  grid-template-columns:1fr 310px;
  gap:10px;
}

/* ── PANEL ── */
.panel {
  background:var(--panel);
  border:1px solid var(--border);
  display:flex; flex-direction:column;
  position:relative;
  overflow:hidden;
}
.panel-head {
  display:flex; align-items:center; justify-content:space-between;
  padding:9px 14px;
  border-bottom:1px solid var(--border);
  font-size:9px; letter-spacing:3px; color:var(--cyan);
  flex-shrink:0;
}
.live-dot {
  width:7px; height:7px; border-radius:50%;
  background:var(--green);
  box-shadow:0 0 10px var(--green);
  animation:blink 1.4s infinite;
}
.live-dot.amber { background:var(--amber); box-shadow:0 0 10px var(--amber); }
.live-dot.cyan  { background:var(--cyan);  box-shadow:0 0 10px var(--cyan); }
.live-dot.purple{ background:var(--purple);box-shadow:0 0 10px var(--purple); }

/* ── WORLD MAP ── */
#map-svg-wrap {
  flex:1; min-height:380px; position:relative;
}
#world-svg {
  width:100%; height:100%;
  display:block;
}
#world-svg .land {
  fill:#081f10; stroke:#00ff8818; stroke-width:.4;
}
#world-svg .graticule {
  fill:none; stroke:rgba(0,229,255,0.05); stroke-width:.5;
}
#world-svg .equator {
  fill:none; stroke:rgba(0,229,255,0.2); stroke-width:.8;
}
#world-svg .tropic {
  fill:none; stroke:rgba(255,170,0,0.12); stroke-width:.5; stroke-dasharray:4,6;
}
#world-svg .orbit-track {
  fill:none; stroke:rgba(0,229,255,0.22); stroke-width:1.2; stroke-dasharray:5,5;
}

.map-footer {
  padding:9px 14px;
  border-top:1px solid var(--border);
  display:flex; gap:24px; flex-shrink:0;
  font-size:10px; color:var(--dim);
}
.map-footer b { color:var(--cyan); }

/* ── RIGHT COLUMN ── */
.right-col { display:flex; flex-direction:column; gap:10px; }

/* ── TELEMETRY GRID ── */
.tele-grid {
  display:grid; grid-template-columns:1fr 1fr; gap:1px;
  background:var(--border); border-top:1px solid var(--border);
}
.tele-cell {
  background:var(--panel); padding:13px 14px;
  display:flex; flex-direction:column; justify-content:space-between;
}
.tele-lbl {
  font-size:8px; letter-spacing:2px; color:var(--dim); margin-bottom:5px;
}
.tele-val {
  font-family:'Orbitron',sans-serif;
  font-size:16px; color:var(--green); line-height:1;
}
.tele-val.c { color:var(--cyan); }
.tele-val.a { color:var(--amber); }
.tele-val small { font-size:9px; color:var(--dim); font-family:'Share Tech Mono',monospace; margin-left:2px; }

/* Orbit progress */
.orbit-prog-wrap { padding:12px 14px; border-top:1px solid var(--border); flex-shrink:0; }
.prog-lbl { font-size:8px; letter-spacing:2px; color:var(--dim); margin-bottom:6px; display:flex; justify-content:space-between; }
.prog-track {
  height:3px; background:rgba(0,229,255,0.1);
}
.prog-fill {
  height:100%; background:linear-gradient(90deg,var(--cyan2),var(--cyan));
  box-shadow:0 0 8px var(--cyan);
  transition:width 1s linear;
}

/* ── CREW ── */
.crew-scroll { overflow-y:auto; max-height:300px; }
.crew-item {
  display:flex; align-items:center; gap:10px;
  padding:10px 14px;
  border-bottom:1px solid rgba(11,61,106,0.5);
  transition:background .2s;
}
.crew-item:hover { background:rgba(0,229,255,0.03); }
.crew-item:last-child { border-bottom:none; }
.crew-avatar {
  width:28px; height:28px; border-radius:50%;
  border:1px solid var(--cyan2);
  background:rgba(0,229,255,0.1);
  display:flex; align-items:center; justify-content:center;
  font-size:13px; flex-shrink:0;
}
.crew-info { flex:1; }
.crew-name { font-size:11px; color:var(--bright); }
.crew-craft {
  font-size:9px; color:var(--dim); letter-spacing:1px; margin-top:2px;
}
.crew-badge {
  font-size:8px; letter-spacing:1px;
  padding:2px 7px; border:1px solid var(--border2);
  color:var(--cyan2);
}

/* ── 3D SECTION ── */
.three-row {
  display:grid;
  grid-template-columns:1fr 340px;
  gap:10px;
}
#three-wrap { position:relative; min-height:440px; flex:1; }
#three-canvas {
  position:absolute; inset:0;
  width:100% !important; height:100% !important;
  display:block;
  cursor:grab;
}
#three-canvas:active { cursor:grabbing; }
.three-hud {
  position:absolute; inset:0;
  pointer-events:none;
  padding:14px;
  display:flex; flex-direction:column; justify-content:space-between;
}
.hud-top { display:flex; justify-content:space-between; align-items:flex-start; }
.hud-tag {
  font-size:9px; letter-spacing:2px; color:rgba(0,229,255,0.5);
  border:1px solid rgba(0,229,255,0.15);
  padding:3px 8px;
}
.hud-iss-pos {
  text-align:right;
  font-size:9px; color:rgba(0,229,255,0.6);
  line-height:1.6;
}
.hud-bottom {
  display:flex; gap:16px;
}
.hud-stat {
  font-size:9px; color:rgba(0,229,255,0.4); letter-spacing:1px;
}
.hud-stat b { color:rgba(0,229,255,0.8); }

/* ── ORBITAL MECHANICS PANEL ── */
.mech-panel { padding:16px; display:flex; flex-direction:column; gap:14px; }
.mech-item {}
.mech-label { font-size:8px; letter-spacing:3px; color:var(--dim); margin-bottom:5px; }
.mech-val {
  font-family:'Orbitron',sans-serif;
  font-size:20px; color:var(--cyan);
}
.mech-val.green { color:var(--green); }
.mech-val.amber { color:var(--amber); }
.mech-val small { font-size:10px; color:var(--dim); margin-left:3px; font-family:'Share Tech Mono',monospace; }

.mech-bar { margin-top:6px; height:2px; background:rgba(0,229,255,0.1); }
.mech-bar-fill { height:100%; background:var(--cyan); }

/* ── EXTRA INFO STRIP ── */
.info-strip {
  display:grid; grid-template-columns:repeat(4,1fr); gap:1px;
  background:var(--border);
  border:1px solid var(--border);
}
.istrip-cell {
  background:var(--panel2); padding:14px 16px;
  text-align:center;
}
.istrip-lbl { font-size:8px; letter-spacing:2px; color:var(--dim); margin-bottom:6px; }
.istrip-val {
  font-family:'Orbitron',sans-serif;
  font-size:20px; color:var(--cyan);
}
.istrip-val.green { color:var(--green); }

/* ── FOOTER ── */
.footer {
  text-align:center;
  padding:12px;
  font-size:9px; letter-spacing:3px; color:var(--dim);
  border-top:1px solid var(--border);
}

/* ── ANIMATIONS ── */
@keyframes blink { 0%,100%{opacity:1}50%{opacity:.2} }
@keyframes pulse { 0%,100%{opacity:1}50%{opacity:.6} }
@keyframes alertPulse {
  0%,100%{box-shadow:0 0 10px rgba(0,255,136,.2)}
  50%{box-shadow:0 0 25px rgba(0,255,136,.5)}
}
@keyframes spin { from{transform:rotate(0deg)}to{transform:rotate(360deg)} }
@keyframes fadeIn { from{opacity:0;transform:translateY(4px)}to{opacity:1;transform:translateY(0)} }

.loading-msg {
  padding:20px; text-align:center;
  font-size:10px; letter-spacing:3px; color:var(--dim);
  animation:pulse 1.5s infinite;
}

/* ── ANOMALY MONITOR ── */
.anomaly-section {
  display:grid;
  grid-template-columns:220px 1fr 240px;
  gap:10px;
}
.radar-wrap {
  display:flex; flex-direction:column;
}
.radar-canvas-wrap {
  flex:1; position:relative; min-height:230px;
  display:flex; align-items:center; justify-content:center;
  background:radial-gradient(circle, #030e08 60%, #020912 100%);
  overflow:hidden;
}
#radar-canvas { display:block; }

.anomaly-stats {
  display:grid; grid-template-columns:1fr 1fr; gap:1px;
  background:var(--border);
  border-top:1px solid var(--border);
  flex-shrink:0;
}
.a-stat {
  background:var(--panel); padding:10px 12px; text-align:center;
}
.a-stat-lbl { font-size:8px; letter-spacing:2px; color:var(--dim); margin-bottom:4px; }
.a-stat-val {
  font-family:'Orbitron',sans-serif; font-size:20px; color:var(--cyan);
}
.a-stat-val.red   { color:var(--red);   text-shadow:0 0 12px rgba(255,51,102,0.5); }
.a-stat-val.amber { color:var(--amber); text-shadow:0 0 12px rgba(255,170,0,0.4); }
.a-stat-val.green { color:var(--green); }

/* Feed */
.anomaly-feed-wrap {
  display:flex; flex-direction:column;
}
.anomaly-feed {
  flex:1; overflow-y:auto; max-height:360px;
}
.anomaly-feed::-webkit-scrollbar { width:3px; }
.anomaly-feed::-webkit-scrollbar-track { background:transparent; }
.anomaly-feed::-webkit-scrollbar-thumb { background:var(--border2); }

.a-event {
  display:grid;
  grid-template-columns:auto 1fr auto;
  gap:10px;
  padding:10px 14px;
  border-bottom:1px solid rgba(11,61,106,0.4);
  align-items:start;
  animation:slideIn .35s ease;
  transition:background .2s;
}
.a-event:hover { background:rgba(255,51,102,0.04); }
.a-event:last-child { border-bottom:none; }

@keyframes slideIn { from{opacity:0;transform:translateX(-8px)}to{opacity:1;transform:translateX(0)} }

.a-sev {
  width:8px; height:8px; border-radius:50%; margin-top:3px; flex-shrink:0;
}
.sev-critical { background:var(--red);   box-shadow:0 0 8px var(--red);   animation:blink .6s infinite; }
.sev-high     { background:#ff6600;      box-shadow:0 0 8px #ff6600;      animation:blink .9s infinite; }
.sev-medium   { background:var(--amber); box-shadow:0 0 6px var(--amber); }
.sev-low      { background:var(--cyan);  box-shadow:0 0 4px var(--cyan); opacity:.7; }

.a-body {}
.a-type {
  font-size:9px; letter-spacing:2px; color:var(--red);
  margin-bottom:3px;
}
.a-type.high   { color:#ff6600; }
.a-type.medium { color:var(--amber); }
.a-type.low    { color:var(--cyan2); }
.a-flight { font-size:12px; color:var(--bright); }
.a-detail { font-size:9px; color:var(--dim); margin-top:2px; line-height:1.5; }

.a-time {
  font-size:9px; color:var(--dim); white-space:nowrap;
}

/* Squawk badge */
.squawk-badge {
  display:inline-block;
  font-family:'Orbitron',sans-serif;
  font-size:10px; font-weight:700;
  padding:2px 8px;
  border:1px solid;
  margin-left:6px;
  vertical-align:middle;
}
.sq-7500 { border-color:var(--red);   color:var(--red);   background:rgba(255,51,102,0.1); animation:blink .7s infinite; }
.sq-7600 { border-color:#ff6600;      color:#ff6600;      background:rgba(255,102,0,0.1); }
.sq-7700 { border-color:var(--amber); color:var(--amber); background:rgba(255,170,0,0.1); }

/* Detail panel */
.anomaly-detail {
  display:flex; flex-direction:column; gap:0;
}
.a-info-block {
  padding:12px 14px;
  border-bottom:1px solid var(--border);
  flex-shrink:0;
}
.a-info-lbl { font-size:8px; letter-spacing:2px; color:var(--dim); margin-bottom:5px; }
.a-info-val {
  font-family:'Orbitron',sans-serif;
  font-size:16px; color:var(--cyan);
}
.a-info-val.red    { color:var(--red); }
.a-info-val.amber  { color:var(--amber); }

/* Threat tape */
.threat-tape {
  height:4px; background:rgba(255,51,102,0.1);
  margin-top:6px; overflow:hidden;
}
.threat-fill {
  height:100%; background:linear-gradient(90deg,var(--amber),var(--red));
  box-shadow:0 0 8px var(--red);
  transition:width .6s ease;
}

/* Scanner line */
.scan-status {
  padding:8px 14px; border-top:1px solid var(--border);
  font-size:9px; letter-spacing:2px; color:var(--dim);
  display:flex; justify-content:space-between; flex-shrink:0;
}
.scan-pulse {
  width:6px; height:6px; border-radius:50%; background:var(--red);
  box-shadow:0 0 6px var(--red); animation:blink .8s infinite;
  display:inline-block; margin-right:6px;
}
.scan-pulse.ok { background:var(--green); box-shadow:0 0 6px var(--green); animation:blink 2s infinite; }

/* anomaly section header anomaly count badge */
.anomaly-badge {
  display:inline-flex; align-items:center; justify-content:center;
  width:20px; height:20px; border-radius:50%;
  background:var(--red); color:#fff;
  font-size:10px; font-weight:bold;
  font-family:'Orbitron',sans-serif;
  margin-left:8px;
  box-shadow:0 0 10px rgba(255,51,102,0.6);
  animation:blink 1s infinite;
  display:none;
}

/* ── EBS TICKER ── */
#ebs-bar {
  position:fixed; top:0; left:0; right:0; z-index:10000;
  height:34px; display:flex; align-items:stretch;
  font-family:'Share Tech Mono',monospace;
  pointer-events:none;
  transform:translateY(-100%);
  transition:transform .4s ease;
}
#ebs-bar.visible { transform:translateY(0); }
#ebs-label {
  flex-shrink:0; display:flex; align-items:center;
  padding:0 14px; gap:8px;
  font-family:'Orbitron',sans-serif; font-size:10px; letter-spacing:3px; font-weight:700;
  white-space:nowrap; pointer-events:auto;
}
#ebs-bar.ebs-alert  { background:#8b0000; }
#ebs-bar.ebs-warn   { background:#7a4a00; }
#ebs-bar.ebs-info   { background:#003366; }
#ebs-bar.ebs-alert  #ebs-label { color:#fff; border-right:2px solid #ff3333; }
#ebs-bar.ebs-warn   #ebs-label { color:#ffcc00; border-right:2px solid #ffaa00; }
#ebs-bar.ebs-info   #ebs-label { color:#66ccff; border-right:2px solid #0088ff; }
#ebs-ticker-wrap {
  flex:1; overflow:hidden; display:flex; align-items:center;
  background:rgba(0,0,0,0.6); backdrop-filter:blur(4px);
  border-bottom:1px solid rgba(255,255,255,0.1);
}
#ebs-ticker-text {
  white-space:nowrap; font-size:12px; letter-spacing:1px;
  animation:ebsScroll 0s linear infinite;
  padding-left:100%;
}
#ebs-bar.ebs-alert #ebs-ticker-text { color:#ffaaaa; }
#ebs-bar.ebs-warn  #ebs-ticker-text { color:#ffe066; }
#ebs-bar.ebs-info  #ebs-ticker-text { color:#aaddff; }
#ebs-dismiss {
  flex-shrink:0; padding:0 14px; cursor:pointer; pointer-events:auto;
  display:flex; align-items:center; font-size:16px; opacity:.7;
  background:rgba(0,0,0,0.4); border:none; color:inherit;
}
#ebs-dismiss:hover { opacity:1; }
@keyframes ebsScroll {
  from { transform:translateX(0); }
  to   { transform:translateX(-100%); }
}

/* ── BODY PAD FOR EBS BAR ── */
body.ebs-active { padding-top:34px; }

/* ── MASTER SOUND TOGGLE ── */
#sound-toggle {
  position:fixed; bottom:18px; right:18px; z-index:9998;
  width:46px; height:46px; border-radius:50%;
  border:2px solid var(--border2); background:var(--panel);
  color:var(--cyan); font-size:20px;
  cursor:pointer; display:flex; align-items:center; justify-content:center;
  transition:all .2s; box-shadow:0 0 20px rgba(0,229,255,0.2);
}
#sound-toggle:hover { background:var(--panel2); box-shadow:0 0 30px rgba(0,229,255,0.4); }
#sound-toggle.muted { border-color:var(--dim); color:var(--dim); box-shadow:none; }

/* ── RADIO PANEL ── */
.radio-section {
  display:grid; grid-template-columns:1fr 1fr; gap:10px;
}
.radio-player {
  display:flex; flex-direction:column;
}
.now-playing {
  display:flex; align-items:center; gap:16px; padding:16px;
  border-bottom:1px solid var(--border); flex-shrink:0;
  background:linear-gradient(135deg,rgba(0,229,255,0.04),transparent);
}
.np-icon {
  width:48px; height:48px; border-radius:4px;
  border:1px solid var(--border2);
  background:rgba(0,229,255,0.08);
  display:flex; align-items:center; justify-content:center;
  font-size:24px; flex-shrink:0;
  position:relative; overflow:hidden;
}
.np-icon.playing::after {
  content:'';
  position:absolute; inset:0;
  background:rgba(0,229,255,0.15);
  animation:pulse 1s infinite;
}
.np-info { flex:1; min-width:0; }
.np-station { font-size:13px; color:var(--bright); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.np-freq { font-family:'Orbitron',sans-serif; font-size:18px; color:var(--cyan); margin-top:2px; }
.np-tag { font-size:9px; color:var(--dim); margin-top:3px; letter-spacing:1px; }
.np-controls { display:flex; gap:8px; align-items:center; flex-shrink:0; }
.radio-btn {
  width:36px; height:36px; border-radius:50%;
  border:1px solid var(--border2); background:transparent;
  color:var(--cyan); cursor:pointer; font-size:14px;
  display:flex; align-items:center; justify-content:center;
  transition:all .2s;
}
.radio-btn:hover { background:rgba(0,229,255,0.1); }
.radio-btn.play-btn {
  width:42px; height:42px; font-size:18px;
  border-color:var(--cyan);
  box-shadow:0 0 14px rgba(0,229,255,0.2);
}
.radio-btn.play-btn.active {
  background:rgba(0,229,255,0.15);
  box-shadow:0 0 20px rgba(0,229,255,0.4);
  animation:pulse 1.5s infinite;
}
.vol-bar {
  display:flex; align-items:center; gap:8px; padding:10px 16px;
  border-bottom:1px solid var(--border); flex-shrink:0;
}
.vol-bar label { font-size:8px; letter-spacing:2px; color:var(--dim); white-space:nowrap; }
#vol-slider {
  flex:1; -webkit-appearance:none; height:3px;
  background:rgba(0,229,255,0.15); outline:none; border-radius:2px;
}
#vol-slider::-webkit-slider-thumb {
  -webkit-appearance:none; width:12px; height:12px; border-radius:50%;
  background:var(--cyan); cursor:pointer; box-shadow:0 0 8px var(--cyan);
}
.station-list-wrap { flex:1; overflow-y:auto; max-height:340px; }
.station-list-wrap::-webkit-scrollbar { width:3px; }
.station-list-wrap::-webkit-scrollbar-thumb { background:var(--border2); }
.station-item {
  display:flex; align-items:center; gap:10px; padding:10px 14px;
  border-bottom:1px solid rgba(11,61,106,0.4); cursor:pointer;
  transition:background .15s;
}
.station-item:hover { background:rgba(0,229,255,0.04); }
.station-item.active { background:rgba(0,229,255,0.08); border-left:2px solid var(--cyan); }
.station-item.emergency { border-left:2px solid var(--red) !important; }
.station-item.emergency:hover,.station-item.emergency.active { background:rgba(255,51,102,0.06); }
.si-icon { font-size:16px; width:22px; text-align:center; flex-shrink:0; }
.si-info { flex:1; min-width:0; }
.si-name { font-size:11px; color:var(--bright); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.si-meta { font-size:9px; color:var(--dim); margin-top:1px; }
.si-freq {
  font-family:'Orbitron',sans-serif; font-size:11px; color:var(--cyan); flex-shrink:0;
}
.si-freq.red { color:var(--red); }

/* Frequency visualizer bars */
.freq-viz { height:32px; padding:8px 14px; border-top:1px solid var(--border); display:flex; align-items:flex-end; gap:2px; flex-shrink:0; }
.freq-bar { flex:1; background:var(--cyan); opacity:.5; border-radius:1px 1px 0 0; transition:height .1s; }

/* Emergency stations highlight */
.emerg-badge {
  font-size:8px; padding:1px 5px; border:1px solid var(--red); color:var(--red);
  margin-left:4px; letter-spacing:1px;
}

/* ── CREATOR BRAND ── */
.creator-tag {
  font-size:8px; letter-spacing:3px; color:var(--dim);
  display:flex; align-items:center; gap:6px;
}
.creator-tag span { color:var(--cyan); }

/* ── UPTIME BAR ── */
.uptime-bar {
  display:flex; align-items:center; gap:12px;
  padding:7px 14px; background:rgba(0,255,136,0.04);
  border-top:1px solid rgba(0,255,136,0.12); flex-shrink:0;
  font-size:9px; letter-spacing:2px; color:var(--dim);
}
.uptime-bar b { color:var(--green); }

/* responsive */
@media(max-width:960px){
  .main-grid,.three-row,.anomaly-section,.radio-section{grid-template-columns:1fr;}
  .info-strip{grid-template-columns:repeat(2,1fr);}
  .header{grid-template-columns:1fr;gap:10px;}
  .header-stats{display:grid;grid-template-columns:repeat(3,1fr);}
}
</style>
</head>
<body>

<!-- ═══ EBS EMERGENCY TICKER ═══ -->
<div id="ebs-bar" class="ebs-info">
  <div id="ebs-label">
    <span id="ebs-dot" style="width:8px;height:8px;border-radius:50%;background:#fff;animation:blink .7s infinite;display:inline-block"></span>
    <span id="ebs-type">EBS</span>
  </div>
  <div id="ebs-ticker-wrap">
    <div id="ebs-ticker-text">INITIALIZING EMERGENCY BROADCAST SYSTEM — FETCHING LOCAL ALERTS...</div>
  </div>
  <button id="ebs-dismiss" title="Dismiss">✕</button>
</div>

<!-- ═══ SOUND TOGGLE ═══ -->
<button id="sound-toggle" title="Toggle Sound Alerts">🔊</button>

<canvas id="stars-canvas"></canvas>

<div class="app">

  <!-- ═══ HEADER ═══ -->
  <div class="header">
    <div class="logo">ORBIT<em>WATCH</em></div>

    <div class="header-stats">
      <div class="hstat">
        <div class="hstat-label">PEOPLE IN SPACE</div>
        <div class="hstat-val green" id="h-people">—</div>
      </div>
      <div class="hstat">
        <div class="hstat-label">ISS ALTITUDE</div>
        <div class="hstat-val">408 <span style="font-size:10px;color:var(--dim)">KM</span></div>
      </div>
      <div class="hstat">
        <div class="hstat-label">ORBITAL VELOCITY</div>
        <div class="hstat-val">7.66 <span style="font-size:10px;color:var(--dim)">KM/S</span></div>
      </div>
      <div class="hstat">
        <div class="hstat-label">ORBITAL PERIOD</div>
        <div class="hstat-val">92.9 <span style="font-size:10px;color:var(--dim)">MIN</span></div>
      </div>
      <div class="hstat">
        <div class="hstat-label">INCLINATION</div>
        <div class="hstat-val">51.6 <span style="font-size:10px;color:var(--dim)">DEG</span></div>
      </div>
      <div class="hstat" style="border-right:none">
        <div class="hstat-label">UTC TIME</div>
        <div class="hstat-val amber" id="h-clock">--:--:--</div>
      </div>
    </div>

    <div class="header-right">
      <div id="alert-box">● MONITORING ISS</div>
      <div style="display:flex;gap:16px;align-items:center">
        <div id="clock-display">ORBIT WATCH v4.0</div>
        <div class="creator-tag">BY <span>@MrCodeBlood</span></div>
      </div>
    </div>
  </div>

  <!-- ═══ MAIN: MAP + RIGHT ═══ -->
  <div class="main-grid">

    <!-- 2D World Map -->
    <div class="panel">
      <div class="panel-head">
        <span>LIVE TRACKING MAP — EQUIRECTANGULAR — ISS REAL-TIME POSITION</span>
        <div class="live-dot"></div>
      </div>
      <div id="map-svg-wrap">
        <svg id="world-svg"></svg>
      </div>
      <div class="map-footer">
        <span>ISS LAT <b id="f-lat">—</b></span>
        <span>ISS LON <b id="f-lon">—</b></span>
        <span>YOUR POS <b id="f-you">LOCATING...</b></span>
        <span style="margin-left:auto;font-size:9px">◉ ISS &nbsp; ◉ YOU</span>
      </div>
    </div>

    <!-- Right Column -->
    <div class="right-col">

      <!-- Telemetry -->
      <div class="panel">
        <div class="panel-head">
          <span>ISS TELEMETRY — LIVE</span>
          <div class="live-dot cyan"></div>
        </div>
        <div class="tele-grid">
          <div class="tele-cell">
            <div class="tele-lbl">LATITUDE</div>
            <div class="tele-val" id="t-lat">—</div>
          </div>
          <div class="tele-cell">
            <div class="tele-lbl">LONGITUDE</div>
            <div class="tele-val" id="t-lon">—</div>
          </div>
          <div class="tele-cell">
            <div class="tele-lbl">ALTITUDE</div>
            <div class="tele-val c">408<small>km</small></div>
          </div>
          <div class="tele-cell">
            <div class="tele-lbl">VELOCITY</div>
            <div class="tele-val c">27,600<small>km/h</small></div>
          </div>
          <div class="tele-cell">
            <div class="tele-lbl">DIST FROM YOU</div>
            <div class="tele-val a" id="t-dist">—</div>
          </div>
          <div class="tele-cell">
            <div class="tele-lbl">TIMESTAMP</div>
            <div class="tele-val" style="font-size:11px" id="t-ts">—</div>
          </div>
        </div>
        <div class="orbit-prog-wrap">
          <div class="prog-lbl">
            <span>ORBITAL PROGRESS</span>
            <span id="prog-pct">0%</span>
          </div>
          <div class="prog-track">
            <div class="prog-fill" id="prog-fill" style="width:0%"></div>
          </div>
        </div>
      </div>

      <!-- Crew -->
      <div class="panel" style="flex:1">
        <div class="panel-head">
          <span>CREW IN SPACE</span>
          <div class="live-dot amber"></div>
        </div>
        <div class="crew-scroll" id="crew-list">
          <div class="loading-msg">FETCHING CREW DATA...</div>
        </div>
      </div>

    </div>
  </div>

  <!-- ═══ 3D + ORBITAL MECHANICS ═══ -->
  <div class="three-row">

    <!-- 3D Three.js Globe -->
    <div class="panel">
      <div class="panel-head">
        <span>3D ORBITAL VISUALIZATION — DRAG TO ROTATE — SCROLL TO ZOOM</span>
        <div class="live-dot purple"></div>
      </div>
      <div id="three-wrap">
        <canvas id="three-canvas"></canvas>
        <div class="three-hud">
          <div class="hud-top">
            <div class="hud-tag">INTERNATIONAL SPACE STATION</div>
            <div class="hud-iss-pos">
              <div id="hud-lat">LAT —</div>
              <div id="hud-lon">LON —</div>
              <div id="hud-alt">ALT 408 km</div>
            </div>
          </div>
          <div class="hud-bottom">
            <div class="hud-stat">ORBIT <b id="hud-orbits">—</b></div>
            <div class="hud-stat">PERIOD <b>92.9 MIN</b></div>
            <div class="hud-stat">INCL <b>51.6°</b></div>
            <div class="hud-stat" style="margin-left:auto">LIVE DATA</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Orbital Mechanics -->
    <div class="panel">
      <div class="panel-head">
        <span>ORBITAL MECHANICS</span>
        <div class="live-dot cyan"></div>
      </div>
      <div class="mech-panel">
        <div class="mech-item">
          <div class="mech-label">SEMI-MAJOR AXIS</div>
          <div class="mech-val">6,779 <small>km</small></div>
          <div class="mech-bar"><div class="mech-bar-fill" style="width:62%"></div></div>
        </div>
        <div class="mech-item">
          <div class="mech-label">ECCENTRICITY</div>
          <div class="mech-val green">0.0006</div>
          <div class="mech-bar"><div class="mech-bar-fill" style="width:4%;background:var(--green)"></div></div>
        </div>
        <div class="mech-item">
          <div class="mech-label">APOGEE</div>
          <div class="mech-val">415 <small>km</small></div>
        </div>
        <div class="mech-item">
          <div class="mech-label">PERIGEE</div>
          <div class="mech-val">402 <small>km</small></div>
        </div>
        <div class="mech-item">
          <div class="mech-label">ORBITS COMPLETED TODAY</div>
          <div class="mech-val amber" id="m-orbits">—</div>
        </div>
        <div class="mech-item">
          <div class="mech-label">REVOLUTIONS SINCE LAUNCH</div>
          <div class="mech-val" style="font-size:15px" id="m-total">~145,000+</div>
        </div>
      </div>
    </div>

  </div>

  <!-- ═══ INFO STRIP ═══ -->
  <div class="info-strip">
    <div class="istrip-cell">
      <div class="istrip-lbl">ISS LAUNCH DATE</div>
      <div class="istrip-val" style="font-size:13px">NOV 20, 1998</div>
    </div>
    <div class="istrip-cell">
      <div class="istrip-lbl">MASS</div>
      <div class="istrip-val">420,000 <span style="font-size:10px;color:var(--dim)">KG</span></div>
    </div>
    <div class="istrip-cell">
      <div class="istrip-lbl">SOLAR ARRAY SPAN</div>
      <div class="istrip-val">73 <span style="font-size:10px;color:var(--dim)">METRES</span></div>
    </div>
    <div class="istrip-cell">
      <div class="istrip-lbl">DAYS IN ORBIT</div>
      <div class="istrip-val green" id="days-orbit">—</div>
    </div>
  </div>

  <!-- ═══ ANOMALY MONITOR ═══ -->
  <div class="panel" style="border-color:rgba(255,51,102,0.3);border-top:2px solid var(--red)">
    <div class="panel-head" style="border-bottom-color:rgba(255,51,102,0.25)">
      <span style="color:var(--red)">
        ⚡ AIRSPACE ANOMALY MONITOR
        <span class="anomaly-badge" id="anomaly-badge">0</span>
      </span>
      <div style="display:flex;gap:14px;align-items:center">
        <span style="font-size:9px;color:var(--dim)" id="scan-region">GLOBAL SCAN</span>
        <div class="live-dot" style="background:var(--red);box-shadow:0 0 10px var(--red)"></div>
      </div>
    </div>

    <div class="anomaly-section">

      <!-- Radar -->
      <div class="panel radar-wrap" style="border:none">
        <div class="panel-head" style="font-size:8px;border-bottom-color:rgba(255,51,102,.2);color:var(--red)">
          <span>RADAR SWEEP</span>
          <span id="radar-aircraft-count" style="color:var(--dim)">— AIRCRAFT</span>
        </div>
        <div class="radar-canvas-wrap">
          <canvas id="radar-canvas" width="200" height="200"></canvas>
        </div>
        <div class="anomaly-stats">
          <div class="a-stat">
            <div class="a-stat-lbl">CRITICAL</div>
            <div class="a-stat-val red" id="cnt-critical">0</div>
          </div>
          <div class="a-stat">
            <div class="a-stat-lbl">HIGH</div>
            <div class="a-stat-val amber" id="cnt-high">0</div>
          </div>
          <div class="a-stat">
            <div class="a-stat-lbl">MEDIUM</div>
            <div class="a-stat-val" id="cnt-medium">0</div>
          </div>
          <div class="a-stat">
            <div class="a-stat-lbl">SCANNED</div>
            <div class="a-stat-val green" id="cnt-scanned">—</div>
          </div>
        </div>
      </div>

      <!-- Live Feed -->
      <div class="panel anomaly-feed-wrap" style="border:none">
        <div class="panel-head" style="font-size:8px;border-bottom-color:rgba(255,51,102,.2)">
          <span style="color:var(--dim)">LIVE ANOMALY EVENT LOG</span>
          <span style="color:var(--dim);font-size:8px" id="feed-count">0 EVENTS</span>
        </div>
        <div class="anomaly-feed" id="anomaly-feed">
          <div class="loading-msg" style="color:var(--dim)">INITIALIZING AIRSPACE SCANNER...</div>
        </div>
        <div class="scan-status">
          <span><span class="scan-pulse ok" id="scan-dot"></span><span id="scan-label">SCANNING GLOBAL AIRSPACE</span></span>
          <span id="last-scan">—</span>
        </div>
      </div>

      <!-- Detail / Reference -->
      <div class="panel anomaly-detail" style="border:none">
        <div class="panel-head" style="font-size:8px;color:var(--red);border-bottom-color:rgba(255,51,102,.2)">
          <span>SQUAWK CODE REFERENCE</span>
        </div>

        <div class="a-info-block">
          <div class="a-info-lbl">SQUAWK 7500 — HIJACK</div>
          <div class="a-info-val red">CRITICAL</div>
          <div class="threat-tape"><div class="threat-fill" style="width:100%"></div></div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">Unlawful interference. Silent distress. Highest priority.</div>
        </div>

        <div class="a-info-block">
          <div class="a-info-lbl">SQUAWK 7600 — RADIO FAILURE</div>
          <div class="a-info-val" style="color:#ff6600">HIGH</div>
          <div class="threat-tape"><div class="threat-fill" style="width:70%;background:linear-gradient(90deg,#ff6600,#ff9900)"></div></div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">Loss of communication. Aircraft squawking blind.</div>
        </div>

        <div class="a-info-block">
          <div class="a-info-lbl">SQUAWK 7700 — EMERGENCY</div>
          <div class="a-info-val amber">HIGH</div>
          <div class="threat-tape"><div class="threat-fill" style="width:75%;background:linear-gradient(90deg,var(--amber),#ff8800)"></div></div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">General emergency. Engine failure, medical, fuel.</div>
        </div>

        <div class="a-info-block">
          <div class="a-info-lbl">RAPID DESCENT DETECTED</div>
          <div class="a-info-val amber">MEDIUM</div>
          <div class="threat-tape"><div class="threat-fill" style="width:50%;background:linear-gradient(90deg,var(--cyan2),var(--amber))"></div></div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">Vertical rate &lt; −15 m/s (&gt;3,000 ft/min descent).</div>
        </div>

        <div class="a-info-block">
          <div class="a-info-lbl">EXTREME SPEED ANOMALY</div>
          <div class="a-info-val" style="color:var(--cyan)">LOW</div>
          <div class="threat-tape"><div class="threat-fill" style="width:30%;background:var(--cyan)"></div></div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">Ground speed &gt; 350 m/s at low altitude. Unusual.</div>
        </div>

        <div class="a-info-block" style="border-bottom:none">
          <div class="a-info-lbl">SPI — IDENT SIGNAL</div>
          <div class="a-info-val" style="color:var(--cyan)">LOW</div>
          <div style="font-size:9px;color:var(--dim);margin-top:6px">Special Position Indicator activated by ATC request.</div>
        </div>
      </div>

    </div>
  </div>

  <!-- ═══ RADIO SCANNER + EMERGENCY FREQUENCIES ═══ -->
  <div class="radio-section">

    <!-- Player + station list -->
    <div class="panel radio-player">
      <div class="panel-head" style="border-bottom-color:rgba(0,229,255,0.2)">
        <span>📻 LIVE RADIO SCANNER — EMERGENCY &amp; NEWS FREQUENCIES</span>
        <div class="live-dot cyan"></div>
      </div>

      <!-- Now Playing -->
      <div class="now-playing">
        <div class="np-icon" id="np-icon">📡</div>
        <div class="np-info">
          <div class="np-station" id="np-name">SELECT A STATION</div>
          <div class="np-freq" id="np-freq">— MHz</div>
          <div class="np-tag" id="np-tags">EMERGENCY · NEWS · SCANNER</div>
        </div>
        <div class="np-controls">
          <button class="radio-btn" id="prev-btn" title="Previous">⏮</button>
          <button class="radio-btn play-btn" id="play-btn" title="Play/Pause">▶</button>
          <button class="radio-btn" id="next-btn" title="Next">⏭</button>
        </div>
      </div>

      <!-- Volume -->
      <div class="vol-bar">
        <label>VOL</label>
        <input type="range" id="vol-slider" min="0" max="1" step="0.01" value="0.7">
        <label id="vol-pct" style="width:30px;text-align:right">70%</label>
      </div>

      <!-- Frequency visualizer -->
      <div class="freq-viz" id="freq-viz">
        <!-- bars injected by JS -->
      </div>

      <!-- Station List -->
      <div class="station-list-wrap">
        <div id="station-list">
          <div class="loading-msg">SCANNING RADIO FREQUENCIES...</div>
        </div>
      </div>

      <div class="uptime-bar">
        <span class="scan-pulse ok" style="width:6px;height:6px;border-radius:50%;background:var(--green);box-shadow:0 0 6px var(--green);animation:blink 2s infinite;display:inline-block"></span>
        SESSION UPTIME: <b id="uptime-display">00:00:00</b>
        &nbsp;·&nbsp; KEEPALIVE: <b id="keepalive-status" style="color:var(--green)">ACTIVE</b>
        &nbsp;·&nbsp; MAX 40HR MODE
      </div>
    </div>

    <!-- Emergency frequency reference + alerts -->
    <div class="panel" style="display:flex;flex-direction:column">
      <div class="panel-head" style="color:var(--red);border-bottom-color:rgba(255,51,102,0.25)">
        <span>⚡ EMERGENCY FREQUENCY REFERENCE</span>
        <div class="live-dot" style="background:var(--red);box-shadow:0 0 8px var(--red);animation:blink .8s infinite"></div>
      </div>

      <div style="display:grid;grid-template-columns:1fr 1fr;gap:1px;background:var(--border);flex-shrink:0">
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">CIVIL AIR PATROL</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--red)">121.5 <span style="font-size:10px;color:var(--dim)">MHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">Int'l Distress — Mayday</div>
        </div>
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">MARITIME DISTRESS</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--amber)">156.8 <span style="font-size:10px;color:var(--dim)">MHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">VHF Ch16 — Coast Guard</div>
        </div>
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">NOAA WEATHER (US)</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--cyan)">162.4 <span style="font-size:10px;color:var(--dim)">MHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">NWS Emergency Alerts</div>
        </div>
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">MIL AIR COMMAND</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--purple)">243.0 <span style="font-size:10px;color:var(--dim)">MHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">NATO Guard — Aircraft</div>
        </div>
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">POLICE/FIRE DISPATCH</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--green)">155.3 <span style="font-size:10px;color:var(--dim)">MHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">Public Safety Primary</div>
        </div>
        <div style="background:var(--panel);padding:10px 12px">
          <div style="font-size:8px;letter-spacing:2px;color:var(--dim);margin-bottom:4px">AM BROADCAST EMERG</div>
          <div style="font-family:Orbitron,sans-serif;font-size:16px;color:var(--bright)">640 <span style="font-size:10px;color:var(--dim)">kHz</span></div>
          <div style="font-size:9px;color:var(--dim);margin-top:2px">EAS Primary Entry Point</div>
        </div>
      </div>

      <!-- Local NWS Alerts -->
      <div class="panel-head" style="color:var(--amber);border-bottom-color:rgba(255,170,0,0.2);margin-top:0;border-top:1px solid var(--border)">
        <span>🌩 LOCAL NWS ALERTS — YOUR AREA</span>
        <div class="live-dot amber"></div>
      </div>
      <div id="nws-alert-list" style="flex:1;overflow-y:auto;max-height:220px">
        <div class="loading-msg">FETCHING LOCAL WEATHER ALERTS...</div>
      </div>

      <!-- Audio element (hidden) -->
      <audio id="radio-audio" preload="none" crossorigin="anonymous" style="display:none"></audio>
    </div>

  </div>

  <div class="footer">
    DATA: OPEN NOTIFY · OPENSKY NETWORK · NWS WEATHER · RADIO BROWSER API &nbsp;·&nbsp;
    CREATED BY <span style="color:var(--cyan);letter-spacing:2px">@MrCodeBlood</span> &nbsp;·&nbsp;
    ORBIT WATCH v4.0 &copy; 2025 &nbsp;·&nbsp; 40-HOUR CONTINUOUS OPERATION MODE
  </div>

</div>

<script>
// ════════════════════════════════════════════════════════
//  ORBIT WATCH — Handmade Open Notify API Integration
// ════════════════════════════════════════════════════════

const CFG = {
  REFRESH_MS:    5000,
  ALERT_KM:      1500,
  EARTH_R:       6371,
  ISS_ALT:       408,
};

const STATE = {
  iss:  { lat:0, lon:0, ts:0 },
  user: { lat:null, lon:null },
  map:  { ready:false, proj:null, path:null, svg:null, issG:null, userG:null, orbitPath:null },
};

// ─────────────────────────────────────────────
//  HANDMADE OPEN NOTIFY API
// ─────────────────────────────────────────────
const API = {
  BASE: 'https://api.open-notify.org',

  async _get(ep) {
    // Try direct, fall back to CORS proxy
    try {
      const r = await fetch(this.BASE + ep);
      if (!r.ok) throw new Error('HTTP ' + r.status);
      return r.json();
    } catch(e) {
      const proxy = 'https://api.allorigins.win/get?url=' + encodeURIComponent(this.BASE + ep);
      const r2 = await fetch(proxy);
      const wrapper = await r2.json();
      return JSON.parse(wrapper.contents);
    }
  },

  // GET /iss-now.json
  async issPosition() {
    const d = await this._get('/iss-now.json');
    return {
      lat: parseFloat(d.iss_position.latitude),
      lon: parseFloat(d.iss_position.longitude),
      ts:  d.timestamp,
    };
  },

  // GET /astros.json
  async astronauts() {
    const d = await this._get('/astros.json');
    return { count: d.number, people: d.people };
  },
};

// ─────────────────────────────────────────────
//  UTILITIES
// ─────────────────────────────────────────────
function haversine(la1, lo1, la2, lo2) {
  const R = CFG.EARTH_R;
  const dLa = (la2-la1)*Math.PI/180, dLo = (lo2-lo1)*Math.PI/180;
  const a = Math.sin(dLa/2)**2 + Math.cos(la1*Math.PI/180)*Math.cos(la2*Math.PI/180)*Math.sin(dLo/2)**2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
}
function latStr(v) { return Math.abs(v).toFixed(4)+'° '+(v>=0?'N':'S'); }
function lonStr(v) { return Math.abs(v).toFixed(4)+'° '+(v>=0?'E':'W'); }
function el(id)    { return document.getElementById(id); }

// ─────────────────────────────────────────────
//  STAR FIELD
// ─────────────────────────────────────────────
function initStars() {
  const c = el('stars-canvas'), ctx = c.getContext('2d');
  function resize() { c.width = innerWidth; c.height = innerHeight; }
  resize();
  window.addEventListener('resize', resize);

  const stars = Array.from({length:280}, () => ({
    x: Math.random(), y: Math.random(),
    r: Math.random()*1.4+.2,
    phase: Math.random()*Math.PI*2,
    speed: Math.random()*.008+.002,
  }));

  (function tick() {
    ctx.clearRect(0,0,c.width,c.height);
    stars.forEach(s => {
      s.phase += s.speed;
      const a = .2 + .6*Math.abs(Math.sin(s.phase));
      ctx.beginPath();
      ctx.arc(s.x*c.width, s.y*c.height, s.r, 0, Math.PI*2);
      ctx.fillStyle = `rgba(180,220,255,${a})`;
      ctx.fill();
    });
    requestAnimationFrame(tick);
  })();
}

// ─────────────────────────────────────────────
//  CLOCK
// ─────────────────────────────────────────────
function startClock() {
  (function tick() {
    const now = new Date();
    const t = now.toUTCString().slice(17,25);
    el('h-clock').textContent = t;
    el('clock-display').textContent = 'UTC ' + t;
    setTimeout(tick, 1000);
  })();
}

// ─────────────────────────────────────────────
//  GEOLOCATION
// ─────────────────────────────────────────────
function initGeo() {
  if (!navigator.geolocation) { el('f-you').textContent = 'UNAVAILABLE'; return; }
  navigator.geolocation.getCurrentPosition(pos => {
    STATE.user.lat = pos.coords.latitude;
    STATE.user.lon = pos.coords.longitude;
    el('f-you').textContent = `${pos.coords.latitude.toFixed(2)}°, ${pos.coords.longitude.toFixed(2)}°`;
    if (STATE.map.ready) placeUserDot();
  }, () => { el('f-you').textContent = 'DENIED'; });
}

// ─────────────────────────────────────────────
//  ISS ALERT
// ─────────────────────────────────────────────
function checkAlert(lat, lon) {
  const box = el('alert-box');
  if (STATE.user.lat === null) { box.textContent = '● MONITORING ISS'; return; }
  const dist = Math.round(haversine(STATE.user.lat, STATE.user.lon, lat, lon));
  el('t-dist').textContent = dist.toLocaleString() + ' km';

  if (dist < CFG.ALERT_KM) {
    box.textContent = '⚡ ISS OVERHEAD NOW!';
    box.className = 'active';
    if (Notification?.permission === 'granted') {
      new Notification('🛸 ISS Overhead!', { body: `ISS is only ${dist.toLocaleString()} km from you!` });
    }
  } else if (dist < 4000) {
    box.textContent = `▲ ISS NEARBY — ${dist.toLocaleString()} km`;
    box.className = 'nearby';
  } else {
    box.textContent = `● ISS — ${dist.toLocaleString()} km away`;
    box.className = '';
  }
}

// ─────────────────────────────────────────────
//  2D WORLD MAP
// ─────────────────────────────────────────────
async function initMap() {
  const wrap = el('map-svg-wrap');
  const W = wrap.clientWidth || 800;
  const H = wrap.clientHeight || 380;

  const svg = d3.select('#world-svg').attr('viewBox', `0 0 ${W} ${H}`);
  STATE.map.svg = svg;

  const proj = d3.geoEquirectangular()
    .scale(W / (2 * Math.PI))
    .translate([W/2, H/2]);
  const path = d3.geoPath().projection(proj);
  STATE.map.proj = proj;
  STATE.map.path = path;

  // Ocean
  svg.append('rect').attr('width',W).attr('height',H).attr('fill','#020912');

  // Grid
  svg.append('path').datum(d3.geoGraticule()()).attr('class','graticule').attr('d',path);

  // Land
  try {
    const world = await d3.json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
    svg.append('g')
      .selectAll('path')
      .data(topojson.feature(world, world.objects.countries).features)
      .enter().append('path')
      .attr('class','land').attr('d',path);
  } catch(e) { console.warn('Map tiles unavailable'); }

  // Equator
  const eqData = { type:'Feature', geometry:{ type:'LineString', coordinates: d3.range(-180,181,2).map(x=>[x,0]) }};
  svg.append('path').datum(eqData).attr('class','equator').attr('d',path);

  // Tropics
  [23.5,-23.5].forEach(lat => {
    const td = { type:'Feature', geometry:{ type:'LineString', coordinates: d3.range(-180,181,2).map(x=>[x,lat]) }};
    svg.append('path').datum(td).attr('class','tropic').attr('d',path);
  });

  // Orbit track placeholder
  STATE.map.orbitPath = svg.append('path').attr('class','orbit-track');

  // User dot
  const userG = svg.append('g').attr('opacity',0);
  userG.append('circle').attr('r',8).attr('fill','rgba(255,170,0,0.15)').attr('stroke','#ffaa00').attr('stroke-width',1.5);
  userG.append('circle').attr('r',3).attr('fill','#ffaa00');
  userG.append('text').attr('x',10).attr('y',-8).attr('fill','#ffaa00')
    .attr('font-family','Share Tech Mono,monospace').attr('font-size','9px').text('YOU');
  STATE.map.userG = userG;

  // ISS group
  const issG = svg.append('g');
  // Outer glow ring animated
  issG.append('circle').attr('r',16).attr('fill','none').attr('stroke','rgba(0,229,255,0.15)').attr('stroke-width',1.5)
    .style('animation','pulse 2s infinite');
  issG.append('circle').attr('r',6).attr('fill','rgba(0,229,255,0.2)').attr('stroke','var(--cyan)').attr('stroke-width',1);
  issG.append('circle').attr('r',2.5).attr('fill','#00e5ff');
  // Crosshairs
  [[-14,0,14,0],[0,-14,0,14]].forEach(([x1,y1,x2,y2])=>
    issG.append('line').attr('x1',x1).attr('y1',y1).attr('x2',x2).attr('y2',y2)
      .attr('stroke','rgba(0,229,255,0.35)').attr('stroke-width',.8)
  );
  issG.append('text').attr('x',12).attr('y',-9).attr('fill','#00e5ff')
    .attr('font-family','Share Tech Mono,monospace').attr('font-size','10px').attr('letter-spacing','1px').text('ISS');
  STATE.map.issG = issG;
  STATE.map.ready = true;

  placeUserDot();
}

function placeUserDot() {
  if (!STATE.map.ready || STATE.user.lat === null) return;
  const [x,y] = STATE.map.proj([STATE.user.lon, STATE.user.lat]);
  STATE.map.userG.attr('transform',`translate(${x},${y})`).attr('opacity',1);
}

function updateMapISS(lat, lon) {
  if (!STATE.map.ready) return;
  const [x,y] = STATE.map.proj([lon, lat]);
  STATE.map.issG.attr('transform',`translate(${x},${y})`);

  // Approximate orbit track
  const inc = 51.6 * Math.PI / 180;
  const pts = [];
  for (let t = -50; t <= 50; t++) {
    const ang = t * Math.PI / 180 * 4.5;
    const oLat = Math.asin(Math.sin(inc)*Math.sin(ang)) * 180/Math.PI;
    let oLon = lon + t * 4.5;
    while (oLon > 180)  oLon -= 360;
    while (oLon < -180) oLon += 360;
    pts.push([oLon, oLat]);
  }
  try {
    const feat = {type:'Feature',geometry:{type:'LineString',coordinates:pts}};
    STATE.map.orbitPath.datum(feat).attr('d',STATE.map.path);
  } catch(e) {}
}

// ─────────────────────────────────────────────
//  UI UPDATES
// ─────────────────────────────────────────────
function updateUI(lat, lon, ts) {
  const ls = latStr(lat), ln = lonStr(lon);
  el('t-lat').textContent = ls;
  el('t-lon').textContent = ln;
  el('f-lat').textContent = ls;
  el('f-lon').textContent = ln;
  el('hud-lat').textContent = 'LAT ' + lat.toFixed(3) + '°';
  el('hud-lon').textContent = 'LON ' + lon.toFixed(3) + '°';
  el('t-ts').textContent = new Date(ts*1000).toUTCString().slice(17,25);

  const PERIOD_MS = 92.9 * 60 * 1000;
  const pct = ((Date.now() % PERIOD_MS) / PERIOD_MS * 100).toFixed(1);
  el('prog-fill').style.width = pct + '%';
  el('prog-pct').textContent = pct + '%';

  const msToday = Date.now() - new Date().setUTCHours(0,0,0,0);
  const orbitsToday = Math.floor(msToday / PERIOD_MS);
  el('m-orbits').textContent = orbitsToday;
  el('hud-orbits').textContent = '#' + orbitsToday;

  // Days in orbit since Nov 20 1998
  const daysOrbit = Math.floor((Date.now() - new Date('1998-11-20').getTime())/(1000*86400));
  el('days-orbit').textContent = daysOrbit.toLocaleString();
}

function renderCrew(people, count) {
  el('h-people').textContent = count;
  const crafts = { ISS:'🛸', CSS:'🚀', 'Shenzhou':'🚀', Tiangong:'🔴' };
  el('crew-list').innerHTML = people.map((p,i) => `
    <div class="crew-item" style="animation:fadeIn .3s ease ${i*.07}s both">
      <div class="crew-avatar">${crafts[p.craft] || '👨‍🚀'}</div>
      <div class="crew-info">
        <div class="crew-name">${p.name}</div>
        <div class="crew-craft">${p.craft}</div>
      </div>
      <div class="crew-badge">${p.craft.toUpperCase()}</div>
    </div>
  `).join('');
}

// ─────────────────────────────────────────────
//  3D THREE.JS
// ─────────────────────────────────────────────
function init3D() {
  const wrap = el('three-wrap');
  const canvas = el('three-canvas');
  let W = wrap.clientWidth, H = wrap.clientHeight;

  const renderer = new THREE.WebGLRenderer({ canvas, antialias:true, alpha:true });
  renderer.setSize(W, H);
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

  const scene = new THREE.Scene();
  const cam = new THREE.PerspectiveCamera(45, W/H, 0.1, 500);
  cam.position.set(0, 0, 3.8);

  // Resize observer
  new ResizeObserver(() => {
    W = wrap.clientWidth; H = wrap.clientHeight;
    renderer.setSize(W, H);
    cam.aspect = W/H;
    cam.updateProjectionMatrix();
  }).observe(wrap);

  // ── Stars ──
  const sv = [];
  for (let i=0;i<4000;i++) {
    const t = Math.random()*Math.PI*2, p = Math.acos(2*Math.random()-1);
    const r = 60 + Math.random()*40;
    sv.push(r*Math.sin(p)*Math.cos(t), r*Math.sin(p)*Math.sin(t), r*Math.cos(p));
  }
  const sGeo = new THREE.BufferGeometry();
  sGeo.setAttribute('position', new THREE.Float32BufferAttribute(sv, 3));
  scene.add(new THREE.Points(sGeo, new THREE.PointsMaterial({color:0xffffff,size:.06,transparent:true,opacity:.8})));

  // ── Procedural Earth Texture ──
  const tc = document.createElement('canvas');
  tc.width = 1024; tc.height = 512;
  const tx = tc.getContext('2d');

  // Ocean
  const oceanGrad = tx.createLinearGradient(0,0,0,512);
  oceanGrad.addColorStop(0,'#020d1a'); oceanGrad.addColorStop(1,'#030f20');
  tx.fillStyle = oceanGrad; tx.fillRect(0,0,1024,512);

  // Grid
  tx.strokeStyle='rgba(0,229,255,0.04)'; tx.lineWidth=.5;
  for(let i=0;i<=8;i++){tx.beginPath();tx.moveTo(0,i*64);tx.lineTo(1024,i*64);tx.stroke();}
  for(let i=0;i<=16;i++){tx.beginPath();tx.moveTo(i*64,0);tx.lineTo(i*64,512);tx.stroke();}

  // Continents (simplified blobs)
  tx.fillStyle = '#081f10';
  const continents = [
    // N America
    {cx:180,cy:165,rx:80,ry:85,rot:-.15},
    // S America
    {cx:240,cy:300,rx:48,ry:90,rot:.12},
    // Europe
    {cx:510,cy:150,rx:52,ry:55,rot:0},
    // Africa
    {cx:515,cy:280,rx:58,ry:100,rot:.08},
    // Asia
    {cx:710,cy:155,rx:160,ry:90,rot:0},
    // Australia
    {cx:800,cy:340,rx:52,ry:38,rot:.1},
    // Antarctica (simplified)
    {cx:512,cy:490,rx:200,ry:22,rot:0},
    // Greenland
    {cx:290,cy:90,rx:35,ry:28,rot:0},
  ];
  continents.forEach(({cx,cy,rx,ry,rot})=>{
    tx.save();
    tx.translate(cx,cy); tx.rotate(rot);
    tx.beginPath(); tx.ellipse(0,0,rx,ry,0,0,Math.PI*2);
    tx.fill(); tx.restore();
  });

  // Subtle glow on land
  tx.fillStyle = 'rgba(0,255,136,0.03)';
  continents.forEach(({cx,cy,rx,ry,rot})=>{
    tx.save();
    tx.translate(cx,cy); tx.rotate(rot);
    tx.beginPath(); tx.ellipse(0,0,rx+3,ry+3,0,0,Math.PI*2);
    tx.fill(); tx.restore();
  });

  // Equator line
  tx.strokeStyle='rgba(0,229,255,0.15)'; tx.lineWidth=1.5;
  tx.beginPath(); tx.moveTo(0,256); tx.lineTo(1024,256); tx.stroke();

  const earthTex = new THREE.CanvasTexture(tc);

  // ── Earth ──
  const earthGeo = new THREE.SphereGeometry(1, 64, 64);
  const earthMat = new THREE.MeshPhongMaterial({
    map: earthTex,
    color:0x0a3f6b, emissive:0x001a33,
    specular:0x224466, shininess:12,
  });
  const earth = new THREE.Mesh(earthGeo, earthMat);
  scene.add(earth);

  // ── Atmosphere layers ──
  const makeAtmos = (r, color, opacity) => {
    const m = new THREE.Mesh(
      new THREE.SphereGeometry(r,32,32),
      new THREE.MeshPhongMaterial({color,transparent:true,opacity,side:THREE.FrontSide,depthWrite:false})
    );
    scene.add(m); return m;
  };
  makeAtmos(1.02, 0x0055ff, 0.06);
  makeAtmos(1.05, 0x003399, 0.04);
  makeAtmos(1.08, 0x0022aa, 0.025);

  // ── Orbit ring ──
  const ORB_R = (CFG.EARTH_R + CFG.ISS_ALT) / CFG.EARTH_R; // ~1.064
  const orbitRing = new THREE.Mesh(
    new THREE.TorusGeometry(ORB_R, 0.0025, 8, 200),
    new THREE.MeshBasicMaterial({color:0x00e5ff, transparent:true, opacity:.35})
  );
  orbitRing.rotation.x = Math.PI/2;
  scene.add(orbitRing);

  // ISS orbital plane group (inclined 51.6°)
  const orbitalPlane = new THREE.Group();
  orbitalPlane.rotation.z = 51.6 * Math.PI/180;
  scene.add(orbitalPlane);
  orbitalPlane.add(orbitRing);

  // ── ISS mesh ──
  const issGroup = new THREE.Group();
  scene.add(issGroup);

  // Main body
  const issBody = new THREE.Mesh(
    new THREE.BoxGeometry(.04,.012,.012),
    new THREE.MeshPhongMaterial({color:0xccddee,emissive:0x002244,shininess:60})
  );
  issGroup.add(issBody);

  // Solar panels (cross shape)
  const panelMat = new THREE.MeshPhongMaterial({color:0x00aaff,emissive:0x001133,transparent:true,opacity:.9});
  [[-0.06,0,0],[0.06,0,0]].forEach(([px,py,pz])=>{
    const p = new THREE.Mesh(new THREE.BoxGeometry(.025,.045,.002), panelMat);
    p.position.set(px,py,pz); issGroup.add(p);
  });

  // ISS glow
  const issGlow = new THREE.Mesh(
    new THREE.SphereGeometry(.018,16,16),
    new THREE.MeshBasicMaterial({color:0x00e5ff, transparent:true, opacity:.6})
  );
  issGroup.add(issGlow);

  // ── ISS Trail ──
  const TRAIL_LEN = 150;
  const trailPts = [];
  const trailGeo = new THREE.BufferGeometry();
  const trailLine = new THREE.Line(
    trailGeo,
    new THREE.LineBasicMaterial({color:0x00e5ff, transparent:true, opacity:.25})
  );
  scene.add(trailLine);

  // ── Lights ──
  scene.add(new THREE.AmbientLight(0x112244, .9));
  const sun = new THREE.DirectionalLight(0xffffff, 1.4);
  sun.position.set(5,3,5);
  scene.add(sun);

  // ── Mouse/touch interaction ──
  let drag=false, ax=.25, ay=0, tx2=.25, ty2=0, px=0, py=0;
  let autoRot = true;

  canvas.addEventListener('mousedown', e=>{drag=true;autoRot=false;px=e.clientX;py=e.clientY;});
  window.addEventListener('mouseup', ()=>drag=false);
  canvas.addEventListener('mousemove', e=>{
    if(!drag)return;
    ty2+=(e.clientX-px)*.012; tx2+=(e.clientY-py)*.008;
    tx2=Math.max(-.7,Math.min(.7,tx2));
    px=e.clientX; py=e.clientY;
  });
  canvas.addEventListener('wheel', e=>{
    cam.position.z = Math.max(2,Math.min(7,cam.position.z+e.deltaY*.004));
  },{passive:true});
  let lt=null;
  canvas.addEventListener('touchstart', e=>{drag=true;autoRot=false;lt={x:e.touches[0].clientX,y:e.touches[0].clientY};},{passive:true});
  canvas.addEventListener('touchend', ()=>{drag=false;lt=null;},{passive:true});
  canvas.addEventListener('touchmove', e=>{
    if(!lt)return;
    ty2+=(e.touches[0].clientX-lt.x)*.012;
    tx2+=(e.touches[0].clientY-lt.y)*.008;
    lt={x:e.touches[0].clientX,y:e.touches[0].clientY};
  },{passive:true});

  // ── RENDER LOOP ──
  let glowPhase = 0;
  (function tick() {
    requestAnimationFrame(tick);
    glowPhase += .04;

    if(autoRot) ty2 += .004;
    ax += (tx2-ax)*.08;
    ay += (ty2-ay)*.08;

    earth.rotation.x = ax;
    earth.rotation.y = ay;
    orbitalPlane.rotation.x = ax;
    orbitalPlane.rotation.y = ay;

    // Position ISS from lat/lon
    const {lat,lon} = STATE.iss;
    const phi = (90-lat)*Math.PI/180;
    const theta = (lon+180)*Math.PI/180;
    const r = ORB_R;

    // Raw position
    const ix = -r*Math.sin(phi)*Math.cos(theta);
    const iy =  r*Math.cos(phi);
    const iz =  r*Math.sin(phi)*Math.sin(theta);

    // Apply scene rotation
    const rm = new THREE.Matrix4()
      .makeRotationY(ay)
      .multiply(new THREE.Matrix4().makeRotationX(ax));
    const ipos = new THREE.Vector3(ix,iy,iz).applyMatrix4(rm);

    issGroup.position.copy(ipos);
    issGroup.lookAt(new THREE.Vector3(0,0,0).applyMatrix4(rm));
    issGlow.material.opacity = .4+.3*Math.sin(glowPhase);

    // Trail
    if(trailPts.length===0 || ipos.distanceTo(trailPts[trailPts.length-1])>.001){
      trailPts.push(ipos.clone());
      if(trailPts.length>TRAIL_LEN) trailPts.shift();
      if(trailPts.length>1) trailGeo.setFromPoints(trailPts);
    }

    renderer.render(scene, cam);
  })();
}

// ─────────────────────────────────────────────
//  HANDMADE OPENSKY NETWORK API
// ─────────────────────────────────────────────
const SKYAPI = {
  BASE: 'https://opensky-network.org/api',

  async _get(ep) {
    // Try direct first, then proxy
    const url = this.BASE + ep;
    try {
      const r = await fetch(url, {signal: AbortSignal.timeout(12000)});
      if (!r.ok) throw new Error('HTTP ' + r.status);
      return r.json();
    } catch(e) {
      try {
        const proxy = 'https://api.allorigins.win/get?url=' + encodeURIComponent(url);
        const r2 = await fetch(proxy, {signal: AbortSignal.timeout(15000)});
        const w = await r2.json();
        return JSON.parse(w.contents);
      } catch(e2) {
        // Second proxy fallback
        const p2 = 'https://corsproxy.io/?' + encodeURIComponent(url);
        const r3 = await fetch(p2, {signal: AbortSignal.timeout(15000)});
        return r3.json();
      }
    }
  },

  // GET /states/all — returns all live aircraft states
  // State vector indices:
  // 0:icao24 1:callsign 2:country 3:time_pos 4:last_contact
  // 5:lon 6:lat 7:baro_alt(m) 8:on_ground 9:velocity(m/s)
  // 10:track 11:vertical_rate(m/s) 12:sensors 13:geo_alt
  // 14:squawk 15:spi 16:pos_source
  async allStates(bbox) {
    let ep = '/states/all';
    if (bbox) ep += `?lamin=${bbox.lamin}&lomin=${bbox.lomin}&lamax=${bbox.lamax}&lomax=${bbox.lomax}`;
    const d = await this._get(ep);
    if (!d || !d.states) return [];
    return d.states.map(s => ({
      icao24:       s[0],
      callsign:     (s[1] || '').trim() || null,
      country:      s[2],
      lon:          s[5],
      lat:          s[6],
      alt:          s[7],   // metres baro
      onGround:     s[8],
      velocity:     s[9],   // m/s
      track:        s[10],
      vertRate:     s[11],  // m/s negative=descending
      geoAlt:       s[13],
      squawk:       s[14],
      spi:          s[15],
    }));
  },
};

// ─────────────────────────────────────────────
//  ANOMALY ENGINE
// ─────────────────────────────────────────────
const ANOMALY = {
  events:     [],
  maxEvents:  60,
  counts:     { critical:0, high:0, medium:0, low:0 },
  radarDots:  [],      // {x,y,sev,phase}
  totalScanned: 0,

  SQUAWK_DEFS: {
    '7500': { label:'SQUAWK 7500 — HIJACK',     sev:'critical', sevClass:'sev-critical', typeClass:'' },
    '7600': { label:'SQUAWK 7600 — RADIO FAIL', sev:'high',     sevClass:'sev-high',     typeClass:'high' },
    '7700': { label:'SQUAWK 7700 — EMERGENCY',  sev:'high',     sevClass:'sev-high',     typeClass:'high' },
  },

  classify(ac) {
    const flags = [];
    // Emergency squawk codes
    if (ac.squawk && this.SQUAWK_DEFS[ac.squawk]) {
      const def = this.SQUAWK_DEFS[ac.squawk];
      flags.push({...def, squawk: ac.squawk});
    }
    // Rapid descent (> 3000 ft/min ~15 m/s) not on ground, not too high alt
    if (!ac.onGround && ac.vertRate !== null && ac.vertRate < -15 && ac.alt !== null && ac.alt > 300 && ac.alt < 12000) {
      const fpm = Math.round(ac.vertRate * 196.85);
      flags.push({
        label: `RAPID DESCENT — ${fpm.toLocaleString()} ft/min`,
        sev:'medium', sevClass:'sev-medium', typeClass:'medium', squawk:null,
      });
    }
    // Extreme speed at low altitude
    if (!ac.onGround && ac.velocity !== null && ac.velocity > 350 && ac.alt !== null && ac.alt < 3000) {
      flags.push({
        label: `EXTREME SPEED — ${Math.round(ac.velocity * 1.944)} kts at low alt`,
        sev:'low', sevClass:'sev-low', typeClass:'low', squawk:null,
      });
    }
    // SPI ident
    if (ac.spi && !ac.onGround) {
      flags.push({
        label: 'SPI IDENT ACTIVE — ATC request signal',
        sev:'low', sevClass:'sev-low', typeClass:'low', squawk:null,
      });
    }
    return flags;
  },

  ingest(aircraft) {
    const now = new Date();
    const timeStr = now.toUTCString().slice(17,25);
    this.totalScanned = aircraft.length;
    this.counts = { critical:0, high:0, medium:0, low:0 };
    this.radarDots = [];

    const newEvents = [];
    // Dedup: only flag if we haven't seen this ICAO + squawk recently
    const seen = new Set(this.events.slice(0,20).map(e => e.key));

    aircraft.forEach(ac => {
      const flags = this.classify(ac);
      flags.forEach(flag => {
        this.counts[flag.sev]++;
        const key = (ac.icao24 || '') + flag.sev + (flag.squawk||'x');
        if (!seen.has(key)) {
          newEvents.push({ ...flag, ac, time:timeStr, key });
          seen.add(key);
        }
        // Add radar dot (random position if no lat/lon, otherwise mapped)
        const angle = Math.random() * Math.PI * 2;
        const dist  = Math.random() * .75 + .15;
        this.radarDots.push({
          x: 0.5 + dist * Math.cos(angle) * 0.5,
          y: 0.5 + dist * Math.sin(angle) * 0.5,
          sev: flag.sev,
          phase: Math.random() * Math.PI * 2,
        });
      });
    });

    // Prepend newest events
    this.events = [...newEvents, ...this.events].slice(0, this.maxEvents);
    return newEvents.length;
  },

  renderFeed() {
    const feed = el('anomaly-feed');
    el('cnt-critical').textContent = this.counts.critical;
    el('cnt-high').textContent     = this.counts.high;
    el('cnt-medium').textContent   = this.counts.medium;
    el('cnt-scanned').textContent  = this.totalScanned.toLocaleString();
    el('feed-count').textContent   = this.events.length + ' EVENTS';

    const total = this.counts.critical + this.counts.high + this.counts.medium;
    const badge = el('anomaly-badge');
    if (total > 0) {
      badge.style.display = 'inline-flex';
      badge.textContent = total > 99 ? '99+' : total;
    } else {
      badge.style.display = 'none';
    }

    if (this.events.length === 0) {
      feed.innerHTML = '<div class="loading-msg" style="color:var(--green)">✓ NO ANOMALIES DETECTED — AIRSPACE CLEAR</div>';
      return;
    }

    feed.innerHTML = this.events.map(ev => {
      const ac = ev.ac;
      const callsign = ac.callsign || ac.icao24 || 'UNKNOWN';
      const altFt = ac.alt ? Math.round(ac.alt * 3.281).toLocaleString() + ' ft' : '—';
      const spdKts = ac.velocity ? Math.round(ac.velocity * 1.944).toLocaleString() + ' kts' : '—';
      const sqTag = ev.squawk ? `<span class="squawk-badge sq-${ev.squawk}">${ev.squawk}</span>` : '';
      return `
        <div class="a-event">
          <div class="a-sev ${ev.sevClass}"></div>
          <div class="a-body">
            <div class="a-type ${ev.typeClass}">${ev.label}</div>
            <div class="a-flight">${callsign}${sqTag}</div>
            <div class="a-detail">
              ${ac.country || 'UNKNOWN'} &nbsp;·&nbsp;
              ${altFt} &nbsp;·&nbsp; ${spdKts}
              ${ac.lat && ac.lon ? ` &nbsp;·&nbsp; ${ac.lat.toFixed(2)}°,${ac.lon.toFixed(2)}°` : ''}
            </div>
          </div>
          <div class="a-time">${ev.time}</div>
        </div>`;
    }).join('');
  },
};

// ─────────────────────────────────────────────
//  RADAR ANIMATION
// ─────────────────────────────────────────────
function initRadar() {
  const canvas = el('radar-canvas');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const cx = W/2, cy = H/2, R = W/2 - 8;
  let angle = 0;

  const SEV_COLORS = {
    critical: '#ff3366',
    high:     '#ff6600',
    medium:   '#ffaa00',
    low:      '#00e5ff',
  };

  // Fade trail (persistent)
  const trailCanvas = document.createElement('canvas');
  trailCanvas.width = W; trailCanvas.height = H;
  const tCtx = trailCanvas.getContext('2d');

  (function tick() {
    requestAnimationFrame(tick);
    angle += 0.025;

    // Clear
    ctx.clearRect(0,0,W,H);

    // Background
    ctx.fillStyle = '#030e08';
    ctx.beginPath(); ctx.arc(cx,cy,R,0,Math.PI*2); ctx.fill();

    // Fade trail on trail canvas
    tCtx.fillStyle = 'rgba(3,14,8,0.04)';
    tCtx.fillRect(0,0,W,H);

    // Draw sweep on trail canvas
    const grd = tCtx.createConicalGradient ? null : null;
    tCtx.save();
    tCtx.translate(cx,cy);
    tCtx.rotate(angle);
    const sweep = tCtx.createLinearGradient(0,0,R,0);
    sweep.addColorStop(0,'rgba(0,255,80,0.35)');
    sweep.addColorStop(1,'rgba(0,255,80,0)');
    tCtx.beginPath();
    tCtx.moveTo(0,0);
    tCtx.arc(0,0,R,-0.3,0);
    tCtx.closePath();
    tCtx.fillStyle = sweep;
    tCtx.fill();
    tCtx.restore();

    // Blit trail
    ctx.drawImage(trailCanvas,0,0);

    // Grid rings
    [.25,.5,.75,1].forEach(f => {
      ctx.beginPath(); ctx.arc(cx,cy,R*f,0,Math.PI*2);
      ctx.strokeStyle = `rgba(0,229,255,${f===1?.25:.1})`; ctx.lineWidth=.7; ctx.stroke();
    });

    // Crosshair
    ctx.strokeStyle='rgba(0,229,255,0.12)'; ctx.lineWidth=.5;
    ctx.beginPath(); ctx.moveTo(cx-R,cy); ctx.lineTo(cx+R,cy); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(cx,cy-R); ctx.lineTo(cx,cy+R); ctx.stroke();

    // Sweep line
    ctx.save(); ctx.translate(cx,cy); ctx.rotate(angle);
    ctx.beginPath(); ctx.moveTo(0,0); ctx.lineTo(R,0);
    ctx.strokeStyle='rgba(0,255,80,0.8)'; ctx.lineWidth=1.2; ctx.stroke();
    ctx.restore();

    // Dots
    ANOMALY.radarDots.forEach(d => {
      d.phase += .04;
      const x = d.x * W, y = d.y * H;
      const col = SEV_COLORS[d.sev] || '#00e5ff';
      const pulse = .5 + .5*Math.sin(d.phase);
      ctx.beginPath(); ctx.arc(x,y,2+pulse*1.5,0,Math.PI*2);
      ctx.fillStyle = col; ctx.fill();
      ctx.beginPath(); ctx.arc(x,y,5+pulse*3,0,Math.PI*2);
      ctx.strokeStyle=col.replace(')',`,${.3*pulse})`).replace('rgb','rgba');
      ctx.lineWidth=1; ctx.stroke();
    });

    // Center
    ctx.beginPath(); ctx.arc(cx,cy,3,0,Math.PI*2);
    ctx.fillStyle='#00ff55'; ctx.fill();
  })();
}

// ─────────────────────────────────────────────
//  FETCH ANOMALIES (OpenSky)
// ─────────────────────────────────────────────
async function fetchAnomalies() {
  const dot  = el('scan-dot');
  const lbl  = el('scan-label');
  const last = el('last-scan');

  dot.className = 'scan-pulse amber';
  lbl.textContent = 'SCANNING GLOBAL AIRSPACE...';

  try {
    // Fetch global states (no bbox = world)
    const aircraft = await SKYAPI.allStates();
    el('radar-aircraft-count').textContent = aircraft.length.toLocaleString() + ' AIRCRAFT';

    const newCount = ANOMALY.ingest(aircraft);
    ANOMALY.renderFeed();

    dot.className = 'scan-pulse ok';
    lbl.textContent = 'SCAN COMPLETE — ' + aircraft.length.toLocaleString() + ' AIRCRAFT MONITORED';
    last.textContent = 'LAST: ' + new Date().toUTCString().slice(17,25);

    if (newCount > 0 && Notification?.permission === 'granted') {
      if (ANOMALY.counts.critical > 0) {
        new Notification('🚨 Critical Air Anomaly!', {
          body: `${ANOMALY.counts.critical} critical squawk(s) detected in global airspace.`
        });
      }
    }
  } catch(e) {
    console.warn('OpenSky fetch error:', e);
    dot.className = 'scan-pulse';
    dot.style.background = 'var(--red)';
    lbl.textContent = 'OPENSKY TEMPORARILY UNAVAILABLE — RETRYING';
    last.textContent = 'ERR: ' + new Date().toUTCString().slice(17,25);
    // Still render whatever we have
    ANOMALY.renderFeed();
  }
}

// ═══════════════════════════════════════════════════════
//  SOUND SYSTEM — Web Audio API (no external files)
// ═══════════════════════════════════════════════════════
const SOUND = {
  ctx: null,
  muted: false,

  _ctx() {
    if (!this.ctx) this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    if (this.ctx.state === 'suspended') this.ctx.resume();
    return this.ctx;
  },

  _tone(freq, type, gainVal, duration, startTime) {
    const ctx = this._ctx();
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.connect(gain); gain.connect(ctx.destination);
    osc.type = type || 'sine';
    osc.frequency.setValueAtTime(freq, startTime);
    gain.gain.setValueAtTime(gainVal, startTime);
    gain.gain.exponentialRampToValueAtTime(0.001, startTime + duration);
    osc.start(startTime); osc.stop(startTime + duration);
  },

  critical() {
    if (this.muted) return;
    const ctx = this._ctx(); const now = ctx.currentTime;
    for (let i = 0; i < 4; i++) {
      const t = now + i * 0.35;
      const osc = ctx.createOscillator(); const gain = ctx.createGain();
      osc.connect(gain); gain.connect(ctx.destination);
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(440, t);
      osc.frequency.linearRampToValueAtTime(1100, t + 0.28);
      gain.gain.setValueAtTime(0.28, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.3);
      osc.start(t); osc.stop(t + 0.31);
    }
  },

  high() {
    if (this.muted) return;
    const ctx = this._ctx(); const now = ctx.currentTime;
    [880, 1100, 880].forEach((f,i) => this._tone(f,'square',0.15,0.18,now+i*0.2));
  },

  medium() {
    if (this.muted) return;
    const ctx = this._ctx(); const now = ctx.currentTime;
    [660,660].forEach((f,i) => this._tone(f,'sine',0.12,0.14,now+i*0.22));
  },

  low() {
    if (this.muted) return;
    const ctx = this._ctx();
    this._tone(520,'sine',0.08,0.12,ctx.currentTime);
  },

  ebs() {
    if (this.muted) return;
    const ctx = this._ctx(); const now = ctx.currentTime;
    this._tone(853,'sine',0.2,0.9,now);
    this._tone(960,'sine',0.2,0.9,now);
    this._tone(853,'sine',0.2,0.9,now+1.0);
    this._tone(960,'sine',0.2,0.9,now+1.0);
  },

  iss() {
    if (this.muted) return;
    const ctx = this._ctx(); const now = ctx.currentTime;
    this._tone(1047,'sine',0.18,0.6,now);
    this._tone(1319,'sine',0.14,0.5,now+0.25);
    this._tone(1568,'sine',0.10,0.4,now+0.5);
  },

  init() {
    const btn = el('sound-toggle');
    btn.addEventListener('click', () => {
      this.muted = !this.muted;
      btn.textContent = this.muted ? '🔇' : '🔊';
      btn.className = this.muted ? 'muted' : '';
      btn.id = 'sound-toggle';
      if (!this.muted) this.low();
    });
    document.addEventListener('click', () => this._ctx(), { once:true });
  },
};

// ═══════════════════════════════════════════════════════
//  EBS TICKER — NWS Weather Alerts API
// ═══════════════════════════════════════════════════════
const EBS = {
  alerts:[], current:0, timer:null, dismissed:false,

  SEVERITY: {
    Extreme:  { cls:'ebs-alert', label:'⚡ EXTREME ALERT' },
    Severe:   { cls:'ebs-alert', label:'🚨 SEVERE ALERT' },
    Moderate: { cls:'ebs-warn',  label:'⚠ WARNING' },
    Minor:    { cls:'ebs-info',  label:'ℹ ADVISORY' },
    Unknown:  { cls:'ebs-info',  label:'📡 BROADCAST' },
  },

  async fetch(lat, lon) {
    try {
      const url = `https://api.weather.gov/alerts/active?point=${lat.toFixed(4)},${lon.toFixed(4)}`;
      const r = await fetch(url, {
        headers:{'User-Agent':'OrbitWatch/4.0 (@MrCodeBlood)'},
        signal:AbortSignal.timeout(10000),
      });
      if (!r.ok) throw new Error('NWS '+r.status);
      const d = await r.json();
      this.alerts = (d.features||[]).map(f=>({
        headline: f.properties.headline || f.properties.event || 'Weather Alert',
        severity: f.properties.severity || 'Unknown',
        area:     f.properties.areaDesc || 'Your Area',
        desc:     (f.properties.description||'').slice(0,200),
      }));
      this.renderAlertList();
      if (this.alerts.length > 0) {
        this.show();
        const sev = this.alerts[0].severity;
        if (sev==='Extreme'||sev==='Severe') { SOUND.ebs(); SOUND.critical(); }
        else { SOUND.medium(); }
      } else {
        this.showClear();
      }
    } catch(e) {
      this.showFallback();
      this.renderAlertList();
    }
  },

  _setText(text, durationMs) {
    const t = el('ebs-ticker-text');
    if (!t) return;
    t.textContent = text;
    t.style.animationDuration = Math.max(14000, durationMs) + 'ms';
  },

  show() {
    if (!el('ebs-bar')||this.dismissed) return;
    this.current=0; this.rotate();
    el('ebs-bar').classList.add('visible');
    document.body.classList.add('ebs-active');
  },

  showClear() {
    if (!el('ebs-bar')) return;
    el('ebs-bar').className = 'ebs-info visible';
    el('ebs-type').textContent = '✓ ALL CLEAR';
    this._setText('  ✓ NO ACTIVE WEATHER ALERTS IN YOUR AREA — ORBIT WATCH MONITORING — @MrCodeBlood  ', 18000);
    document.body.classList.add('ebs-active');
  },

  showFallback() {
    if (!el('ebs-bar')||this.dismissed) return;
    el('ebs-bar').className = 'ebs-info visible';
    el('ebs-type').textContent = '📡 ORBIT WATCH';
    const msgs = [
      'ORBIT WATCH v4.0 — CREATED BY @MrCodeBlood — ISS LIVE TRACKING ACTIVE — GLOBAL AIRSPACE MONITORING ENABLED — ALL SYSTEMS ONLINE',
      'EMERGENCY SQUAWK MONITORING: 7500 HIJACK · 7600 RADIO FAILURE · 7700 GENERAL EMERGENCY — OPENSKY NETWORK LIVE',
      'NOAA WEATHER ALERTS · NWS EMERGENCY BROADCAST · RADIO SCANNER · SATELLITE ORBITAL TRACKING — 40-HOUR CONTINUOUS OPERATION',
    ];
    let idx=0;
    const roll = () => { this._setText('  '+msgs[idx++%msgs.length]+'  ', msgs[(idx-1)%msgs.length].length*80); };
    roll();
    clearInterval(this.timer);
    this.timer = setInterval(roll, 20000);
    document.body.classList.add('ebs-active');
    el('ebs-bar').classList.add('visible');
  },

  rotate() {
    if (!this.alerts.length) return;
    const a = this.alerts[this.current%this.alerts.length];
    const sev = this.SEVERITY[a.severity] || this.SEVERITY.Unknown;
    el('ebs-bar').className = sev.cls + ' visible';
    el('ebs-type').textContent = sev.label;
    const txt = `  ⚡ ${a.headline}  ·  ${a.area}  ·  ${a.desc}  `;
    this._setText(txt, txt.length * 75);
    this.current=(this.current+1)%this.alerts.length;
    clearInterval(this.timer);
    this.timer = setInterval(()=>this.rotate(), Math.max(16000,txt.length*75));
  },

  renderAlertList() {
    const list = el('nws-alert-list'); if (!list) return;
    if (!this.alerts.length) {
      list.innerHTML='<div class="loading-msg" style="color:var(--green)">✓ NO ACTIVE ALERTS IN YOUR AREA</div>';
      return;
    }
    list.innerHTML = this.alerts.map(a => {
      const sev = this.SEVERITY[a.severity]||this.SEVERITY.Unknown;
      const col = sev.cls==='ebs-alert'?'var(--red)':sev.cls==='ebs-warn'?'var(--amber)':'var(--cyan2)';
      return `<div style="padding:10px 14px;border-bottom:1px solid rgba(11,61,106,.4)">
        <div style="font-size:9px;letter-spacing:2px;color:${col};margin-bottom:3px">${sev.label} · ${a.severity.toUpperCase()}</div>
        <div style="font-size:11px;color:var(--bright)">${a.headline}</div>
        <div style="font-size:9px;color:var(--dim);margin-top:2px">${a.area}</div>
      </div>`;
    }).join('');
  },

  init() {
    el('ebs-dismiss').addEventListener('click', () => {
      this.dismissed=true;
      el('ebs-bar').classList.remove('visible');
      document.body.classList.remove('ebs-active');
      clearInterval(this.timer);
    });
    this.showFallback();
  },
};

// ═══════════════════════════════════════════════════════
//  RADIO SCANNER — Radio Browser API
// ═══════════════════════════════════════════════════════
const RADIO = {
  stations:[], idx:-1, playing:false, vizBars:[], vizTimer:null,
  EMERG_KW: ['emergency','alert','weather','noaa','news','scanner','police','fire','eas'],

  async fetch() {
    const tags = ['news','emergency','scanner','weather'];
    const results = await Promise.allSettled(tags.map(t => this._fetchTag(t)));
    const seen = new Set(); const all = [];
    results.forEach(r => {
      if (r.status==='fulfilled'&&Array.isArray(r.value)) {
        r.value.forEach(s => { if(!seen.has(s.stationuuid)&&s.url_resolved){seen.add(s.stationuuid);all.push(s);} });
      }
    });
    all.sort((a,b) => (this._isEmerg(b)?1:0)-(this._isEmerg(a)?1:0)||(b.votes||0)-(a.votes||0));
    this.stations = all.slice(0,60);
    this.renderList();
  },

  async _fetchTag(tag) {
    const servers = ['https://de1.api.radio-browser.info','https://nl1.api.radio-browser.info','https://at1.api.radio-browser.info'];
    for (const srv of servers) {
      try {
        const r = await fetch(`${srv}/json/stations/search?tag=${encodeURIComponent(tag)}&limit=30&hidebroken=true&order=votes&reverse=true`,
          {signal:AbortSignal.timeout(8000)});
        if (r.ok) return r.json();
      } catch(e) {}
    }
    return [];
  },

  _isEmerg(s) { const t=((s.name||'')+' '+(s.tags||'')).toLowerCase(); return this.EMERG_KW.some(k=>t.includes(k)); },

  _fakeFreq(s) {
    const h=[...(s.name||'')].reduce((a,c)=>a+c.charCodeAt(0),0);
    return h%5===0 ? ((h%136)*10+540).toLocaleString()+' kHz AM' : (88.1+(h%900)/10).toFixed(1)+' MHz FM';
  },

  renderList() {
    const list = el('station-list');
    if (!this.stations.length){list.innerHTML='<div class="loading-msg">NO STATIONS FOUND — CHECK CONNECTION</div>';return;}
    list.innerHTML = this.stations.map((s,i) => {
      const em = this._isEmerg(s);
      return `<div class="station-item${em?' emergency':''}${i===this.idx?' active':''}" onclick="RADIO.select(${i})">
        <div class="si-icon">${em?'⚡':'📻'}</div>
        <div class="si-info">
          <div class="si-name">${s.name||'Unknown'}${em?'<span class="emerg-badge">EMERG</span>':''}</div>
          <div class="si-meta">${s.country||''} · ${(s.tags||'').split(',').slice(0,3).join(' · ')}</div>
        </div>
        <div class="si-freq${em?' red':''}">${this._fakeFreq(s)}</div>
      </div>`;
    }).join('');
  },

  select(i) {
    this.idx=i; const s=this.stations[i]; if(!s)return;
    el('np-name').textContent = s.name||'Unknown Station';
    el('np-freq').textContent = this._fakeFreq(s);
    el('np-tags').textContent = (s.tags||'news · radio').split(',').slice(0,4).join(' · ').toUpperCase();
    el('np-icon').textContent = this._isEmerg(s)?'⚡':'📡';
    document.querySelectorAll('.station-item').forEach((e,j)=>e.classList.toggle('active',j===i));
    this.play();
  },

  play() {
    const s=this.stations[this.idx]; if(!s)return;
    const audio=el('radio-audio');
    audio.src=s.url_resolved;
    audio.volume=parseFloat(el('vol-slider').value);
    audio.play().then(()=>{
      this.playing=true;
      el('play-btn').textContent='⏸'; el('play-btn').classList.add('active');
      el('np-icon').classList.add('playing'); this.startViz();
    }).catch(()=>{
      el('np-name').textContent='⚠ STREAM UNAVAILABLE — TRY NEXT';
      el('play-btn').classList.remove('active');
    });
  },

  pause() {
    el('radio-audio').pause(); this.playing=false;
    el('play-btn').textContent='▶'; el('play-btn').classList.remove('active');
    el('np-icon').classList.remove('playing'); this.stopViz();
  },

  toggle() { if(this.playing){this.pause();return;} if(this.idx<0&&this.stations.length){this.select(0);return;} this.play(); },
  prev()   { this.select(((this.idx||0)-1+this.stations.length)%this.stations.length); },
  next()   { this.select(((this.idx||0)+1)%this.stations.length); },

  startViz() {
    clearInterval(this.vizTimer);
    this.vizTimer = setInterval(()=>{
      this.vizBars.forEach(b=>{
        const h=Math.max(2,Math.random()*28);
        b.style.height=h+'px'; b.style.opacity=0.3+(h/28)*0.7;
        b.style.background=h>20?'var(--red)':h>12?'var(--amber)':'var(--cyan)';
      });
    },80);
  },

  stopViz() {
    clearInterval(this.vizTimer);
    this.vizBars.forEach(b=>{b.style.height='2px';b.style.opacity='.15';b.style.background='var(--cyan)';});
  },

  init() {
    el('play-btn').addEventListener('click',()=>this.toggle());
    el('prev-btn').addEventListener('click',()=>this.prev());
    el('next-btn').addEventListener('click',()=>this.next());
    el('vol-slider').addEventListener('input',e=>{
      const v=parseFloat(e.target.value);
      el('radio-audio').volume=v; el('vol-pct').textContent=Math.round(v*100)+'%';
    });
    el('radio-audio').addEventListener('error',()=>{if(this.playing)setTimeout(()=>this.next(),2000);});
    el('radio-audio').addEventListener('ended',()=>this.next());
    // Build viz bars
    const wrap=el('freq-viz');
    for(let i=0;i<28;i++){
      const b=document.createElement('div'); b.className='freq-bar';
      b.style.height='2px';b.style.opacity='.15';
      wrap.appendChild(b); this.vizBars.push(b);
    }
    this.fetch();
  },
};

// ═══════════════════════════════════════════════════════
//  40-HOUR KEEPALIVE
// ═══════════════════════════════════════════════════════
const KEEPALIVE = {
  startTime: Date.now(),
  wakeLock: null,

  async requestWakeLock() {
    try {
      if ('wakeLock' in navigator) {
        this.wakeLock = await navigator.wakeLock.request('screen');
        this.wakeLock.addEventListener('release', ()=>{
          el('keepalive-status').textContent='RE-ACQUIRING';
          el('keepalive-status').style.color='var(--amber)';
          setTimeout(()=>this.requestWakeLock(),3000);
        });
        el('keepalive-status').textContent='ACTIVE ✓';
        el('keepalive-status').style.color='var(--green)';
      } else {
        el('keepalive-status').textContent='LIMITED (no WakeLock API)';
        el('keepalive-status').style.color='var(--amber)';
      }
    } catch(e) {
      el('keepalive-status').textContent='AUDIO-ONLY MODE';
      el('keepalive-status').style.color='var(--amber)';
    }
  },

  // Inaudible audio node — prevents timer throttling in background tabs
  startSilentAudio() {
    try {
      const ctx = SOUND._ctx();
      const buf = ctx.createBuffer(1, ctx.sampleRate * 3, ctx.sampleRate);
      const src = ctx.createBufferSource();
      src.buffer=buf; src.loop=true;
      const gain = ctx.createGain(); gain.gain.value=0.00001;
      src.connect(gain); gain.connect(ctx.destination);
      src.start();
    } catch(e) {}
  },

  bindVisibility() {
    document.addEventListener('visibilitychange',()=>{
      if(document.visibilityState==='visible') this.requestWakeLock();
    });
  },

  startHeartbeat() {
    // Extra pings every 30 min to keep data fresh
    setInterval(()=>{ fetchISS(); fetchCrew(); }, 30*60*1000);
    // NWS refresh every 15 min
    if(STATE.user.lat) setInterval(()=>EBS.fetch(STATE.user.lat,STATE.user.lon), 15*60*1000);
  },

  updateUptime() {
    const elapsed = Math.floor((Date.now()-this.startTime)/1000);
    const h=String(Math.floor(elapsed/3600)).padStart(2,'0');
    const m=String(Math.floor((elapsed%3600)/60)).padStart(2,'0');
    const s=String(elapsed%60).padStart(2,'0');
    const d=el('uptime-display');
    if(d){
      d.textContent=`${h}:${m}:${s}`;
      if(elapsed>39*3600) d.style.color='var(--amber)';
      if(elapsed>=40*3600) d.style.color='var(--red)';
    }
  },

  async init() {
    await this.requestWakeLock();
    this.bindVisibility();
    setInterval(()=>this.updateUptime(),1000);
    // Start heartbeat after 1 min
    setTimeout(()=>this.startHeartbeat(),60000);
    // Silent audio after first user interaction
    document.addEventListener('click',()=>this.startSilentAudio(),{once:true});
  },
};

async function fetchISS() {
  try {
    const pos = await API.issPosition();
    STATE.iss = pos;
    updateUI(pos.lat, pos.lon, pos.ts);
    updateMapISS(pos.lat, pos.lon);
    // Sound on ISS overhead
    const wasNearby = STATE.issNearby || false;
    if (STATE.user.lat !== null) {
      const dist = Math.round(haversine(STATE.user.lat, STATE.user.lon, pos.lat, pos.lon));
      if (dist < 1500 && !wasNearby) { SOUND.iss(); STATE.issNearby = true; }
      else if (dist >= 1500) { STATE.issNearby = false; }
    }
    checkAlert(pos.lat, pos.lon);
  } catch(e) {
    console.warn('ISS fetch error:', e);
  }
}

async function fetchCrew() {
  try {
    const {count, people} = await API.astronauts();
    renderCrew(people, count);
  } catch(e) {
    el('crew-list').innerHTML = '<div class="loading-msg">DATA UNAVAILABLE</div>';
    el('h-people').textContent = '?';
  }
}

// ─────────────────────────────────────────────
//  BOOT
// ─────────────────────────────────────────────
async function boot() {
  Notification?.requestPermission?.();
  SOUND.init();
  EBS.init();
  initStars();
  startClock();
  initGeo();
  initRadar();
  await initMap();
  init3D();
  await fetchISS();
  await fetchCrew();
  RADIO.init();
  KEEPALIVE.init();
  fetchAnomalies();

  // Patch anomaly ingest to trigger sounds
  const _origIngest = ANOMALY.ingest.bind(ANOMALY);
  ANOMALY.ingest = function(aircraft) {
    const prevC = this.counts.critical||0, prevH = this.counts.high||0;
    const n = _origIngest(aircraft);
    if (n > 0) {
      if (this.counts.critical > prevC)      SOUND.critical();
      else if (this.counts.high > prevH)     SOUND.high();
      else if (this.counts.medium > 0)       SOUND.medium();
      else                                   SOUND.low();
    }
    return n;
  };

  // Fetch NWS alerts after geo resolves (retry loop)
  const tryNWS = () => {
    if (STATE.user.lat !== null) {
      EBS.fetch(STATE.user.lat, STATE.user.lon);
    } else {
      setTimeout(tryNWS, 2000);
    }
  };
  setTimeout(tryNWS, 3000);

  setInterval(fetchISS,        CFG.REFRESH_MS);
  setInterval(fetchCrew,       5*60*1000);
  setInterval(fetchAnomalies,  60*1000);
}

window.addEventListener('load', boot);
</script>
</body>
</html>
