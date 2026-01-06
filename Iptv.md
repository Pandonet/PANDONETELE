<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>Stream Engine Pro | IPTV & Favorites</title>
    
    <link href="https://vjs.zencdn.net/8.12.0/video-js.css" rel="stylesheet" />
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap" rel="stylesheet" />
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" />

    <style>
        :root {
            --primary: #00f2ff;
            --fav: #ff4757;
            --bg: #050505;
            --panel: rgba(15, 15, 20, 0.8);
            --border: rgba(255, 255, 255, 0.1);
        }

        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        body, html { 
            margin: 0; padding: 0; width: 100%; height: 100%; 
            background: var(--bg); font-family: 'Plus Jakarta Sans', sans-serif; 
            color: white; overflow: hidden;
        }

        /* Reproductor Fullscreen */
        #video-container {
            position: fixed; inset: 0; display: flex; align-items: center; justify-content: center;
            z-index: 1; background: #000;
        }
        .video-js { width: 100% !important; height: 100% !important; }

        /* Sidebar mejorada */
        .sidebar {
            position: fixed; left: 0; top: 0; width: 350px; height: 100%;
            background: var(--panel); backdrop-filter: blur(30px);
            -webkit-backdrop-filter: blur(30px);
            border-right: 1px solid var(--border);
            z-index: 1000; transition: transform 0.5s cubic-bezier(0.2, 1, 0.3, 1);
            display: flex; flex-direction: column;
        }
        .sidebar.closed { transform: translateX(-100%); }

        .header { padding: 30px 20px 15px; }
        .tabs { display: flex; padding: 0 20px; gap: 10px; margin-bottom: 15px; }
        .tab-btn { 
            flex: 1; padding: 8px; border-radius: 8px; background: rgba(255,255,255,0.05);
            border: 1px solid var(--border); color: #888; cursor: pointer; font-size: 0.8rem;
        }
        .tab-btn.active { background: var(--primary); color: #000; font-weight: 700; }

        .search-area { padding: 0 20px 15px; }
        .ui-input {
            width: 100%; background: rgba(255,255,255,0.07);
            border: 1px solid var(--border); padding: 12px;
            border-radius: 12px; color: white; margin-bottom: 8px;
        }

        .channel-list {
            flex: 1; overflow-y: auto; padding: 10px;
            scrollbar-width: none;
        }
        .channel-list::-webkit-scrollbar { display: none; }

        /* Tarjeta de Canal */
        .item {
            display: flex; align-items: center; padding: 12px;
            border-radius: 14px; margin-bottom: 8px; cursor: pointer;
            transition: all 0.2s ease; background: rgba(255,255,255,0.03);
            position: relative;
        }
        .item:hover { background: rgba(255,255,255,0.1); transform: translateX(5px); }
        .item.active { border-left: 4px solid var(--primary); background: rgba(0, 242, 255, 0.1); }

        .item img { width: 45px; height: 45px; object-fit: contain; margin-right: 15px; }
        .item-content { flex: 1; overflow: hidden; }
        .item-name { font-size: 0.9rem; font-weight: 600; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        .item-group { font-size: 0.7rem; color: #777; }

        .fav-icon { 
            padding: 10px; color: #444; transition: 0.3s;
        }
        .fav-icon.is-fav { color: var(--fav); }

        /* Botón Flotante */
        .menu-toggle {
            position: fixed; bottom: 30px; left: 30px; z-index: 2000;
            width: 55px; height: 55px; border-radius: 18px;
            background: var(--primary); color: #000;
            display: flex; align-items: center; justify-content: center;
            font-size: 1.2rem; cursor: pointer; transition: 0.4s;
            box-shadow: 0 8px 25px rgba(0, 242, 255, 0.4);
        }
        .menu-toggle:hover { transform: scale(1.1) rotate(-9deg); }

        /* Loader */
        #loader {
            position: fixed; inset: 0; background: #000; z-index: 9999;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
        }
        .load-bar { width: 150px; height: 2px; background: rgba(255,255,255,0.1); margin-top: 20px; position: relative; overflow: hidden; }
        .load-bar::after { content: ''; position: absolute; left: -100%; width: 100%; height: 100%; background: var(--primary); animation: loading 2s infinite; }
        @keyframes loading { to { left: 100%; } }

        @media (max-width: 768px) { .sidebar { width: 100%; } }
    </style>
</head>
<body>

    <div id="loader">
        <h1 style="letter-spacing: 8px; font-weight: 800; color: var(--primary);">STREAM ENGINE</h1>
        <div class="load-bar"></div>
    </div>

    <div class="menu-toggle" onclick="ui.toggleMenu()">
        <i class="fas fa-bars-staggered"></i>
    </div>

    <div id="video-container">
        <video id="main-player" class="video-js vjs-big-play-centered" controls playsinline></video>
    </div>

    <aside class="sidebar closed" id="sidebar">
        <div class="header">
            <h2 style="margin:0; font-weight:800;">PANDONETTV</h2>
        </div>
        
        <div class="tabs">
            <button class="tab-btn active" onclick="ui.changeTab('all', this)">CANALES</button>
            <button class="tab-btn" onclick="ui.changeTab('favs', this)">FAVORITOS</button>
        </div>

        <div class="search-area">
            <input type="text" id="search" class="ui-input" placeholder="Buscar por nombre..." oninput="ui.filter()">
            <select id="cat-filter" class="ui-input" onchange="ui.filter()"></select>
        </div>

        <div class="channel-list" id="list"></div>
    </aside>

    <script src="https://vjs.zencdn.net/8.12.0/video.min.js"></script>

    <script>
        const M3U_URL = "https://raw.githubusercontent.com/Pandonet/PandoNet-/refs/heads/main/tvenlinea2.m3u";
        
        const core = {
            data: [],
            favorites: JSON.parse(localStorage.getItem('iptv_favs')) || [],
            player: null,

            async start() {
                this.player = videojs('main-player', {
                    fluid: true,
                    playbackRates: [0.5, 1, 1.5, 2],
                    userActions: { hotkeys: true }
                });

                await this.loadM3U();
                ui.init();
            },

            async loadM3U() {
                try {
                    const res = await fetch(M3U_URL);
                    const text = await res.text();
                    this.parse(text);
                } catch (e) { console.error("Error M3U:", e); }
            },

            parse(text) {
                const lines = text.split('\n');
                for (let i = 0; i < lines.length; i++) {
                    if (lines[i].includes('#EXTINF:')) {
                        const url = lines[i+1]?.trim();
                        if (url && url.startsWith('http')) {
                            const name = lines[i].split(',')[1]?.trim() || "Canal TV";
                            const logo = lines[i].match(/tvg-logo="([^"]+)"/)?.[1] || "";
                            const group = lines[i].match(/group-title="([^"]+)"/)?.[1] || "GENERAL";
                            this.data.push({ id: btoa(url).substring(0,16), name, logo, group, url });
                        }
                    }
                }
            },

            toggleFavorite(id) {
                if (this.favorites.includes(id)) {
                    this.favorites = this.favorites.filter(fid => fid !== id);
                } else {
                    this.favorites.push(id);
                }
                localStorage.setItem('iptv_favs', JSON.stringify(this.favorites));
                ui.render();
            }
        };

        const ui = {
            currentTab: 'all',

            init() {
                this.renderCategories();
                this.render();
                document.getElementById('loader').style.display = 'none';
            },

            toggleMenu() {
                document.getElementById('sidebar').classList.toggle('closed');
            },

            changeTab(tab, btn) {
                this.currentTab = tab;
                document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                this.render();
            },

            renderCategories() {
                const cats = [...new Set(core.data.map(c => c.group))];
                const select = document.getElementById('cat-filter');
                select.innerHTML = '<option value="all">TODAS LAS CATEGORÍAS</option>';
                cats.sort().forEach(c => select.innerHTML += `<option value="${c}">${c}</option>`);
            },

            filter() {
                this.render();
            },

            render() {
                const term = document.getElementById('search').value.toLowerCase();
                const cat = document.getElementById('cat-filter').value;
                const container = document.getElementById('list');
                
                let filtered = core.data.filter(ch => {
                    const matchesSearch = ch.name.toLowerCase().includes(term);
                    const matchesCat = cat === 'all' || ch.group === cat;
                    const matchesTab = this.currentTab === 'all' || core.favorites.includes(ch.id);
                    return matchesSearch && matchesCat && matchesTab;
                });

                container.innerHTML = filtered.map(ch => `
                    <div class="item" onclick="ui.play('${ch.url}', this)">
                        <img src="${ch.logo}" onerror="this.src='https://via.placeholder.com/50/111/fff?text=TV'">
                        <div class="item-content">
                            <div class="item-name">${ch.name}</div>
                            <div class="item-group">${ch.group}</div>
                        </div>
                        <div class="fav-icon ${core.favorites.includes(ch.id) ? 'is-fav' : ''}" 
                             onclick="event.stopPropagation(); core.toggleFavorite('${ch.id}')">
                            <i class="fas fa-heart"></i>
                        </div>
                    </div>
                `).join('');
            },

            play(url, el) {
                document.querySelectorAll('.item').forEach(i => i.classList.remove('active'));
                el.classList.add('active');
                core.player.src({ src: url, type: 'application/x-mpegURL' });
                core.player.play().catch(() => {
                    alert("Este canal no está disponible en este momento.");
                });
                if (window.innerWidth < 1024) this.toggleMenu();
            }
        };

        core.start();
    </script>
</body>
</html>
