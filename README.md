# index html

const API = '';
let token = localStorage.getItem('token');
let pendingUserId = null;

function show(name) {
  document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
  document.getElementById(`view-${name}`).classList.add('active');
  if (name === 'browse') loadMatches();
  if (name === 'connections') loadConnections();
}

function setMsg(id, text, ok = false) {
  const el = document.getElementById(id);
  el.textContent = text;
  el.className = 'msg' + (ok ? ' ok' : '');
}

async function api(path, opts = {}) {
  opts.headers = {
    'Content-Type': 'application/json',
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
    ...(opts.headers || {})
  };
  if (opts.body && !(opts.body instanceof FormData))
    opts.body = JSON.stringify(opts.body);
  if (opts.body instanceof FormData) delete opts.headers['Content-Type'];

  const res = await fetch(API + path, opts);
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(data.error || 'Request failed');
  return data;
}

// ---------- REGISTER ----------
async function register() {
  try {
    const r = await api('/api/register', {
      method: 'POST',
      body: {
        idNumber: document.getElementById('reg-id').value.trim(),
        phone: document.getElementById('reg-phone').value.trim(),
        password: document.getElementById('reg-pass').value
      }
    });
    pendingUserId = r.userId;

    // Auto-login to get token
    const login = await api('/api/login', {
      method: 'POST',
      body: {
        phone: document.getElementById('reg-phone').value.trim(),
        password: document.getElementById('reg-pass').value
      }
    });
    token = login.token;
    localStorage.setItem('token', token);
    show('pay');
  } catch (e) { setMsg('reg-msg', e.message); }
}

// ---------- PAY REGISTRATION ----------
async function payRegistration() {
  try {
    setMsg('pay-msg', 'Sending STK push...');
    const r = await api('/api/pay/registration', { method: 'POST' });
    if (r.paid) {
      setMsg('pay-msg', '✅ Paid! Now upload photos.', true);
      setTimeout(() => show('photos'), 800);
    } else {
      setMsg('pay-msg', '📲 Check your phone and enter M-Pesa PIN. Then reload.');
    }
  } catch (e) { setMsg('pay-msg', e.message); }
}

// ---------- UPLOAD PHOTOS ----------
async function uploadPhotos() {
  try {
    const files = document.getElementById('photo-input').files;
    if (!files.length) return setMsg('photo-msg', 'Select at least one photo');
    const fd = new FormData();
    for (const f of files) fd.append('photos', f);
    await api('/api/photos', { method: 'POST', body: fd });
    setMsg('photo-msg', '✅ Uploaded!', true);
    setTimeout(() => show('browse'), 800);
  } catch (e) { setMsg('photo-msg', e.message); }
}

// ---------- LOGIN ----------
async function login() {
  try {
    const r = await api('/api/login', {
      method: 'POST',
      body: {
        phone: document.getElementById('log-phone').value.trim(),
        password: document.getElementById('log-pass').value
      }
    });
    token = r.token;
    localStorage.setItem('token', token);
    if (r.regPaid) show('browse');
    else show('pay');
  } catch (e) { setMsg('login-msg', e.message); }
}

// ---------- BROWSE ----------
async function loadMatches() {
  const wrap = document.getElementById('matches');
  wrap.innerHTML = 'Loading...';
  try {
    const list = await api('/api/matches');
    if (!list.length) { wrap.innerHTML = '<p style="grid-column:1/-1;color:#888">No matches yet — invite friends!</p>'; return; }
    wrap.innerHTML = list.map(u => `
      <div class="card">
        <img src="${u.photo}" alt="profile" />
        <div class="overlay">
          <button onclick="connect('${u.id}', this)">💘 Connect (KES 50)</button>
        </div>
      </div>`).join('');
  } catch (e) { wrap.innerHTML = `<p style="color:#d6336c">${e.message}</p>`; }
}

// ---------- CONNECT ----------
async function connect(targetId, btn) {
  btn.disabled = true;
  btn.textContent = 'Processing...';
  try {
    const r = await api('/api/connect', { method: 'POST', body: { targetId } });
    if (r.paid) {
      btn.textContent = `✅ ${r.target.phone}`;
      btn.classList.add('connected');
    } else {
      btn.textContent = '📲 Check phone for PIN';
    }
  } catch (e) {
    btn.disabled = false;
    btn.textContent = '💘 Connect (KES 50)';
    alert(e.message);
  }
}

// ---------- CONNECTIONS ----------
async function loadConnections() {
  const wrap = document.getElementById('conn-list');
  wrap.innerHTML = 'Loading...';
  try {
    const list = await api('/api/connections');
    if (!list.length) { wrap.innerHTML = '<p style="grid-column:1/-1;color:#888">No connections yet.</p>'; return; }
    wrap.innerHTML = list.map(u => `
      <div class="card">
        <img src="${u.photo}" alt="match" />
        <div class="overlay">
          <p class="connected">📞 ${u.phone}</p>
        </div>
      </div>`).join('');
  } catch (e) { wrap.innerHTML = `<p style="color:#d6336c">${e.message}</p>`; }
}

function logout() {
  token = null;
  localStorage.removeItem('token');
  show('login');
}

// ---------- BOOT ----------
(async function boot() {
  if (!token) { show('login'); return; }
  try {
    const list = await api('/api/matches');
    show('browse');
  } catch {
    show('login');
  }
})();