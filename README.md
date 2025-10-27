#Phase5-NM
Jwt-token-refresh-and-expiry--1
A JWT (JSON Web Token) is used to securely transmit user data between a client and a server. It contains an expiry time (exp) that defines how long it remains valid. Once expired, the token cannot be used. To avoid forcing users to log in again, a refresh token is issued, which lasts longer and can be used to get a new access token.

<!doctype html>

<title>JWT Refresh Rotation Demo (Single HTML)</title> <style> :root{font-family:Inter,system-ui,-apple-system,"Segoe UI",Roboto,"Helvetica Neue",Arial} body{margin:0;background:#0f172a;color:#e6eef8;display:flex;gap:1rem;min-height:100vh;padding:24px} .card{background:#0b1220;border-radius:12px;padding:16px;box-shadow:0 6px 18px rgba(2,6,23,.6);flex:1;min-width:320px} h1{margin:.25rem 0 1rem;font-size:1.2rem} label{display:block;margin-top:.5rem;font-size:.85rem;color:#9fb1d7} input, button, textarea, select{margin-top:.35rem;padding:.6rem;border-radius:8px;border:1px solid #1f2b46;background:#071229;color:#e6eef8;width:100%;box-sizing:border-box} button{cursor:pointer;border:none;padding:.6rem .8rem} .row{display:flex;gap:.6rem} .muted{color:#7f98bd;font-size:.85rem} pre{background:#071226;border-radius:8px;padding:8px;overflow:auto;height:140px} .small{font-size:.85rem} .log{background:#06121f;border-radius:8px;padding:12px;height:420px;overflow:auto;font-family:monospace;font-size:.85rem} .chips{display:flex;gap:.4rem;flex-wrap:wrap} .chip{background:#0b223a;padding:.25rem .5rem;border-radius:999px;font-size:.8rem;border:1px solid #12314a} .warn{color:#ffb4a2} footer{font-size:.8rem;color:#9fb1d7;margin-top:8px} .grid{display:grid;grid-template-columns:1fr 1fr;gap:1rem} @media(max-width:900px){body{flex-direction:column} .grid{grid-template-columns:1fr}} </style>
JWT Refresh Rotation Demo — Single HTML
Simulated server + client in browser. Use the UI to register, login, call protected API, refresh tokens, and attempt token replay.  <label style="margin-top:12px">Actions</label>
  <div style="display:flex;gap:.5rem">
    <button id="btnProtected">Call Protected API</button>
    <button id="btnRefresh">Refresh Access Token</button>
    <button id="btnSimReplay">Simulate Replay (use old refresh)</button>
  </div>

  <label style="margin-top:12px">Client Storage (tokens stored here)</label>
  <div class="chips" id="clientTokens"></div>

  <label style="margin-top:12px">Server-side Store Snapshot</label>
  <pre id="serverStore"></pre>

  <footer>
    <div class="small">How to test replay:</div>
    <ol style="margin:.25rem 0 .5rem .9rem">
      <li>Login -> get refresh token (current).</li>
      <li>Click Refresh -> new refresh token replaces old (rotation).</li>
      <li>Click "Simulate Replay" to attempt using the old (now invalid) refresh token. Server will detect replay and revoke all user's sessions.</li>
    </ol>
    <div class="warn">Note: This is a simulation. In real systems the server persists refresh tokens and uses secure signing/secrets.</div>
  </footer>
</div>

<div>
  <label>Logs</label>
  <div class="log" id="log"></div>

  <label style="margin-top:12px">Latest Access Token (decoded)</label>
  <pre id="accessDecoded">{}</pre>

  <label style="margin-top:12px">Latest Refresh Token (raw)</label>
  <textarea id="rawRefresh" rows="3" readonly></textarea>
</div><script> /* jwt-refresh-demo (single-file) - This simulates a server that issues access + refresh tokens. - Refresh tokens include a unique jti; server stores hashed token + jti. - On refresh: server verifies token, finds stored jti record, then ROTATES: delete old record -> create new refresh token (new jti) and save it. - If an already-used refresh token is presented (jti not found) it's treated as REPLAY: revoke all refresh tokens for that user. - All persistence is in-memory within this page (serverStore object). */ /* ------------------------- Utility (demo-only "hash") ------------------------- For simplicity and portability we use a tiny non-cryptographic "hash" (FNV-1a) to simulate storing a token hash. THIS IS NOT SECURE. In production, use bcrypt/argon2 and server-side secrets. */ function fnv1a(str) { let h = 2166136261 >>> 0; for (let i = 0; i < str.length; i++) { h ^= str.charCodeAt(i); h += (h << 1) + (h << 4) + (h << 7) + (h << 8) + (h << 24); } return (h >>> 0).toString(16); } function nowMs(){ return Date.now(); } function secsFromNow(s){ return Math.floor((Date.now()/1000) + s); } /* ------------------------- "Server" state & functions ------------------------- */ const server = { users: [], // { id, email, password } (password plain here for demo) refreshTokens: [], // { userId, jti, tokenHash, expiresAt } log(msg){ appendLog('[server] '+msg); updateServerView(); }, }; // tiny uid function uid(prefix='u'){ return prefix + '-' + Math.random().toString(36).slice(2,10); } /* Token generation (demo): tokens are
