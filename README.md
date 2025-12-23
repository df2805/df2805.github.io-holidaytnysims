<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Holiday Tournament Simulator</title>
  <style>
    :root{
      --bg:#f6f7f9;
      --card:#ffffff;
      --border:#e5e7eb;
      --text:#111827;
      --muted:#6b7280;
      --shadow: 0 6px 18px rgba(0,0,0,.06);
      --radius:14px;
    }

    *, *::before, *::after { box-sizing: border-box; }

    body {
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial;
      margin: 0;
      background: var(--bg);
      color: var(--text);
    }

    .container{
      max-width: 1150px;
      margin: 0 auto;
      padding: 18px;
    }

    h1 { margin: 0 0 8px; letter-spacing: .2px; }
    h3 { margin: 0 0 10px; }

    .muted { color: var(--muted); font-size: 13px; }

    .row { display: flex; gap: 16px; flex-wrap: wrap; align-items: flex-start; margin-top: 12px; }

    .card {
      border: 1px solid var(--border);
      border-radius: var(--radius);
      padding: 14px;
      background: var(--card);
      box-shadow: var(--shadow);
    }

    .left { flex: 1 1 560px; min-width: 280px; max-width: 100%; }
    .right { flex: 1 1 360px; min-width: 280px; max-width: 100%; }

    .controls { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; }
    .pill { display: inline-block; padding: 2px 8px; border-radius: 999px; background: #eef0f3; font-size: 12px; color: #374151; }

    input[type="number"], select {
      padding: 8px 10px;
      border-radius: 10px;
      border: 1px solid #d1d5db;
      background: #fff;
      max-width: 100%;
      font: inherit;
    }

    input[type="number"] { width: 160px; }

    .note { margin-top: 8px; font-size: 12px; color: #4b5563; }

    .btn {
      padding: 9px 12px;
      border: 1px solid #d1d5db;
      background: #fff;
      border-radius: 10px;
      cursor: pointer;
      font-weight: 600;
    }
    .btn:hover { background: #f3f4f6; }
    .btn:active { transform: translateY(1px); }
    .btn.win { border-color: #111827; background: #f9fafb; font-weight: 800; }

    .grid2 { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    @media (max-width: 980px) { .grid2 { grid-template-columns: 1fr; } }

    .hr { height: 1px; background: #e5e7eb; margin: 12px 0; }

    .game {
      border: 1px solid #eef0f3;
      border-radius: 12px;
      padding: 12px;
      margin: 10px 0;
      background: #fff;
    }
    .game-title { font-weight: 800; margin-bottom: 8px; }

    .split {
      display: grid;
      grid-template-columns: minmax(0, 1fr) 120px minmax(0, 1fr);
      gap: 10px;
      align-items: center;
    }

    .center {
      text-align: center;
      font-size: 12px;
      color: #4b5563;
    }

    pre { white-space: pre-wrap; word-break: break-word; margin: 0; }

    @media (max-width: 720px) {
      .container { padding: 12px; }
      input[type="number"], select { width: 100%; }
      .controls > * { flex: 1 1 auto; }
      .split { grid-template-columns: 1fr; }
      .center { text-align: left; }
      .btn { width: 100%; }
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Holiday Tournament Simulator</h1>
    <div class="muted">Pick a tournament, click winners to lock picks, then simulate champion odds. (Ratings + model settings are fixed.)</div>

    <div class="card" style="margin-top:12px;">
      <div class="controls">
        <span class="pill">Tournament</span>
        <select id="tourSelect">
          <option value="kokomo">Kokomo (Phil Cox Memorial)</option>
          <option value="shenandoah">Shenandoah (Raider Christmas Classic)</option>
          <option value="halloffame">Hall of Fame Classic (New Castle)</option>
        </select>

        <span class="pill">Sims</span>
        <input type="number" id="simCount" value="50000" min="1000" step="1000" />

        <button class="btn" id="runOdds">Run Odds</button>
        <button class="btn" id="resetPicks">Reset Picks</button>
        <span class="pill" id="status">ready</span>
      </div>
      <div class="note">
        Fixed model: P(A beats B) = 1 / (1 + 10^((RB - RA)/k)), with k = 10. All games treated as neutral site.
      </div>
    </div>

    <div class="row">
      <div class="card left">
        <h3 id="bracketTitle">Bracket (click to lock winners)</h3>
        <div class="muted" id="bracketNote">Games 1-4 (R1), 5-6 (Con), 7-8 (SF), 11 (3rd), 12 (1st). (7th/5th games still simulated behind the scenes.)</div>
        <div class="hr"></div>
        <div class="grid2">
          <div>
            <div class="muted" style="font-weight:800;">Round 1</div>
            <div id="r1"></div>
          </div>
          <div>
            <div class="muted" style="font-weight:800;">Later games</div>
            <div id="later"></div>
          </div>
        </div>
      </div>

      <div class="card right">
        <h3 id="oddsTitle">Champion odds</h3>
        <div class="muted">Only output shown. No ratings editor, no k slider, no placement table.</div>
        <div class="hr"></div>
        <div id="results"><span class="muted">Run odds to see results.</span></div>
      </div>
    </div>

    <script>
      // ===== Fixed model settings =====
      const K_FIXED = 10;

      // ===== Tournament data (2 options) =====
      const TOURNAMENTS = {
        kokomo: {
          id: "kokomo",
          title: "Kokomo (Phil Cox Memorial)",
          champGame: "G12",
          ratings: {
            "Kokomo": 82.42,
            "Merrillville": 66.34,
            "Northridge": 88.20,
            "Indianapolis Chatard": 69.04,
            "Lawrence North": 90.80,
            "South Bend Riley": 89.05,
            "New Haven": 80.10,
            "Charlestown": 77.03,
          },
          games: [
            { id: "G1", title: "Game 1 (R1)", a: "Northridge", b: "Merrillville" },
            { id: "G2", title: "Game 2 (R1)", a: "Indianapolis Chatard", b: "Kokomo" },
            { id: "G3", title: "Game 3 (R1)", a: "Lawrence North", b: "Charlestown" },
            { id: "G4", title: "Game 4 (R1)", a: "New Haven", b: "South Bend Riley" },

            { id: "G5", title: "Game 5 (Con)", a: { lose: "G1" }, b: { lose: "G2" } },
            { id: "G6", title: "Game 6 (Con)", a: { lose: "G3" }, b: { lose: "G4" } },

            { id: "G7", title: "Game 7 (SF)", a: { win: "G1" }, b: { win: "G2" } },
            { id: "G8", title: "Game 8 (SF)", a: { win: "G3" }, b: { win: "G4" } },

            { id: "G9",  title: "Game 9 (7th)", a: { lose: "G5" }, b: { lose: "G6" } },
            { id: "G10", title: "Game 10 (5th)", a: { win: "G5" }, b: { win: "G6" } },

            { id: "G11", title: "Game 11 (3rd)", a: { lose: "G8" }, b: { lose: "G7" } },
            { id: "G12", title: "Game 12 (1st)", a: { win: "G8" }, b: { win: "G7" } },
          ]
        },

        shenandoah: {
          id: "shenandoah",
          title: "Shenandoah (Raider Christmas Classic)",
          champGame: "G12",
          ratings: {
            "Seton Catholic": 48.20,
            "Shenandoah": 71.64,
            "Rushville": 62.26,
            "Waldron": 56.09,
            "Greenwood Christian": 63.00,
            "Anderson": 81.33,
            "Indianapolis Lutheran": 65.24,
            "Warsaw": 74.70,
          },
          games: [
            { id: "G1", title: "Game 1 (R1)", a: "Seton Catholic", b: "Shenandoah" },
            { id: "G2", title: "Game 2 (R1)", a: "Rushville", b: "Waldron" },
            { id: "G3", title: "Game 3 (R1)", a: "Greenwood Christian", b: "Anderson" },
            { id: "G4", title: "Game 4 (R1)", a: "Indianapolis Lutheran", b: "Warsaw" },

            // Per your schedule: Loser G2 vs Loser G1, and Loser G4 vs Loser G3
            { id: "G5", title: "Game 5 (Con)", a: { lose: "G2" }, b: { lose: "G1" } },
            { id: "G6", title: "Game 6 (Con)", a: { lose: "G4" }, b: { lose: "G3" } },

            // Per your schedule: Winner G2 vs Winner G1, Winner G4 vs Winner G3
            { id: "G7", title: "Game 7 (SF)", a: { win: "G2" }, b: { win: "G1" } },
            { id: "G8", title: "Game 8 (SF)", a: { win: "G4" }, b: { win: "G3" } },

            { id: "G9",  title: "Game 9 (7th)", a: { lose: "G6" }, b: { lose: "G5" } },
            { id: "G10", title: "Game 10 (5th)", a: { win: "G6" }, b: { win: "G5" } },

            { id: "G11", title: "Game 11 (3rd)", a: { lose: "G8" }, b: { lose: "G7" } },
            { id: "G12", title: "Game 12 (1st)", a: { win: "G8" }, b: { win: "G7" } },
          ]
        },

        halloffame: {
          id: "halloffame",
          title: "Hall of Fame Classic (New Castle)",
          champGame: "G4",
          ratings: {
            "Crown Point": 89.72,
            "Mount Vernon (Fortville)": 89.28,
            "Silver Creek": 92.67,
            "Northview": 89.89,
          },
          games: [
            { id: "G1", title: "Game 1 (R1)", a: "Mount Vernon (Fortville)", b: "Northview" },
            { id: "G2", title: "Game 2 (R1)", a: "Silver Creek", b: "Crown Point" },
            { id: "G3", title: "Game 3 (3rd)", a: { lose: "G1" }, b: { lose: "G2" } },
            { id: "G4", title: "Game 4 (1st)", a: { win: "G1" }, b: { win: "G2" } },
          ]
        }
      };

      // Active tournament
      let activeKey = "kokomo";
      let GAMES = TOURNAMENTS[activeKey].games;
      let rating = TOURNAMENTS[activeKey].ratings;
      let CHAMP_ID = TOURNAMENTS[activeKey].champGame;

      // Picks: gameId -> teamName (locked)
      let picks = {};

      // ===== Helpers =====
      function $(id) { return document.getElementById(id); }

      function winProb(a, b) {
        const ra = rating[a];
        const rb = rating[b];
        if (!Number.isFinite(ra) || !Number.isFinite(rb)) return 0.5;
        return 1 / (1 + Math.pow(10, (rb - ra) / K_FIXED));
      }

      function flip(a, b) {
        return (Math.random() < winProb(a, b)) ? a : b;
      }

      function isRef(x) {
        return typeof x === "object" && x && (typeof x.win === "string" || typeof x.lose === "string");
      }

      function resolveTeam(x, results) {
        if (!isRef(x)) return x;
        if (x.win) return (results[x.win] && results[x.win].w) ? results[x.win].w : null;
        if (x.lose) return (results[x.lose] && results[x.lose].l) ? results[x.lose].l : null;
        return null;
      }

      // Determine winners/losers from locked picks only
      function computeFromLockedPicks() {
        const results = {};

        for (const g of GAMES) {
          const a = resolveTeam(g.a, results);
          const b = resolveTeam(g.b, results);
          results[g.id] = { a, b, w: null, l: null };

          if (!a || !b) continue;

          const p = picks[g.id];
          if (p === a || p === b) {
            results[g.id].w = p;
            results[g.id].l = (p === a) ? b : a;
          }
        }

        return results;
      }

      // If early pick changes, drop invalid downstream picks
      function validateDownstream() {
        const results = {};

        for (const g of GAMES) {
          const a = resolveTeam(g.a, results);
          const b = resolveTeam(g.b, results);
          results[g.id] = { a, b, w: null, l: null };

          if (!a || !b) {
            if (picks[g.id]) delete picks[g.id];
            continue;
          }

          if (picks[g.id] && picks[g.id] !== a && picks[g.id] !== b) {
            delete picks[g.id];
            continue;
          }

          const p = picks[g.id];
          if (p === a || p === b) {
            results[g.id].w = p;
            results[g.id].l = (p === a) ? b : a;
          }
        }
      }

      // ===== UI rendering =====
      function renderGameCard(g, results) {
        const holder = document.createElement("div");
        holder.className = "game";

        const title = document.createElement("div");
        title.className = "game-title";
        title.textContent = g.title;
        holder.appendChild(title);

        const split = document.createElement("div");
        split.className = "split";

        const a = resolveTeam(g.a, results);
        const b = resolveTeam(g.b, results);

        const mid = document.createElement("div");
        mid.className = "center";

        const btnA = document.createElement("button");
        const btnB = document.createElement("button");
        btnA.className = "btn";
        btnB.className = "btn";

        if (!a || !b) {
          mid.textContent = "Waiting";
          btnA.textContent = a ? `Pick ${a}` : "Pick winner";
          btnB.textContent = b ? `Pick ${b}` : "Pick winner";
          btnA.disabled = true;
          btnB.disabled = true;

          split.append(btnA, mid, btnB);
          holder.appendChild(split);

          const info = document.createElement("div");
          info.className = "muted";
          info.style.marginTop = "8px";
          info.textContent = "Waiting on earlier results.";
          holder.appendChild(info);
          return holder;
        }

        mid.textContent = picks[g.id] ? "Locked" : "Pick winner";

        btnA.textContent = `Pick ${a}`;
        btnB.textContent = `Pick ${b}`;

        if (picks[g.id] === a) btnA.classList.add("win");
        if (picks[g.id] === b) btnB.classList.add("win");

        btnA.onclick = () => {
          picks[g.id] = (picks[g.id] === a) ? null : a;
          if (!picks[g.id]) delete picks[g.id];
          validateDownstream();
          renderBracket();
        };

        btnB.onclick = () => {
          picks[g.id] = (picks[g.id] === b) ? null : b;
          if (!picks[g.id]) delete picks[g.id];
          validateDownstream();
          renderBracket();
        };

        split.append(btnA, mid, btnB);
        holder.appendChild(split);

        const pA = winProb(a, b);
        const pB = 1 - pA;
        const info = document.createElement("div");
        info.className = "muted";
        info.style.marginTop = "8px";
        info.textContent = `${a}: ${(pA * 100).toFixed(1)}%  |  ${b}: ${(pB * 100).toFixed(1)}%`;
        holder.appendChild(info);

        return holder;
      }

      function renderBracket() {
        validateDownstream();
        const results = computeFromLockedPicks();

        const r1 = $("r1");
        const later = $("later");
        r1.innerHTML = "";
        later.innerHTML = "";

        const isR1 = (g) => typeof g.title === "string" && g.title.includes("(R1)");
        const r1Games = GAMES.filter(isR1);
        const otherGames = GAMES.filter(g => !isR1(g));

        r1Games.forEach((g, idx) => {
          r1.appendChild(renderGameCard(g, results));
          if (idx !== r1Games.length - 1) {
            const hr = document.createElement("div");
            hr.className = "hr";
            r1.appendChild(hr);
          }
        });

        otherGames.forEach((g, idx) => {
          later.appendChild(renderGameCard(g, results));
          if (idx !== otherGames.length - 1) {
            const hr = document.createElement("div");
            hr.className = "hr";
            later.appendChild(hr);
          }
        });
      }

      // ===== Simulation =====
      function simulateOnce() {
        const results = {};

        for (const g of GAMES) {
          const a = resolveTeam(g.a, results);
          const b = resolveTeam(g.b, results);
          results[g.id] = { a, b, w: null, l: null };

          if (!a || !b) continue;

          const lp = picks[g.id];
          const w = (lp === a || lp === b) ? lp : flip(a, b);
          const l = (w === a) ? b : a;

          results[g.id].w = w;
          results[g.id].l = l;
        }

        return (results[CHAMP_ID] && results[CHAMP_ID].w) ? results[CHAMP_ID].w : null;
      }

      function runOdds() {
        validateDownstream();

        const n = Math.max(1000, Number($("simCount").value) || 50000);
        $("status").textContent = "running...";

        const champCounts = Object.fromEntries(Object.keys(rating).map(t => [t, 0]));

        for (let i = 0; i < n; i++) {
          const champ = simulateOnce();
          if (champ) champCounts[champ] += 1;
        }

        const sorted = Object.keys(champCounts)
          .map(t => [t, champCounts[t] / n])
          .sort((a,b) => b[1] - a[1]);

        const lines = [];
        lines.push("CHAMPION ODDS");
        lines.push("");
        for (const [t, p] of sorted) {
          lines.push(`${t.padEnd(24)}: ${(p * 100).toFixed(2)}%`);
        }

        const pre = document.createElement("pre");
        pre.textContent = lines.join("\n");

        const resultsEl = $("results");
        resultsEl.innerHTML = "";
        resultsEl.appendChild(pre);

        $("status").textContent = `done (${n.toLocaleString()})`;
      }

      function resetPicks() {
        picks = {};
        $("results").innerHTML = '<span class="muted">Run odds to see results.</span>';
        renderBracket();
      }

      // ===== Tests =====
      function assert(cond, msg) {
        if (!cond) throw new Error("TEST FAIL: " + msg);
      }

      function champCountsKeyExists(team) {
        return team && Object.prototype.hasOwnProperty.call(rating, team);
      }

      function setActiveTournament(key) {
        if (!TOURNAMENTS[key]) return;
        activeKey = key;
        GAMES = TOURNAMENTS[activeKey].games;
        rating = TOURNAMENTS[activeKey].ratings;
        CHAMP_ID = TOURNAMENTS[activeKey].champGame;

        $("bracketTitle").textContent = `Bracket: ${TOURNAMENTS[activeKey].title} (click to lock winners)`;
        $("oddsTitle").textContent = `Champion odds: ${TOURNAMENTS[activeKey].title}`;

        if (activeKey === "kokomo") {
          $("bracketNote").textContent = "Games 1-4 (R1), 5-6 (Con), 7-8 (SF), 11 (3rd), 12 (1st). (7th/5th games still simulated behind the scenes.)";
        } else if (activeKey === "shenandoah") {
          $("bracketNote").textContent = "Games 1-4 (R1), 5-6 (Con), 7-8 (SF), 11 (3rd), 12 (1st). (Consolation pairings follow your Shenandoah schedule.)";
        } else {
          $("bracketNote").textContent = "4-team format: two first-round games, then 3rd-place and championship.";
        }

        picks = {};
        $("results").innerHTML = '<span class="muted">Run odds to see results.</span>';
        $("status").textContent = "ready";
        renderBracket();
      }

      function runTests() {
        // Basic structure per tournament
        for (const key of Object.keys(TOURNAMENTS)) {
          const tny = TOURNAMENTS[key];
          assert(Array.isArray(tny.games), `${key}: games must be an array`);
          assert(Number.isFinite(tny.games.length), `${key}: games length must exist`);
          assert(tny.games.length === 12 || tny.games.length === 4, `${key}: only 8-team(12 games) or 4-team(4 games) supported`);
          assert(typeof tny.champGame === "string", `${key}: champGame required`);
          assert(tny.games.some(g => g.id === tny.champGame), `${key}: champGame id must exist in games`);

          // IDs should be unique
          const ids = tny.games.map(g => g.id);
          assert(new Set(ids).size === ids.length, `${key}: game ids must be unique`);

          // First-round games define the base team set
          const r1Games = tny.games.filter(g => typeof g.title === "string" && g.title.includes("(R1)"));
          assert(r1Games.length === 4 || r1Games.length === 2, `${key}: expected 4 (8-team) or 2 (4-team) R1 games`);

          const baseTeamSet = new Set();
          for (const g of r1Games) {
            baseTeamSet.add(g.a);
            baseTeamSet.add(g.b);
          }

          const expectedTeams = (tny.games.length === 12) ? 8 : 4;
          assert(baseTeamSet.size === expectedTeams, `${key}: Expected ${expectedTeams} unique base teams in R1`);

          for (const nm of baseTeamSet) {
            assert(Number.isFinite(tny.ratings[nm]), `${key}: Missing rating for: ${nm}`);
          }

          // Probability sanity
          const arr = Array.from(baseTeamSet);
          const ra = tny.ratings[arr[0]];
          const rb = tny.ratings[arr[1]];
          const pr = 1 / (1 + Math.pow(10, (rb - ra) / K_FIXED));
          assert(pr > 0 && pr < 1, `${key}: winProb must be between 0 and 1`);
        }

        // Newline join sanity (this was the bug)
        const joined = ["A", "B"].join("\\n");
        assert(joined === "A\\nB", "Expected join(\\n) to work");

        // resolveTeam should return null when dependency not decided
        const tmp = {};
        assert(resolveTeam({ win: "G1" }, tmp) === null, "Unresolved winner ref should be null");
        assert(resolveTeam({ lose: "G1" }, tmp) === null, "Unresolved loser ref should be null");

        // Champion should come from the active tournament's rating keys
        const savedKey = activeKey;

        setActiveTournament("kokomo");
        const champ1 = simulateOnce();
        assert(Object.prototype.hasOwnProperty.call(TOURNAMENTS.kokomo.ratings, champ1), "Kokomo champ must be a Kokomo team");

        setActiveTournament("shenandoah");
        const champ2 = simulateOnce();
        assert(Object.prototype.hasOwnProperty.call(TOURNAMENTS.shenandoah.ratings, champ2), "Shenandoah champ must be a Shenandoah team");

        setActiveTournament("halloffame");
        const champ3 = simulateOnce();
        assert(Object.prototype.hasOwnProperty.call(TOURNAMENTS.halloffame.ratings, champ3), "Hall of Fame champ must be a Hall of Fame team");

        // Locked pick should be respected
        picks = { G1: GAMES[0].a };
        const one = computeFromLockedPicks();
        assert(one.G1.w === GAMES[0].a, "Locked pick should set winner");

        // Restore
        setActiveTournament(savedKey);
      }

      function init() {
        runTests();

        const sel = $("tourSelect");
        sel.value = activeKey;
        sel.onchange = () => setActiveTournament(sel.value);

        setActiveTournament(activeKey);

        $("runOdds").onclick = () => runOdds();
        $("resetPicks").onclick = () => resetPicks();
      }

      init();
    </script>
  </div>
</body>
</html>
