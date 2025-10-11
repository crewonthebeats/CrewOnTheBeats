
![WhatsApp Image 2025-05-16 at 01 19 02 (2)](https://github.com/user-attachments/assets/60d1b72e-efb4-4959-8b57-b166d39ba389)



<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>CrewOnTheBeats — Web Player</title>
  <meta name="description" content="Reproductor web simple para alojar en GitHub Pages. Soporta playlist JSON con URLs a archivos de audio." />
  <style>
    :root{--bg:#0f1724;--card:#0b1220;--muted:#9aa4b2;--accent:#ff5a5f}
    *{box-sizing:border-box}
    body{margin:0;font-family:Inter,system-ui,Arial,Helvetica,sans-serif;background:linear-gradient(180deg,#071024 0%,#0b1626 100%);color:#e6eef6;min-height:100vh;display:flex;align-items:center;justify-content:center;padding:24px}
    .player{width:100%;max-width:900px;background:linear-gradient(180deg,rgba(255,255,255,.02),rgba(0,0,0,.04));border-radius:12px;padding:18px;box-shadow:0 8px 28px rgba(2,6,23,.6);}
    .top{display:flex;gap:16px;align-items:center}
    .cover{width:120px;height:120px;border-radius:8px;background:linear-gradient(135deg,var(--accent),#7a4bff);display:flex;align-items:center;justify-content:center;font-weight:700;font-size:20px}
    .meta{flex:1}
    .meta h1{margin:0;font-size:20px}
    .meta p{margin:6px 0 0;color:var(--muted);font-size:13px}
    .controls{display:flex;align-items:center;gap:12px;margin-top:14px}
    button{background:transparent;border:0;color:inherit;cursor:pointer;padding:8px;border-radius:8px}
    .big{font-size:22px}
    .progress{width:100%;height:8px;background:rgba(255,255,255,.06);border-radius:6px;overflow:hidden;margin-top:12px}
    .progress > div{height:100%;background:linear-gradient(90deg,var(--accent),#7a4bff);width:0}
    .playlist{margin-top:16px;max-height:260px;overflow:auto;border-top:1px solid rgba(255,255,255,.03);padding-top:12px}
    .track{padding:10px;border-radius:8px;display:flex;gap:10px;align-items:center;cursor:pointer}
    .track:hover{background:rgba(255,255,255,.02)}
    .track.active{background:linear-gradient(90deg,rgba(255,90,95,.07),rgba(122,75,255,.04));}
    .track .title{font-size:14px}
    .footer{display:flex;align-items:center;justify-content:space-between;margin-top:12px;color:var(--muted);font-size:13px}
    input[type=file]{display:none}
    .upload-btn{background:rgba(255,255,255,.03);padding:8px 10px;border-radius:8px}
    @media (max-width:620px){.top{flex-direction:column;align-items:flex-start}.cover{width:100%;height:120px}}
  </style>
</head>
<body>
  <div class="player" id="player">
    <!--
      CrewOnTheBeats Web Player
      Uso rápido:
      1) Sube este archivo a tu repo GitHub (por ejemplo: /crewonthebeats/index.html) y activa GitHub Pages.
      2) Crea un archivo playlist.json en la misma carpeta con este formato:
         [
           { "title": "Canción 1", "url": "https://raw.githubusercontent.com/tuusuario/tu-repo/main/audio/track1.mp3", "artist": "Artista" },
           { "title": "Canción 2", "url": "https://.../track2.mp3" }
         ]
      3) Opcional: renombra playlist.json o apunta a otra URL desde el campo "Cargar playlist URL".
      4) También puedes arrastrar y soltar archivos locales para reproducirlos en la sesión.
    -->

    <div class="top">
      <div class="cover" id="cover">COB</div>
      <div class="meta">
        <h1 id="nowTitle">CrewOnTheBeats</h1>
        <p id="nowArtist">Reproductor web — sube tu playlist.json</p>

        <div class="controls">
          <button id="btnPrev" title="Anterior">⏮️</button>
          <button id="btnPlay" class="big" title="Play/Pausa">▶️</button>
          <button id="btnNext" title="Siguiente">⏭️</button>

          <label class="upload-btn" title="Cargar playlist JSON">
            Cargar playlist URL
            <input id="playlistUrl" placeholder="URL de playlist.json" />
            <button id="btnLoad">Cargar</button>
          </label>

          <div style="flex:1"></div>

          <button id="btnShuffle" title="Aleatorio">🔀</button>
          <button id="btnRepeat" title="Repetir">🔁</button>
        </div>

        <div class="progress" id="progress"><div></div></div>
        <div class="footer">
          <div id="time">0:00 / 0:00</div>
          <div>
            Vol <input id="vol" type="range" min="0" max="1" step="0.01" value="0.9" />
          </div>
        </div>
      </div>
    </div>

    <div style="display:flex;gap:10px;align-items:center;margin-top:12px">
      <label class="upload-btn">Agregar archivos locales
        <input id="files" type="file" accept="audio/*" multiple />
      </label>
      <button id="btnClear">Limpiar</button>
    </div>

    <div class="playlist" id="playlist"></div>

    <audio id="audio"></audio>
  </div>

  <script>
    // Estado
    const audio = document.getElementById('audio');
    const playlistEl = document.getElementById('playlist');
    const nowTitle = document.getElementById('nowTitle');
    const nowArtist = document.getElementById('nowArtist');
    const progressBar = document.querySelector('#progress > div');
    const timeEl = document.getElementById('time');
    const cover = document.getElementById('cover');

    let tracks = [];
    let index = 0;
    let isShuffle = false;
    let isRepeat = false;

    function renderPlaylist(){
      playlistEl.innerHTML = '';
      tracks.forEach((t,i)=>{
        const div = document.createElement('div');
        div.className = 'track'+(i===index?' active':'');
        div.innerHTML = `<div style="flex:1"><div class=title>${t.title||('Track '+(i+1))}</div><div style='color:var(--muted);font-size:12px'>${t.artist||t.url}</div></div><div>${formatDuration(t.duration)}</div>`;
        div.onclick = ()=>{ index = i; playIndex(); };
        playlistEl.appendChild(div);
      });
    }

    function formatDuration(s){ if(!s && s!==0) return '--:--'; s = Math.floor(s); const m = Math.floor(s/60); const sec = s%60; return m+':'+String(sec).padStart(2,'0'); }

    function playIndex(){
      if(!tracks.length) return;
      const t = tracks[index];
      audio.src = t.url || t.objectURL;
      audio.play();
      nowTitle.textContent = t.title || 'Pista '+(index+1);
      nowArtist.textContent = t.artist || '';
      cover.textContent = (t.title||'COB').slice(0,3).toUpperCase();
      renderPlaylist();
    }

    document.getElementById('btnPlay').onclick = async ()=>{
      if(audio.paused){ await audio.play(); document.getElementById('btnPlay').textContent='⏸️'; }
      else{ audio.pause(); document.getElementById('btnPlay').textContent='▶️'; }
    }
    document.getElementById('btnNext').onclick = ()=>{ if(isShuffle) index = Math.floor(Math.random()*tracks.length); else index = (index+1)%tracks.length; playIndex(); }
    document.getElementById('btnPrev').onclick = ()=>{ if(audio.currentTime>3) audio.currentTime = 0; else index = (index-1+tracks.length)%tracks.length; playIndex(); }
    document.getElementById('btnShuffle').onclick = ()=>{ isShuffle = !isShuffle; document.getElementById('btnShuffle').style.opacity = isShuffle?1:.6 }
    document.getElementById('btnRepeat').onclick = ()=>{ isRepeat = !isRepeat; document.getElementById('btnRepeat').style.opacity = isRepeat?1:.6 }
    document.getElementById('vol').oninput = (e)=>{ audio.volume = e.target.value }

    audio.ontimeupdate = ()=>{
      if(audio.duration) progressBar.style.width = (audio.currentTime/audio.duration*100)+'%';
      timeEl.textContent = formatDuration(audio.currentTime)+' / '+formatDuration(audio.duration);
    }
    audio.onended = ()=>{
      if(isRepeat) { audio.currentTime = 0; audio.play(); return; }
      if(isShuffle) index = Math.floor(Math.random()*tracks.length);
      else index = (index+1)%tracks.length;
      playIndex();
    }

    // Cargar playlist desde URL (JSON)
    async function loadPlaylistFromUrl(url){
      try{
        const res = await fetch(url);
        if(!res.ok) throw new Error('No se pudo cargar');
        const data = await res.json();
        // Espera que sea array de objetos {title, url, artist}
        tracks = data.map(d=>({title:d.title, url:d.url, artist:d.artist}));
        index = 0; renderPlaylist(); playIndex();
      }catch(err){ alert('Error cargando playlist: '+err.message) }
    }

    document.getElementById('btnLoad').onclick = ()=>{
      const url = document.getElementById('playlistUrl').value.trim();
      if(!url) return alert('Pega la URL pública de playlist.json');
      loadPlaylistFromUrl(url);
    }

    // Cargar archivos locales (drag & drop + file input)
    document.getElementById('files').onchange = async (e)=>{
      const files = Array.from(e.target.files);
      for(const f of files){
        const obj = { title: f.name, artist:'Local', objectURL: URL.createObjectURL(f) };
        tracks.push(obj);
      }
      renderPlaylist(); if(!audio.src) playIndex();
    }

    // Drag & drop support
    const playerEl = document.getElementById('player');
    ['dragenter','dragover'].forEach(ev=> playerEl.addEventListener(ev,(e)=>{ e.preventDefault(); playerEl.style.outline='2px dashed rgba(255,255,255,.05)'; }));
    ['dragleave','drop'].forEach(ev=> playerEl.addEventListener(ev,(e)=>{ e.preventDefault(); playerEl.style.outline='none'; }));
    playerEl.addEventListener('drop', (e)=>{
      const dt = e.dataTransfer;
      if(!dt) return;
      const files = Array.from(dt.files).filter(f=>f.type.startsWith('audio'));
      for(const f of files){ tracks.push({ title:f.name, artist:'Local', objectURL: URL.createObjectURL(f) }); }
      renderPlaylist(); if(!audio.src) playIndex();
    });

    // Limpiar playlist
    document.getElementById('btnClear').onclick = ()=>{ tracks=[]; audio.pause(); audio.src=''; renderPlaylist(); nowTitle.textContent='CrewOnTheBeats'; nowArtist.textContent='Reproductor web — sube tu playlist.json' }

    // Intentar cargar playlist.json por defecto (en la misma carpeta)
    (async ()=>{
      const defaultUrl = 'playlist.json';
      try{
        const r = await fetch(defaultUrl);
        if(r.ok){ const data = await r.json(); tracks = data.map(d=>({title:d.title, url:d.url, artist:d.artist})); renderPlaylist(); }
      }catch(err){}
    })();

    // Pequeña función para medir duración (opcional) -- no bloqueará la UI
    async function fetchDurations(){
      for(const t of tracks){ if(t.duration) continue; if(t.objectURL) { try{ const aud = document.createElement('audio'); aud.src = t.objectURL; aud.preload='metadata'; aud.onloadedmetadata = ()=>{ t.duration = aud.duration; renderPlaylist(); } }catch(e){} } }
    }

    // Observador que intenta obtener duraciones cuando haya cambios
    new MutationObserver(()=>{ fetchDurations() }).observe(playlistEl,{childList:true,subtree:true});
  </script>
</body>
</html>
