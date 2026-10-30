
<section id="fortnite-cup">
  <style>
    #fortnite-cup {
      font-family: Arial, sans-serif;
      color: #fff;
      background: #101526;
      padding: 24px;
      border-radius: 16px;
      max-width: 1100px;
      margin: 20px auto;
    }
    #fortnite-cup * { box-sizing: border-box; }
    #fortnite-cup h2 { color: #64dcff; }
    #fortnite-cup .cup-card {
      background: #1d2540;
      border-radius: 12px;
      padding: 18px;
      margin: 16px 0;
    }
    #fortnite-cup input,
    #fortnite-cup select {
      background: #0d1428;
      color: white;
      border: 1px solid #566080;
      border-radius: 7px;
      padding: 10px;
      margin: 5px;
      max-width: 100%;
    }
    #fortnite-cup button {
      background: #347aff;
      color: white;
      border: none;
      border-radius: 7px;
      padding: 11px 15px;
      margin: 5px;
      cursor: pointer;
      font-weight: bold;
    }
    #fortnite-cup button:hover { background: #195ce0; }
    #fortnite-cup .cup-table { overflow-x: auto; }
    #fortnite-cup table {
      width: 100%;
      border-collapse: collapse;
    }
    #fortnite-cup th,
    #fortnite-cup td {
      padding: 12px;
      text-align: left;
      border-bottom: 1px solid #394260;
    }
    #fortnite-cup th { color: #64dcff; }
    #fortnite-cup .cup-status { color: #8ff0ad; }
    #fortnite-cup .cup-note { color: #b5c1db; font-size: 13px; }
  </style>

  <h1>🏆 FORTNITE DUO CUP</h1>
  <p>6 Games • Classement automatique • Points FNCS</p>

  <div class="cup-card">
    <h2>➕ Ajouter un duo</h2>
    <input id="cup-p1" placeholder="Pseudo joueur 1">
    <input id="cup-p2" placeholder="Pseudo joueur 2">
    <button id="cup-add">Ajouter le duo</button>
  </div>

  <div class="cup-card">
    <h2>🎮 Entrer les résultats</h2>
    <select id="cup-game"></select>
    <select id="cup-duo"></select>
    <input id="cup-top" type="number" min="1" max="50"
           placeholder="Placement (1 à 50)">
    <input id="cup-kills" type="number" min="0"
           placeholder="Kills du duo">
    <button id="cup-save">Enregistrer</button>
    <p id="cup-message" class="cup-status" aria-live="polite"></p>
  </div>

  <div class="cup-card">
    <h2>🏆 Classement général</h2>
    <div class="cup-table">
      <table>
        <thead>
          <tr>
            <th>#</th><th>Duo</th><th>Games</th>
            <th>Top 1</th><th>Kills</th><th>Points</th>
          </tr>
        </thead>
        <tbody id="cup-ranking"></tbody>
      </table>
    </div>
  </div>

  <div class="cup-card">
    <h2>📋 Résultats par game</h2>
    <select id="cup-view-game"></select>
    <div class="cup-table">
      <table>
        <thead>
          <tr>
            <th>Duo</th><th>Top</th><th>Kills</th>
            <th>Points</th><th>Action</th>
          </tr>
        </thead>
        <tbody id="cup-results"></tbody>
      </table>
    </div>
  </div>

  <p class="cup-note">
    Les résultats sont enregistrés dans ce navigateur.
    Cette version ne synchronise pas encore les visiteurs.
  </p>

  <script>
  (() => {
    const root = document.getElementById("fortnite-cup");
    if (root.dataset.initialized) return;
    root.dataset.initialized = "true";

    const $ = id => root.querySelector("#" + id);

    // Barème FNCS configurable
    const PLACEMENT = [
      65,56,52,48,44,40,38,36,34,32,30,28,26,
      24,22,20,18,16,14,12,10,8,6,4,2
    ];
    const KILL_POINTS = 4;
    const GAMES = 6;

    const KEY = "fortnite-duo-cup-v1";
    let state;
    try {
      state = JSON.parse(localStorage.getItem(KEY)) || {};
    } catch {
      state = {};
    }

    let duos = Array.isArray(state.duos) ? state.duos : [];
    let results = state.results || {};

    function save() {
      localStorage.setItem(KEY, JSON.stringify({duos, results}));
    }

    function score(top, kills) {
      return (PLACEMENT[top - 1] || 0) + kills * KILL_POINTS;
    }

    function option(select, value, label) {
      const o = document.createElement("option");
      o.value = value;
      o.textContent = label;
      select.appendChild(o);
    }

    for (let g = 1; g <= GAMES; g++) {
      option($("cup-game"), g, "Game " + g);
      option($("cup-view-game"), g, "Game " + g);
    }

    $("cup-add").addEventListener("click", () => {
      const p1 = $("cup-p1").value.trim();
      const p2 = $("cup-p2").value.trim();
      if (!p1 || !p2) {
        alert("Entre les deux pseudos.");
        return;
      }
      duos.push({id: crypto.randomUUID(), p1, p2});
      $("cup-p1").value = "";
      $("cup-p2").value = "";
      save();
      render();
    });

    $("cup-save").addEventListener("click", () => {
      const game = Number($("cup-game").value);
      const id = $("cup-duo").value;
      const topText = $("cup-top").value;
      const killsText = $("cup-kills").value;
      const top = Number(topText);
      const kills = Number(killsText);

      if (!id || topText === "" || killsText === "" ||
          !Number.isInteger(top) || top < 1 || top > 50 ||
          !Number.isInteger(kills) || kills < 0) {
        alert("Vérifie le duo, le placement et les kills.");
        return;
      }

      results[game + "_" + id] = {top, kills};
      save();
      $("cup-message").textContent = "✅ Résultat enregistré !";
      $("cup-view-game").value = String(game);
      render();
    });

    $("cup-view-game").addEventListener("change", render);

    function getStats(d) {
      let points = 0, kills = 0, wins = 0, played = 0;

      for (let g = 1; g <= GAMES; g++) {
        const r = results[g + "_" + d.id];
        if (!r) continue;
        played++;
        kills += r.kills;
        points += score(r.top, r.kills);
        if (r.top === 1) wins++;
      }

      return {...d, points, kills, wins, played};
    }

    function cell(tr, value) {
      const td = document.createElement("td");
      td.textContent = value;
      tr.appendChild(td);
      return td;
    }

    function render() {
      const duoSelect = $("cup-duo");
      const previous = duoSelect.value;
      duoSelect.replaceChildren();

      for (const d of duos) {
        option(duoSelect, d.id, d.p1 + " / " + d.p2);
      }
      if (duos.some(d => d.id === previous)) {
        duoSelect.value = previous;
      }

      const ranking = duos.map(getStats).sort((a, b) =>
        b.points - a.points ||
        b.wins - a.wins ||
        b.kills - a.kills ||
        a.p1.localeCompare(b.p1)
      );

      const rankBody = $("cup-ranking");
      rankBody.replaceChildren();

      ranking.forEach((d, i) => {
        const tr = document.createElement("tr");
        cell(tr, i === 0 ? "🥇 1" :
                 i === 1 ? "🥈 2" :
                 i === 2 ? "🥉 3" : i + 1);
        cell(tr, d.p1 + " / " + d.p2);
        cell(tr, d.played + "/6");
        cell(tr, d.wins);
        cell(tr, d.kills);
        cell(tr, d.points);
        rankBody.appendChild(tr);
      });

      const game = Number($("cup-view-game").value);
      const gameRows = duos
        .map(d => ({d, r: results[game + "_" + d.id]}))
        .filter(x => x.r)
        .sort((a, b) =>
          score(b.r.top, b.r.kills) -
          score(a.r.top, a.r.kills)
        );

      const resultBody = $("cup-results");
      resultBody.replaceChildren();

      for (const {d, r} of gameRows) {
        const tr = document.createElement("tr");
        cell(tr, d.p1 + " / " + d.p2);
        cell(tr, r.top);
        cell(tr, r.kills);
        cell(tr, score(r.top, r.kills));

        const action = cell(tr, "");
        const btn = document.createElement("button");
        btn.textContent = "Modifier";
        btn.addEventListener("click", () => {
          $("cup-game").value = String(game);
          $("cup-duo").value = d.id;
          $("cup-top").value = r.top;
          $("cup-kills").value = r.kills;
          root.scrollIntoView({behavior:"smooth"});
        });
        action.appendChild(btn);
        resultBody.appendChild(tr);
      }
    }

    render();
  })();
  </script>
</section>
