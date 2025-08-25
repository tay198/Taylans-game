<!doctype html>

<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Say Hello</title>
    <style>
      :root {
        --bg: #0f172a;      /* slate-900 */
        --card: #111827;    /* gray-900 */
        --text: #e5e7eb;    /* gray-200 */
        --muted: #9ca3af;   /* gray-400 */
        --accent: #22d3ee;  /* cyan-400 */
        --accent-2: #38bdf8;/* sky-400 */
        --error: #f87171;   /* red-400 */
        --ok: #34d399;      /* emerald-400 */
      }
      * { box-sizing: border-box; }
      html, body { height: 100%; }
      body {
        margin: 0;
        font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Helvetica Neue, Arial, "Apple Color Emoji", "Segoe UI Emoji";
        background: radial-gradient(1200px 800px at 10% 10%, rgba(34,211,238,.12), transparent 60%),
                    radial-gradient(900px 700px at 90% 90%, rgba(56,189,248,.12), transparent 60%),
                    var(--bg);
        color: var(--text);
        display: grid;
        place-items: center;
        padding: 24px;
      }
      .card {
        width: 100%;
        max-width: 560px;
        background: linear-gradient(180deg, rgba(255,255,255,0.04), rgba(255,255,255,0.02));
        backdrop-filter: blur(8px);
        border: 1px solid rgba(255,255,255,0.08);
        border-radius: 20px;
        box-shadow: 0 10px 30px rgba(0,0,0,.4);
        padding: 28px;
      }
      h1 { margin: 0 0 8px; font-size: 28px; letter-spacing: .5px; }
      p.hint { margin: 0 0 24px; color: var(--muted); }
      form { display: grid; gap: 12px; grid-template-columns: 1fr auto; align-items: center; }
      label { grid-column: 1 / -1; font-size: 14px; color: var(--muted); }
      input[type="text"] {
        width: 100%;
        padding: 12px 14px;
        font-size: 16px;
        background: #0b1220;
        color: var(--text);
        border: 1px solid rgba(255,255,255,0.12);
        border-radius: 12px;
        outline: none;
      }
      input[type="text"]:focus { border-color: var(--accent); box-shadow: 0 0 0 3px rgba(34,211,238,.18); }
      button {
        padding: 12px 16px;
        font-size: 16px;
        border: 0;
        border-radius: 12px;
        background: linear-gradient(90deg, var(--accent), var(--accent-2));
        color: #041014;
        font-weight: 700;
        cursor: pointer;
      }
      .row { display: contents; }
      .helper { grid-column: 1 / -1; font-size: 12px; color: var(--muted); }
      .msg { margin-top: 18px; padding: 14px 16px; border-radius: 12px; background: rgba(255,255,255,.04); border: 1px solid rgba(255,255,255,.08); }
      .msg.ok { border-color: rgba(52, 211, 153, .45); }
      .msg.error { border-color: rgba(248, 113, 113, .45); color: #fecaca; }
      .tiny { font-size: 12px; color: var(--muted); margin-top: 12px; }
      .hidden { display: none; }
    </style>
  </head>
  <body>
    <main class="card" role="main">
      <h1>Say hello 👋</h1>
      <p class="hint">Type your name and press <strong>Enter</strong> or click <em>Greet</em>.</p><form id="hello-form" autocomplete="on">
    <label for="name">Name</label>
    <div class="row">
      <input id="name" name="name" type="text" placeholder="e.g. taylan" autofocus aria-describedby="helper" required />
      <button type="submit" id="greet-btn">Greet</button>
    </div>
    <div id="helper" class="helper">Try typing <code>taylan</code>.</div>
  </form>

  <div id="greeting" class="msg hidden" aria-live="polite"></div>
  <div class="tiny">Tip: You can also use <code>?name=YourName</code> in the URL.</div>
</main>

<script>
  const form = document.getElementById('hello-form');
  const input = document.getElementById('name');
  const msg = document.getElementById('greeting');

  function escapeHtml(str) {
    return str.replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'':'&#39;'}[c]));
  }

  function showMessage(text, kind = 'ok') {
    msg.classList.remove('hidden', 'ok', 'error');
    msg.classList.add(kind);
    msg.innerHTML = text;
  }

  function greet(name) {
    const clean = name.trim();
    if (!clean) {
      showMessage('Please enter a name.', 'error');
      return;
    }
    const safe = escapeHtml(clean);
    showMessage(`Hello, <strong>${safe}</strong>!`);
    try { localStorage.setItem('lastName', clean); } catch (_) {}
  }

  form.addEventListener('submit', (e) => {
    e.preventDefault();
    greet(input.value);
  });

  // Prefill from URL (?name=...) or last used value.
  (function init() {
    const params = new URLSearchParams(location.search);
    const fromUrl = params.get('name');
    let initial = fromUrl;
    if (!initial) {
      try { initial = localStorage.getItem('lastName') || ''; } catch (_) { initial = ''; }
    }
    if (initial) {
      input.value = initial;
      greet(initial);
    }
  })();
</script>

  </body>
</html>
