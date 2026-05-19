# acervo-figurinos
Acervo de Visualidades da Cena, do curso da Licenciatura em Teatro, IFF Campus Campos Centro
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Acervo de Figurinos</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,600;0,700;0,900;1,400;1,600&family=Cormorant+Garamond:ital,wght@0,400;0,500;0,600;1,400;1,500&family=DM+Mono:wght@400;500&family=Inter:wght@300;400;500;600&display=swap" rel="stylesheet">

  <!-- Firebase SDKs -->
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
    import {
      getFirestore, collection, addDoc, updateDoc, deleteDoc, doc,
      onSnapshot, serverTimestamp, enableIndexedDbPersistence
    } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";
    import {
      getStorage, ref, uploadBytes, getDownloadURL, deleteObject
    } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

    const firebaseConfig = {
      apiKey: "AIzaSyBu-6L9crKFXtOBRfOWs_96f3A5OsJ7gNY",
      authDomain: "acervo-figurinos.firebaseapp.com",
      databaseURL: "https://acervo-figurinos-default-rtdb.firebaseio.com",
      projectId: "acervo-figurinos",
      storageBucket: "acervo-figurinos.firebasestorage.app",
      messagingSenderId: "976611294453",
      appId: "1:976611294453:web:1fbd7c130c1d2703d219bc"
    };

    let db, storage, isConfigured = false;

    try {
      const app = initializeApp(firebaseConfig);
      db = getFirestore(app);
      storage = getStorage(app);
      isConfigured = true;

      // Persistência offline — dados ficam no navegador mesmo sem internet
      enableIndexedDbPersistence(db).catch(e => {
        if (e.code === 'failed-precondition') console.warn('Persistência: múltiplas abas abertas');
        else if (e.code === 'unimplemented') console.warn('Persistência não suportada neste browser');
      });

      console.log("✅ Firebase conectado!");
    } catch(e) {
      console.error("❌ Erro ao conectar Firebase:", e);
    }

    // ── Ouvir coleção sem orderBy (evita erro de índice) ─────────
    // Ordenação feita no JS para não depender de índice composto
    function ouvirColecao(nomeCol, callback) {
      if (!isConfigured) { window.ouvirLocal(nomeCol, callback); return () => {}; }
      return onSnapshot(
        collection(db, nomeCol),
        { includeMetadataChanges: false },
        snap => {
          const docs = snap.docs.map(d => ({ id: d.id, ...d.data() }));
          // Ordena por criadoEm desc no cliente
          docs.sort((a, b) => {
            const ta = a.criadoEm?.toMillis ? a.criadoEm.toMillis() : new Date(a.criadoEm||0).getTime();
            const tb = b.criadoEm?.toMillis ? b.criadoEm.toMillis() : new Date(b.criadoEm||0).getTime();
            return tb - ta;
          });
          callback(docs);
        },
        err => {
          console.error(`Erro ao ouvir ${nomeCol}:`, err.code, err.message);
          // Fallback para local se der erro de permissão
          window.ouvirLocal(nomeCol, callback);
        }
      );
    }

    // ── Upload de imagem ─────────────────────────────────────────
    async function uploadImagem(file) {
      if (!isConfigured || !storage) throw new Error("Firebase Storage não disponível");
      const nome = `figurinos/${Date.now()}_${file.name.replace(/[^a-zA-Z0-9.]/g,'_')}`;
      const snap = await uploadBytes(ref(storage, nome), file);
      return await getDownloadURL(snap.ref);
    }

    // ── CRUD genérico ────────────────────────────────────────────
    async function salvar(col, dados) {
      if (!isConfigured) return window.usarModoLocal(col, dados);
      const ts = new Date().toISOString(); // timestamp ISO como fallback
      return await addDoc(collection(db, col), { ...dados, criadoEm: serverTimestamp(), criadoEmISO: ts });
    }
    async function atualizar(col, id, dados) {
      if (!isConfigured) return window.atualizarLocal(col, id, dados);
      await updateDoc(doc(db, col, id), { ...dados, atualizadoEm: serverTimestamp() });
    }
    async function deletar(col, id) {
      if (!isConfigured) return window.deletarLocal(col, id);
      await deleteDoc(doc(db, col, id));
    }

    // ── Expor API global ─────────────────────────────────────────
    window.Firebase = {
      isConfigured: () => isConfigured,
      // Figurinos
      salvarFigurino:    (d) => salvar('figurinos', d),
      atualizarFigurino: (id, d) => atualizar('figurinos', id, d),
      deletarFigurino:   async (id, fotoPath) => {
        await deletar('figurinos', id);
        if (fotoPath && storage) try { await deleteObject(ref(storage, fotoPath)); } catch(_){}
      },
      ouvirFigurinos:    (cb) => ouvirColecao('figurinos', cb),
      // Alunos
      salvarAluno:       (d) => salvar('alunos', d),
      atualizarAluno:    (id, d) => atualizar('alunos', id, d),
      deletarAluno:      (id) => deletar('alunos', id),
      ouvirAlunos:       (cb) => ouvirColecao('alunos', cb),
      // Empréstimos
      salvarEmprestimo:  (d) => salvar('emprestimos', d),
      atualizarEmprestimo: (id, d) => atualizar('emprestimos', id, d),
      ouvirEmprestimos:  (cb) => ouvirColecao('emprestimos', cb),
      // Storage
      uploadImagem,
    };

    window.dispatchEvent(new Event('firebase-ready'));
  </script>

  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --background:  #0f0f13;
      --surface-1:   #16161d;
      --surface-2:   #1e1e28;
      --surface-3:   #26263300;
      --border:      rgba(255,255,255,.08);
      --border-gold: rgba(184,134,11,.35);
      --text:        #e8e6e0;
      --text2:       #8a8578;
      --text3:       #5c574f;
      --gold:        #b8860b;
      --gold-light:  #d4a017;
      --gold-dim:    rgba(184,134,11,.15);
      --red:         #c0392b;
      --red-dim:     rgba(192,57,43,.15);
      --green:       #27ae60;
      --green-dim:   rgba(39,174,96,.15);
      --blue:        #2980b9;
      --blue-dim:    rgba(41,128,185,.15);
      --radius:      8px;
      --radius-lg:   12px;
      --font-head:   'Cormorant Garamond', serif;
      --font-body:   'Inter', sans-serif;
      --font-mono:   'DM Mono', monospace;
      --shadow:      0 1px 12px rgba(0,0,0,.4);
      --shadow-lg:   0 8px 40px rgba(0,0,0,.6);
      --sidebar-w:   242px;
    }

    html, body { height: 100%; }
    body {
      font-family: var(--font-body);
      background: var(--background);
      color: var(--text);
      font-size: 14px;
      line-height: 1.5;
    }

    #app { display: flex; height: 100vh; overflow: hidden; }

    /* ════════════════════ BANNER ════════════════════ */
    #config-banner {
      position: fixed; top: 0; left: 0; right: 0; z-index: 9999;
      background: var(--gold); color: #0f0f13;
      font-size: 13px; font-weight: 500;
      padding: 9px 20px; text-align: center; display: none;
    }
    #config-banner a { color: #0f0f13; font-weight: 700; text-decoration: underline; }
    #config-banner.show { display: block; }

    /* ════════════════════ SIDEBAR ════════════════════ */
    #sidebar {
      width: var(--sidebar-w);
      min-width: var(--sidebar-w);
      background: var(--surface-1);
      border-right: 1px solid var(--border);
      display: flex;
      flex-direction: column;
      height: 100vh;
      overflow: hidden;
      flex-shrink: 0;
      transition: width .25s ease, min-width .25s ease;
    }
    #sidebar.collapsed {
      width: 56px;
      min-width: 56px;
    }

    /* Header */
    #sidebar-header {
      padding: 20px 16px 16px;
      border-bottom: 1px solid var(--border);
      display: flex; align-items: center; gap: 10px;
      overflow: hidden;
    }
    .logo-icon-wrap {
      width: 32px; height: 32px; border-radius: 8px;
      background: var(--gold); display: flex; align-items: center;
      justify-content: center; flex-shrink: 0;
    }
    .logo-icon-wrap svg { width: 16px; height: 16px; color: #0f0f13; }
    .logo-texts { overflow: hidden; white-space: nowrap; }
    .logo-eyebrow {
      font-size: 9px; font-weight: 700; letter-spacing: .14em;
      text-transform: uppercase; color: var(--text3); line-height: 1;
      margin-bottom: 2px;
    }
    .logo-name {
      font-family: var(--font-head); font-size: 17px; font-weight: 700;
      color: var(--gold); line-height: 1;
    }
    #sidebar.collapsed .logo-texts { display: none; }

    /* Nav */
    #sidebar nav {
      flex: 1; padding: 10px 8px;
      overflow-y: auto; overflow-x: hidden;
    }
    .nav-section-label {
      font-size: 9px; font-weight: 700; letter-spacing: .14em;
      text-transform: uppercase; color: var(--text3);
      padding: 10px 10px 4px;
      white-space: nowrap; overflow: hidden;
    }
    #sidebar.collapsed .nav-section-label { opacity: 0; height: 0; padding: 0; }

    .nav-item {
      display: flex; align-items: center; gap: 10px;
      padding: 9px 10px; border-radius: var(--radius);
      color: var(--text2); cursor: pointer;
      transition: all .15s; margin-bottom: 1px;
      white-space: nowrap; overflow: hidden;
      position: relative;
    }
    .nav-item:hover { background: var(--surface-2); color: var(--text); }
    .nav-item.active { background: var(--gold-dim); color: var(--gold-light); }
    .nav-item.active .nav-icon-svg { opacity: 1; }

    .nav-icon-svg {
      width: 16px; height: 16px; flex-shrink: 0; opacity: .7;
    }
    .nav-item.active .nav-icon-svg { opacity: 1; }

    .nav-label {
      font-size: 13.5px; font-weight: 500;
      overflow: hidden; white-space: nowrap;
      transition: opacity .2s;
    }
    #sidebar.collapsed .nav-label { opacity: 0; width: 0; overflow: hidden; }

    .nav-badge {
      margin-left: auto; font-size: 10px; font-weight: 700;
      background: var(--red); color: #fff;
      padding: 1px 5px; border-radius: 6px; flex-shrink: 0;
    }
    #sidebar.collapsed .nav-badge { display: none; }

    /* Footer */
    #sidebar-footer {
      padding: 10px 8px 12px;
      border-top: 1px solid var(--border);
      overflow: hidden;
    }
    #sidebar-toggle {
      display: flex; align-items: center; gap: 10px;
      padding: 9px 10px; border-radius: var(--radius);
      color: var(--text3); cursor: pointer;
      transition: all .15s; white-space: nowrap;
    }
    #sidebar-toggle:hover { background: var(--surface-2); color: var(--text2); }
    #sidebar-toggle svg { width: 16px; height: 16px; flex-shrink: 0; }
    .toggle-label { font-size: 13px; font-weight: 500; }
    #sidebar.collapsed .toggle-label { display: none; }

    /* ════════════════════ MAIN ════════════════════ */
    #main { flex: 1; display: flex; flex-direction: column; overflow: hidden; min-width: 0; }

    #topbar {
      height: 56px; padding: 0 24px;
      border-bottom: 1px solid var(--border);
      display: flex; align-items: center; gap: 14px;
      background: var(--surface-1);
      flex-shrink: 0;
    }
    #menu-btn { display: none; }
    #page-title {
      font-family: var(--font-head); font-size: 20px; font-weight: 600;
      color: var(--text); flex: 1; line-height: 1;
    }
    #page-subtitle {
      font-family: var(--font-body); font-size: 12px; color: var(--text3);
      font-style: normal; font-weight: 400;
    }
    #topbar-actions { display: flex; gap: 8px; align-items: center; }

    #content {
      flex: 1; overflow-y: auto; padding: 24px;
      background: var(--background);
    }

    /* ════════════════════ BUTTONS ════════════════════ */
    .btn {
      display: inline-flex; align-items: center; gap: 7px;
      padding: 8px 16px; border-radius: var(--radius); border: none;
      font-family: var(--font-body); font-size: 13px; font-weight: 600;
      cursor: pointer; transition: all .15s; white-space: nowrap;
    }
    .btn svg { width: 14px; height: 14px; flex-shrink: 0; }
    .btn-primary { background: var(--gold); color: #0f0f13; }
    .btn-primary:hover { background: var(--gold-light); }
    .btn-secondary { background: var(--surface-2); color: var(--text); border: 1px solid var(--border); }
    .btn-secondary:hover { border-color: rgba(255,255,255,.2); }
    .btn-ghost { background: transparent; color: var(--text2); border: 1px solid var(--border); }
    .btn-ghost:hover { color: var(--text); background: var(--surface-2); }
    .btn-danger { background: var(--red-dim); color: var(--red); border: 1px solid rgba(192,57,43,.3); }
    .btn-danger:hover { background: rgba(192,57,43,.25); }
    .btn-sm { padding: 5px 12px; font-size: 12px; }
    .btn-sm svg { width: 12px; height: 12px; }
    .btn-lg { padding: 11px 24px; font-size: 14px; }
    .btn:disabled { opacity: .4; cursor: not-allowed; }
    .btn-icon {
      width: 32px; height: 32px; padding: 0;
      display: inline-flex; align-items: center; justify-content: center;
      border-radius: var(--radius); border: 1px solid var(--border);
      background: transparent; color: var(--text2); cursor: pointer;
      transition: all .15s;
    }
    .btn-icon:hover { background: var(--surface-2); color: var(--text); }
    .btn-icon svg { width: 14px; height: 14px; }

    /* ════════════════════ CARDS ════════════════════ */
    .card {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: var(--radius-lg); padding: 20px;
    }
    .card-grid { display: grid; gap: 14px; }

    /* Stat cards */
    .stat-card {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: var(--radius-lg); padding: 18px 20px;
      cursor: pointer; transition: border-color .15s, transform .15s;
    }
    .stat-card:hover { border-color: var(--border-gold); transform: translateY(-1px); }
    .stat-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
    .stat-icon-box {
      width: 36px; height: 36px; border-radius: var(--radius);
      display: flex; align-items: center; justify-content: center;
    }
    .stat-icon-box svg { width: 18px; height: 18px; }
    .stat-value {
      font-family: var(--font-head); font-size: 30px; font-weight: 700;
      color: var(--text); line-height: 1;
    }
    .stat-label {
      font-size: 12px; color: var(--text3); margin-top: 4px;
      text-transform: uppercase; letter-spacing: .07em; font-weight: 500;
    }

    /* ════════════════════ FORMS ════════════════════ */
    .form-group { margin-bottom: 16px; }
    .form-label {
      display: block; font-size: 12px; font-weight: 600; color: var(--text2);
      margin-bottom: 6px; text-transform: uppercase; letter-spacing: .08em;
    }
    .form-input, .form-select, .form-textarea {
      width: 100%; padding: 9px 12px;
      background: var(--surface-2); border: 1px solid var(--border);
      border-radius: var(--radius); color: var(--text);
      font-family: var(--font-body); font-size: 14px;
      transition: border-color .15s; outline: none;
    }
    .form-input::placeholder { color: var(--text3); }
    .form-input:focus, .form-select:focus, .form-textarea:focus {
      border-color: var(--gold);
    }
    .form-select option { background: var(--surface-2); color: var(--text); }
    .form-textarea { resize: vertical; min-height: 80px; }
    .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
    .form-error { color: var(--red); font-size: 12px; margin-top: 4px; }
    .form-hint { color: var(--text3); font-size: 12px; margin-top: 4px; }

    /* ════════════════════ TABLE ════════════════════ */
    .table-wrap { overflow-x: auto; }
    table { width: 100%; border-collapse: collapse; }
    th {
      padding: 10px 16px;
      font-size: 11px; font-weight: 700; letter-spacing: .09em;
      text-transform: uppercase; color: var(--text3);
      background: var(--surface-2); border-bottom: 1px solid var(--border);
      text-align: left; white-space: nowrap;
    }
    td { padding: 13px 16px; border-bottom: 1px solid var(--border); font-size: 13.5px; }
    tbody tr:last-child td { border-bottom: none; }
    tbody tr:hover td { background: rgba(255,255,255,.02); }
    tr.row-warn td { background: rgba(192,57,43,.04) !important; }
    tr.row-warn td:first-child { box-shadow: inset 3px 0 0 var(--red); }

    /* ════════════════════ BADGES ════════════════════ */
    .badge {
      display: inline-flex; align-items: center;
      padding: 2px 8px; border-radius: 4px;
      font-size: 11px; font-weight: 600; letter-spacing: .04em;
      white-space: nowrap;
    }
    .badge-green  { background: var(--green-dim); color: var(--green); }
    .badge-red    { background: var(--red-dim);   color: var(--red); }
    .badge-blue   { background: var(--blue-dim);  color: var(--blue); }
    .badge-yellow { background: var(--gold-dim);  color: var(--gold-light); }
    .badge-gray   { background: rgba(255,255,255,.06); color: var(--text2); }
    .badge-purple { background: rgba(142,68,173,.15); color: #9b59b6; }

    /* ════════════════════ FIGURINO CARDS ════════════════════ */
    .figurino-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(195px, 1fr));
      gap: 16px;
    }
    .figurino-card {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: var(--radius-lg); overflow: hidden;
      transition: border-color .15s, transform .15s;
    }
    .figurino-card:hover { border-color: var(--border-gold); transform: translateY(-2px); }
    .figurino-img {
      width: 100%; height: 158px; background: var(--surface-2);
      display: flex; align-items: center; justify-content: center;
      position: relative; overflow: hidden;
    }
    .figurino-img img { width: 100%; height: 100%; object-fit: cover; }
    .figurino-img-placeholder { opacity: .25; }
    .figurino-img-placeholder svg { width: 48px; height: 48px; }
    .figurino-num {
      position: absolute; top: 8px; left: 8px;
      font-family: var(--font-mono); font-size: 10px;
      background: rgba(0,0,0,.7); color: var(--gold-light);
      padding: 2px 6px; border-radius: 4px;
    }
    .figurino-info { padding: 12px 14px 14px; }
    .figurino-nome {
      font-family: var(--font-head); font-size: 15px; font-weight: 700;
      color: var(--text); margin-bottom: 4px; cursor: pointer; line-height: 1.2;
    }
    .figurino-nome:hover { color: var(--gold-light); }
    .figurino-meta { font-size: 11px; color: var(--text3); margin-bottom: 10px; }
    .figurino-footer {
      display: flex; align-items: center; justify-content: space-between;
    }

    /* ════════════════════ UPLOAD ════════════════════ */
    .upload-area {
      border: 1px dashed rgba(255,255,255,.2); border-radius: var(--radius-lg);
      padding: 32px; text-align: center; cursor: pointer; transition: all .2s;
      background: var(--surface-2);
    }
    .upload-area:hover, .upload-area.drag {
      border-color: var(--gold); background: var(--gold-dim);
    }
    .upload-preview { width: 100%; max-height: 200px; object-fit: contain; border-radius: var(--radius); margin-top: 12px; }

    /* ════════════════════ SEARCH ════════════════════ */
    .search-wrap { position: relative; }
    .search-wrap input { padding-left: 36px; }
    .search-icon {
      position: absolute; left: 11px; top: 50%; transform: translateY(-50%);
      color: var(--text3);
    }
    .search-icon svg { width: 14px; height: 14px; }

    /* ════════════════════ MODALS ════════════════════ */
    .modal-bg {
      position: fixed; inset: 0; background: rgba(0,0,0,.75);
      display: flex; align-items: center; justify-content: center;
      z-index: 1000; padding: 20px; backdrop-filter: blur(4px);
    }
    .modal {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: var(--radius-lg); max-width: 560px; width: 100%;
      max-height: 90vh; overflow-y: auto; box-shadow: var(--shadow-lg);
    }
    .modal-header {
      padding: 18px 22px; border-bottom: 1px solid var(--border);
      display: flex; align-items: center; justify-content: space-between;
    }
    .modal-title {
      font-family: var(--font-head); font-size: 18px; font-weight: 700; color: var(--text);
    }
    .modal-body { padding: 22px; }
    .modal-footer {
      padding: 16px 22px; border-top: 1px solid var(--border);
      display: flex; gap: 8px; justify-content: flex-end;
    }
    .modal-close {
      background: none; border: none; cursor: pointer;
      color: var(--text3); font-size: 18px; line-height: 1;
      transition: color .15s; padding: 2px;
    }
    .modal-close:hover { color: var(--text); }

    /* ════════════════════ TOAST ════════════════════ */
    #toast-container {
      position: fixed; bottom: 20px; right: 20px;
      display: flex; flex-direction: column; gap: 8px; z-index: 9998;
    }
    .toast {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: var(--radius); padding: 11px 16px;
      font-size: 13px; display: flex; align-items: center; gap: 10px;
      min-width: 270px; animation: slideIn .25s ease;
      box-shadow: var(--shadow-lg); color: var(--text);
    }
    .toast.success { border-left: 3px solid var(--green); }
    .toast.error   { border-left: 3px solid var(--red); }
    .toast.info    { border-left: 3px solid var(--blue); }
    @keyframes slideIn { from { transform: translateX(110%); opacity: 0; } to { transform: none; opacity: 1; } }

    /* ════════════════════ EMPTY ════════════════════ */
    .empty { text-align: center; padding: 60px 20px; }
    .empty-icon { margin-bottom: 16px; color: var(--text3); display: flex; justify-content: center; }
    .empty-icon svg { width: 48px; height: 48px; opacity: .3; }
    .empty-title { font-family: var(--font-head); font-size: 18px; color: var(--text2); margin-bottom: 6px; }
    .empty-text { font-size: 13px; color: var(--text3); margin-bottom: 20px; }

    /* ════════════════════ CHIPS ════════════════════ */
    .chips { display: flex; flex-wrap: wrap; gap: 6px; }
    .chip {
      padding: 4px 12px; border-radius: 4px; font-size: 12px; font-weight: 600;
      border: 1px solid var(--border); cursor: pointer; transition: all .15s;
      color: var(--text2); background: var(--surface-2);
      text-transform: uppercase; letter-spacing: .06em;
    }
    .chip.selected { background: var(--gold); color: #0f0f13; border-color: var(--gold); }
    .chip:hover:not(.selected) { border-color: var(--gold); color: var(--gold-light); }

    /* ════════════════════ TABS ════════════════════ */
    .tabs { display: flex; gap: 0; border-bottom: 1px solid var(--border); margin-bottom: 22px; }
    .tab {
      padding: 9px 18px; cursor: pointer; font-size: 12px; font-weight: 600;
      color: var(--text3); border-bottom: 2px solid transparent; margin-bottom: -1px;
      transition: all .15s; text-transform: uppercase; letter-spacing: .08em;
    }
    .tab:hover { color: var(--text2); }
    .tab.active { color: var(--gold-light); border-bottom-color: var(--gold); }

    /* ════════════════════ LOADER ════════════════════ */
    .loader { display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 80px; gap: 14px; }
    .spin {
      width: 32px; height: 32px; border: 2px solid var(--border);
      border-top-color: var(--gold); border-radius: 50%;
      animation: spin 1s linear infinite;
    }
    .loader-text { font-size: 13px; color: var(--text3); }
    @keyframes spin { to { transform: rotate(360deg); } }

    /* ════════════════════ MISC ════════════════════ */
    .cor-dot {
      width: 11px; height: 11px; border-radius: 3px;
      display: inline-block; border: 1px solid rgba(255,255,255,.15);
      vertical-align: middle;
    }
    .code-block {
      background: #0a0a0d; border: 1px solid var(--border);
      border-radius: var(--radius); padding: 16px;
      font-family: var(--font-mono); font-size: 12px; color: var(--gold-light);
      overflow-x: auto; white-space: pre; margin: 10px 0; line-height: 1.7;
    }
    .step { display: flex; gap: 12px; align-items: flex-start; margin-bottom: 16px; }
    .step-num {
      min-width: 26px; height: 26px; background: var(--gold); color: #0f0f13;
      border-radius: 6px; display: flex; align-items: center; justify-content: center;
      font-weight: 700; font-size: 12px; flex-shrink: 0; margin-top: 1px;
    }
    .step-text { font-size: 14px; color: var(--text); line-height: 1.6; }
    .divider {
      display: flex; align-items: center; gap: 10px; margin: 20px 0;
      font-size: 11px; color: var(--text3); text-transform: uppercase; letter-spacing: .1em;
    }
    .divider::before, .divider::after { content: ''; flex: 1; height: 1px; background: var(--border); }
    .overlay { position: fixed; inset: 0; background: rgba(0,0,0,.6); z-index: 99; display: none; }
    .overlay.show { display: block; }

    /* pick list */
    .pick-item {
      display: flex; align-items: center; gap: 12px; padding: 11px 16px;
      border-bottom: 1px solid var(--border); cursor: pointer; transition: background .12s;
    }
    .pick-item:last-child { border-bottom: none; }
    .pick-item:hover { background: rgba(255,255,255,.03); }
    .pick-item.picked { background: var(--gold-dim); }
    .pick-check {
      width: 18px; height: 18px; border-radius: 4px; border: 1.5px solid var(--border);
      display: flex; align-items: center; justify-content: center; flex-shrink: 0; transition: all .12s;
    }
    .pick-item.picked .pick-check { background: var(--gold); border-color: var(--gold); }
    .pick-check-icon { display: none; font-size: 11px; color: #0f0f13; font-weight: 800; }
    .pick-item.picked .pick-check-icon { display: block; }

    /* scrollbar */
    ::-webkit-scrollbar { width: 5px; height: 5px; }
    ::-webkit-scrollbar-track { background: transparent; }
    ::-webkit-scrollbar-thumb { background: rgba(255,255,255,.1); border-radius: 3px; }
    ::-webkit-scrollbar-thumb:hover { background: rgba(255,255,255,.2); }

    /* section headers */
    .page-header {
      font-family: var(--font-head); font-size: 24px; font-weight: 700;
      color: var(--text); line-height: 1;
    }
    .page-sub { font-size: 13px; color: var(--text3); margin-top: 4px; }

    /* responsive */
    @media (max-width: 860px) {
      #sidebar { position: fixed; left: -260px; height: 100%; z-index: 100; transition: left .25s; }
      #sidebar.mobile-open { left: 0; width: var(--sidebar-w) !important; min-width: var(--sidebar-w) !important; }
      #menu-btn { display: inline-flex !important; }
      .form-row { grid-template-columns: 1fr; }
      #content { padding: 16px; }
    }

    /* bar chart helper */
    .bar-fill { transition: width .5s ease; }
  </style>
</head>
<body>

<div id="config-banner">
  ⚠️ Firebase não configurado — dados salvos apenas localmente.
  <a href="#" onclick="document.querySelector('[data-page=config]').click();return false;">Configurar agora →</a>
  <button onclick="this.parentElement.style.display='none'" style="background:none;border:none;cursor:pointer;margin-left:12px;font-size:16px;color:#0f0f13">✕</button>
</div>

<div id="toast-container"></div>
<div class="overlay" id="overlay" onclick="closeSidebar()"></div>

<div id="app">
  <!-- ══ SIDEBAR ══ -->
  <aside id="sidebar">
    <div id="sidebar-header">
      <div class="logo-icon-wrap">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M2 20h20M6 20V10M10 20V4M14 20V10M18 20V4"/></svg>
      </div>
      <div class="logo-texts">
        <div class="logo-eyebrow">Universidade de Artes</div>
        <div class="logo-name">Acervo Teatro</div>
      </div>
    </div>

    <nav id="nav">
      <div class="nav-section-label">Principal</div>

      <div class="nav-item active" data-page="dashboard">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/><rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/></svg>
        <span class="nav-label">Dashboard</span>
      </div>

      <div class="nav-item" data-page="figurinos">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20.38 3.46L16 2a4 4 0 0 1-8 0L3.62 3.46a2 2 0 0 0-1.34 2.23l.58 3.57a1 1 0 0 0 .99.84H6v10c0 1.1.9 2 2 2h8a2 2 0 0 0 2-2V10h2.15a1 1 0 0 0 .99-.84l.58-3.57a2 2 0 0 0-1.34-2.23z"/></svg>
        <span class="nav-label">Figurinos</span>
      </div>

      <div class="nav-section-label">Gestão</div>

      <div class="nav-item" data-page="alunos">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>
        <span class="nav-label">Alunos</span>
      </div>

      <div class="nav-item" data-page="emprestimos">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3M3 16v3a2 2 0 0 0 2 2h3m8 0h3a2 2 0 0 0 2-2v-3"/><path d="M7 12l5 5 5-5M12 7v10"/></svg>
        <span class="nav-label">Empréstimos</span>
      </div>

      <div class="nav-item" data-page="devolucoes">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/><path d="M3 3v5h5"/></svg>
        <span class="nav-label">Devoluções</span>
      </div>

      <div class="nav-section-label">Relatórios</div>

      <div class="nav-item" data-page="historico">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/></svg>
        <span class="nav-label">Histórico</span>
      </div>

      <div class="nav-item" data-page="relatorios">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/></svg>
        <span class="nav-label">Relatórios</span>
      </div>

      <div class="nav-item" data-page="analises">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="20" x2="18" y2="10"/><line x1="12" y1="20" x2="12" y2="4"/><line x1="6" y1="20" x2="6" y2="14"/></svg>
        <span class="nav-label">Análises</span>
      </div>

      <div class="nav-section-label">Sistema</div>

      <div class="nav-item" data-page="config">
        <svg class="nav-icon-svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1-2.83 2.83l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-4 0v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83-2.83l.06-.06A1.65 1.65 0 0 0 4.68 15a1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1 0-4h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 2.83-2.83l.06.06A1.65 1.65 0 0 0 9 4.68a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 4 0v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 2.83l-.06.06A1.65 1.65 0 0 0 19.4 9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 0 4h-.09a1.65 1.65 0 0 0-1.51 1z"/></svg>
        <span class="nav-label">Configurações</span>
      </div>
    </nav>

    <div id="sidebar-footer">
      <div id="sidebar-toggle" onclick="toggleSidebar()">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="3" y1="12" x2="21" y2="12"/><line x1="3" y1="6" x2="21" y2="6"/><line x1="3" y1="18" x2="21" y2="18"/></svg>
        <span class="toggle-label">Recolher menu</span>
      </div>
    </div>
  </aside>

  <!-- ══ MAIN ══ -->
  <div id="main">
    <div id="topbar">
      <button class="btn-icon" id="menu-btn" onclick="openSidebar()" style="display:none">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="3" y1="12" x2="21" y2="12"/><line x1="3" y1="6" x2="21" y2="6"/><line x1="3" y1="18" x2="21" y2="18"/></svg>
      </button>
      <div style="flex:1">
        <div id="page-title">Dashboard</div>
      </div>
      <div id="topbar-actions"></div>
    </div>
    <div id="content">
      <div class="loader"><div class="spin"></div><span class="loader-text">Carregando...</span></div>
    </div>
  </div>
</div>

<script>
// ════════════════════════════════════════════════════════════
// STATE & LOCAL STORAGE FALLBACK
// ════════════════════════════════════════════════════════════
const state = {
  page: 'dashboard',
  figurinos: [],
  alunos: [],
  emprestimos: [],
  historico: [],
  filtros: { busca: '', categoria: '', cor: '', tamanho: '', estado: '' },
  unsubs: [],
};

// Modo local (sem Firebase)
const LS = {
  get(key) { try { return JSON.parse(localStorage.getItem(key) || '[]'); } catch { return []; } },
  set(key, val) { localStorage.setItem(key, JSON.stringify(val)); },
};

window.usarModoLocal = (col, dados) => {
  const arr = LS.get(col);
  const item = { ...dados, id: Date.now().toString(), criadoEm: new Date().toISOString() };
  arr.unshift(item);
  LS.set(col, arr);
  state[col] = arr;
  render();
};
window.atualizarLocal = (col, id, dados) => {
  const arr = LS.get(col).map(i => i.id === id ? { ...i, ...dados } : i);
  LS.set(col, arr);
  state[col] = arr;
  render();
};
window.deletarLocal = (col, id) => {
  const arr = LS.get(col).filter(i => i.id !== id);
  LS.set(col, arr);
  state[col] = arr;
  render();
};
window.ouvirLocal = (col, cb) => {
  state[col] = LS.get(col);
  cb(state[col]);
};

// ════════════════════════════════════════════════════════════
// FIREBASE INIT
// ════════════════════════════════════════════════════════════
function initFirebase() {
  const FB = window.Firebase;
  if (!FB) return;

  if (FB.isConfigured()) {
    document.getElementById('config-banner').classList.remove('show');
  }

  state.unsubs.forEach(u => typeof u === 'function' && u());
  state.unsubs = [];

  state.unsubs.push(FB.ouvirFigurinos(data => { state.figurinos = data; render(); }));
  state.unsubs.push(FB.ouvirAlunos(data => { state.alunos = data; render(); }));
  state.unsubs.push(FB.ouvirEmprestimos(data => {
    state.emprestimos = data;
    // Auto-marcar atrasados
    data.forEach(e => {
      if (e.status === 'ativo' && e.dataDevolucao && new Date(e.dataDevolucao) < new Date()) {
        if (FB.isConfigured()) FB.atualizarEmprestimo(e.id, { status: 'atrasado' });
      }
    });
    render();
  }));
}

window.addEventListener('firebase-ready', () => {
  setTimeout(initFirebase, 100);
});
// Fallback se firebase-ready já disparou
setTimeout(() => {
  if (!state.figurinos.length && !state.alunos.length) initFirebase();
}, 800);

// ════════════════════════════════════════════════════════════
// NAVIGATION
// ════════════════════════════════════════════════════════════
document.getElementById('nav').addEventListener('click', e => {
  const item = e.target.closest('.nav-item');
  if (!item) return;
  const page = item.dataset.page;
  if (page) navigate(page);
  closeSidebar();
});

function navigate(page, params = {}) {
  state.page = page;
  state.params = params;
  document.querySelectorAll('.nav-item').forEach(el =>
    el.classList.toggle('active', el.dataset.page === page));
  render();
}

function toggleSidebar() {
  const sb = document.getElementById('sidebar');
  sb.classList.toggle('collapsed');
}
function openSidebar() {
  document.getElementById('sidebar').classList.add('mobile-open');
  document.getElementById('overlay').classList.add('show');
}
function closeSidebar() {
  document.getElementById('sidebar').classList.remove('mobile-open');
  document.getElementById('overlay').classList.remove('show');
}

// ════════════════════════════════════════════════════════════
// TOAST
// ════════════════════════════════════════════════════════════
function toast(msg, type = 'success') {
  const el = document.createElement('div');
  el.className = `toast ${type}`;
  const icons = { success: '✅', error: '❌', info: 'ℹ️' };
  el.innerHTML = `<span>${icons[type]||''}</span><span>${msg}</span>`;
  document.getElementById('toast-container').appendChild(el);
  setTimeout(() => el.remove(), 3500);
}

// ════════════════════════════════════════════════════════════
// RENDER ENGINE
// ════════════════════════════════════════════════════════════
function render() {
  const titles = {
    dashboard: 'Dashboard', figurinos: 'Figurinos', alunos: 'Alunos',
    emprestimos: 'Empréstimos', devolucoes: 'Devoluções',
    historico: 'Histórico', relatorios: 'Relatórios Semestrais',
    analises: 'Análises', config: 'Configurações',
    'figurino-novo': 'Novo Figurino', 'figurino-editar': 'Editar Figurino',
    'aluno-novo': 'Novo Aluno', 'aluno-editar': 'Editar Aluno',
    'emprestimo-novo': 'Novo Empréstimo',
  };
  document.getElementById('page-title').textContent = titles[state.page] || state.page;

  const content = document.getElementById('content');
  const actions = document.getElementById('topbar-actions');
  actions.innerHTML = '';

  switch(state.page) {
    case 'dashboard': content.innerHTML = renderDashboard(); break;
    case 'figurinos': content.innerHTML = renderFigurinos(); actions.innerHTML = `<button class="btn btn-primary btn-sm" onclick="navigate('figurino-novo')">＋ Novo Figurino</button>`; break;
    case 'figurino-novo': content.innerHTML = renderFigurinoForm(); break;
    case 'figurino-editar': content.innerHTML = renderFigurinoForm(state.params?.id); break;
    case 'alunos': content.innerHTML = renderAlunos(); actions.innerHTML = `<button class="btn btn-primary btn-sm" onclick="navigate('aluno-novo')">＋ Novo Aluno</button>`; break;
    case 'aluno-novo': content.innerHTML = renderAlunoForm(); break;
    case 'aluno-editar': content.innerHTML = renderAlunoForm(state.params?.id); break;
    case 'emprestimos': content.innerHTML = renderEmprestimos(); actions.innerHTML = `<button class="btn btn-primary btn-sm" onclick="navigate('emprestimo-novo')">＋ Novo Empréstimo</button>`; break;
    case 'emprestimo-novo': content.innerHTML = renderEmprestimoForm(); break;
    case 'devolucoes': content.innerHTML = renderDevolucoes(); break;
    case 'historico': content.innerHTML = renderHistorico(); break;
    case 'relatorios': content.innerHTML = renderRelatorios(); break;
    case 'analises': content.innerHTML = renderAnalises(); break;
    case 'config': content.innerHTML = renderConfig(); break;
    default: content.innerHTML = '<div class="empty"><div class="empty-icon">🔍</div><p class="empty-text">Página não encontrada</p></div>';
  }
}

// ════════════════════════════════════════════════════════════
// HELPERS
// ════════════════════════════════════════════════════════════
const CORES_HEX = {
  vermelho:'#ef4444', azul:'#3b82f6', verde:'#10b981', amarelo:'#fbbf24',
  preto:'#374151', branco:'#e5e7eb', roxo:'#8b5cf6', rosa:'#ec4899',
  laranja:'#f97316', marrom:'#92400e', cinza:'#6b7280', dourado:'#d97706',
  prata:'#9ca3af', outro:'#6b7280'
};
const ESTADOS_LABEL = { novo:'Novo', bom:'Bom', desgastado:'Desgastado', otimo:'Ótimo', regular:'Regular', ruim:'Ruim' };
const CATEGORIAS_LABEL = { figurino:'Figurino', acessorio:'Acessório', cena:'Cena/Adereço' };
const TAMANHOS = ['PP','P','M','G','GG','XG','U'];
const CORES = Object.keys(CORES_HEX);
const CATEGORIAS = Object.keys(CATEGORIAS_LABEL);
const ESTADOS = ['novo','bom','desgastado','otimo','regular','ruim'];

function formatDate(d) {
  if (!d) return '—';
  const dt = d?.toDate ? d.toDate() : (typeof d === 'string' ? new Date(d) : d);
  if (isNaN(dt)) return '—';
  return dt.toLocaleDateString('pt-BR');
}
function diasAte(d) {
  if (!d) return null;
  const dt = d?.toDate ? d.toDate() : (typeof d === 'string' ? new Date(d) : d);
  return Math.ceil((dt - new Date()) / 86400000);
}

// ════════════════════════════════════════════════════════════
// PAGES
// ════════════════════════════════════════════════════════════

function renderDashboard() {
  const totalFig = state.figurinos.reduce((s,f) => s + (f.quantidade||1), 0);
  const disponiveis = state.figurinos.reduce((s,f) => s + (f.quantidadeDisponivel||f.quantidade||1), 0);
  const ativos = state.emprestimos.filter(e => e.status === 'ativo').length;
  const atrasados = state.emprestimos.filter(e => e.status === 'atrasado').length;
  const alunosAtivos = state.alunos.filter(a => a.ativo !== false).length;

  const recentes = [...state.figurinos].slice(0, 4);
  const empRecentes = [...state.emprestimos].slice(0, 5);

  return `
  <div style="max-width:1100px">
    <div class="card-grid" style="grid-template-columns:repeat(auto-fill,minmax(220px,1fr));margin-bottom:24px">
      ${statCard('👗','Total de Figurinos', totalFig, 'rgba(59,130,246,.2)', 'figurinos')}
      ${statCard('✅','Disponíveis', disponiveis, 'rgba(34,197,94,.2)', 'figurinos')}
      ${statCard('🤝','Empréstimos Ativos', ativos, 'rgba(240,165,0,.2)', 'emprestimos')}
      ${statCard('⚠️','Devoluções Atrasadas', atrasados, 'rgba(239,68,68,.2)', 'devolucoes')}
      ${statCard('👥','Alunos Cadastrados', alunosAtivos, 'rgba(168,85,247,.2)', 'alunos')}
    </div>

    <div style="display:grid;grid-template-columns:1fr 1fr;gap:20px">
      <div class="card">
        <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px">
          <strong style="font-family:var(--font-head)">Figurinos Recentes</strong>
          <button class="btn btn-secondary btn-sm" onclick="navigate('figurinos')">Ver todos</button>
        </div>
        ${recentes.length === 0 ? `<div class="empty" style="padding:30px"><div class="empty-icon">👗</div><p class="empty-text" style="font-size:14px">Nenhum figurino ainda</p><button class="btn btn-primary btn-sm" onclick="navigate('figurino-novo')">Adicionar</button></div>` : `
        <div style="display:flex;flex-direction:column;gap:10px">
          ${recentes.map(f => `
            <div style="display:flex;align-items:center;gap:12px;padding:10px;background:var(--surface2);border-radius:8px;cursor:pointer" onclick="navigate('figurinos')">
              <div style="width:40px;height:40px;border-radius:8px;overflow:hidden;background:var(--border);display:flex;align-items:center;justify-content:center;font-size:18px;flex-shrink:0">
                ${f.imagemUrl ? `<img src="${f.imagemUrl}" style="width:100%;height:100%;object-fit:cover">` : '👗'}
              </div>
              <div style="flex:1;min-width:0">
                <div style="font-weight:500;font-size:14px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">${f.nome}</div>
                <div style="font-size:12px;color:var(--text2)">${CATEGORIAS_LABEL[f.categoria]||f.categoria||''} · ${f.quantidade||1} un.</div>
              </div>
              ${badgeDisp(f.quantidadeDisponivel ?? f.quantidade ?? 1)}
            </div>`).join('')}
        </div>`}
      </div>

      <div class="card">
        <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px">
          <strong style="font-family:var(--font-head)">Últimos Empréstimos</strong>
          <button class="btn btn-secondary btn-sm" onclick="navigate('emprestimos')">Ver todos</button>
        </div>
        ${empRecentes.length === 0 ? `<div class="empty" style="padding:30px"><div class="empty-icon">🤝</div><p class="empty-text" style="font-size:14px">Nenhum empréstimo ainda</p></div>` : `
        <div style="display:flex;flex-direction:column;gap:10px">
          ${empRecentes.map(e => `
            <div style="display:flex;align-items:center;gap:12px;padding:10px;background:var(--surface2);border-radius:8px">
              <div style="width:36px;height:36px;border-radius:50%;background:var(--border);display:flex;align-items:center;justify-content:center;font-size:16px;flex-shrink:0">👤</div>
              <div style="flex:1">
                <div style="font-weight:500;font-size:14px">${e.alunoNome||'—'}</div>
                <div style="font-size:12px;color:var(--text2)">Dev: ${formatDate(e.dataDevolucao)}</div>
              </div>
              ${statusBadge(e.status)}
            </div>`).join('')}
        </div>`}
      </div>
    </div>
  </div>`;
}

function statCard(icon, label, val, bg, page) {
  return `<div class="stat-card" onclick="navigate('${page}')" style="cursor:pointer">
    <div class="stat-icon" style="background:${bg}">${icon}</div>
    <div><div class="stat-value">${val}</div><div class="stat-label">${label}</div></div>
  </div>`;
}
function badgeDisp(qty) {
  if (qty <= 0) return `<span class="badge badge-red">Esgotado</span>`;
  return `<span class="badge badge-green">${qty} disp.</span>`;
}
function statusBadge(s) {
  const map = { ativo:'badge-blue', atrasado:'badge-red', devolvido:'badge-green' };
  const lbl = { ativo:'Ativo', atrasado:'Atrasado', devolvido:'Devolvido' };
  return `<span class="badge ${map[s]||'badge-gray'}">${lbl[s]||s}</span>`;
}

// ── FIGURINOS ────────────────────────────────────────────────
function renderFigurinos() {
  const { busca, categoria, cor, tamanho, estado } = state.filtros;
  let lista = state.figurinos;
  if (busca) lista = lista.filter(f => f.nome.toLowerCase().includes(busca.toLowerCase()));
  if (categoria) lista = lista.filter(f => f.categoria === categoria);
  if (cor) lista = lista.filter(f => f.cor === cor);
  if (tamanho) lista = lista.filter(f => Array.isArray(f.tamanhos) ? f.tamanhos.includes(tamanho) : f.tamanho === tamanho);
  if (estado) lista = lista.filter(f => f.estado === estado);

  return `
  <div>
    <div class="card" style="margin-bottom:20px">
      <div class="search-wrap" style="margin-bottom:12px">
        <span class="search-icon">🔍</span>
        <input class="form-input" placeholder="Buscar figurino..." value="${busca}" oninput="state.filtros.busca=this.value;render()">
      </div>
      <div style="display:flex;gap:8px;flex-wrap:wrap">
        <select class="form-select" style="width:auto;flex:1;min-width:120px" onchange="state.filtros.categoria=this.value;render()">
          <option value="">Categoria</option>
          ${CATEGORIAS.map(c => `<option value="${c}" ${categoria===c?'selected':''}>${CATEGORIAS_LABEL[c]}</option>`).join('')}
        </select>
        <select class="form-select" style="width:auto;flex:1;min-width:100px" onchange="state.filtros.tamanho=this.value;render()">
          <option value="">Tamanho</option>
          ${TAMANHOS.map(t => `<option value="${t}" ${tamanho===t?'selected':''}>${t}</option>`).join('')}
        </select>
        <select class="form-select" style="width:auto;flex:1;min-width:100px" onchange="state.filtros.estado=this.value;render()">
          <option value="">Estado</option>
          ${ESTADOS.map(e => `<option value="${e}" ${estado===e?'selected':''}>${ESTADOS_LABEL[e]}</option>`).join('')}
        </select>
        ${(busca||categoria||cor||tamanho||estado) ? `<button class="btn btn-secondary btn-sm" onclick="state.filtros={busca:'',categoria:'',cor:'',tamanho:'',estado:''};render()">✕ Limpar</button>` : ''}
      </div>
    </div>

    ${lista.length === 0 ? `
      <div class="empty">
        <div class="empty-icon">👗</div>
        <p class="empty-text">Nenhum figurino encontrado</p>
        <button class="btn btn-primary" onclick="navigate('figurino-novo')">＋ Adicionar Figurino</button>
      </div>` : `
      <div class="figurino-grid">
        ${lista.map(f => renderFigurinoCard(f)).join('')}
      </div>`}
  </div>`;
}

function renderFigurinoCard(f) {
  const disp = f.quantidadeDisponivel ?? f.quantidade ?? 1;
  return `
  <div class="figurino-card">
    <div class="figurino-img" onclick="abrirFigurinoModal('${f.id}')">
      ${f.imagemUrl ? `<img src="${f.imagemUrl}" onerror="this.parentElement.innerHTML='👗'">` : '👗'}
    </div>
    <div class="figurino-info">
      <div class="figurino-nome" onclick="abrirFigurinoModal('${f.id}')" style="cursor:pointer">${f.nome}</div>
      <div class="figurino-meta">
        <span>${CATEGORIAS_LABEL[f.categoria]||f.categoria||'—'}</span>
        ${f.tamanhos?.length ? `<span>${f.tamanhos.join(', ')}</span>` : (f.tamanho ? `<span>${f.tamanho}</span>` : '')}
        ${f.cor ? `<span><span class="cor-dot" style="background:${CORES_HEX[f.cor]||f.cor}"></span> ${f.cor}</span>` : ''}
      </div>
      <div style="display:flex;align-items:center;justify-content:space-between;margin-top:10px">
        ${badgeDisp(disp)}
        <div style="display:flex;gap:6px">
          <button class="btn btn-secondary btn-sm" onclick="navigate('figurino-editar',{id:'${f.id}'})">✏️</button>
          <button class="btn btn-danger btn-sm" onclick="confirmarDeletarFigurino('${f.id}','${f.nome.replace(/'/g,"\\'")}')">🗑️</button>
        </div>
      </div>
    </div>
  </div>`;
}

function abrirFigurinoModal(id) {
  const f = state.figurinos.find(x => x.id === id);
  if (!f) return;
  const disp = f.quantidadeDisponivel ?? f.quantidade ?? 1;
  const modal = document.createElement('div');
  modal.className = 'modal-bg';
  modal.innerHTML = `
  <div class="modal" style="max-width:480px">
    <div class="modal-header">
      <div class="modal-title">${f.nome}</div>
      <button class="modal-close" onclick="this.closest('.modal-bg').remove()">✕</button>
    </div>
    <div class="modal-body">
      ${f.imagemUrl ? `<img src="${f.imagemUrl}" style="width:100%;height:220px;object-fit:cover;border-radius:8px;margin-bottom:16px">` : ''}
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;font-size:14px">
        <div><span style="color:var(--text2)">Categoria</span><br><strong>${CATEGORIAS_LABEL[f.categoria]||f.categoria||'—'}</strong></div>
        <div><span style="color:var(--text2)">Tamanhos</span><br><strong>${f.tamanhos?.join(', ')||f.tamanho||'—'}</strong></div>
        <div><span style="color:var(--text2)">Cor</span><br><strong style="display:flex;align-items:center;gap:6px"><span class="cor-dot" style="background:${CORES_HEX[f.cor]||'#888'}"></span>${f.cor||'—'}</strong></div>
        <div><span style="color:var(--text2)">Estado</span><br><strong>${ESTADOS_LABEL[f.estado]||f.estado||'—'}</strong></div>
        <div><span style="color:var(--text2)">Quantidade Total</span><br><strong>${f.quantidade||1}</strong></div>
        <div><span style="color:var(--text2)">Disponíveis</span><br><strong>${disp}</strong></div>
        ${f.localizacao ? `<div style="grid-column:span 2"><span style="color:var(--text2)">Localização</span><br><strong>${f.localizacao}</strong></div>` : ''}
        ${f.descricao ? `<div style="grid-column:span 2"><span style="color:var(--text2)">Descrição</span><br><span>${f.descricao}</span></div>` : ''}
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn btn-secondary" onclick="this.closest('.modal-bg').remove()">Fechar</button>
      <button class="btn btn-primary" onclick="this.closest('.modal-bg').remove();navigate('figurino-editar',{id:'${id}'})">✏️ Editar</button>
    </div>
  </div>`;
  document.body.appendChild(modal);
  modal.addEventListener('click', e => { if(e.target===modal) modal.remove(); });
}

function confirmarDeletarFigurino(id, nome) {
  const modal = document.createElement('div');
  modal.className = 'modal-bg';
  modal.innerHTML = `
  <div class="modal" style="max-width:400px">
    <div class="modal-header"><div class="modal-title">Confirmar exclusão</div></div>
    <div class="modal-body"><p>Tem certeza que deseja remover <strong>"${nome}"</strong>? Esta ação não pode ser desfeita.</p></div>
    <div class="modal-footer">
      <button class="btn btn-secondary" onclick="this.closest('.modal-bg').remove()">Cancelar</button>
      <button class="btn btn-danger" onclick="deletarFigurino('${id}');this.closest('.modal-bg').remove()">Excluir</button>
    </div>
  </div>`;
  document.body.appendChild(modal);
}

async function deletarFigurino(id) {
  const f = state.figurinos.find(x => x.id === id);
  const FB = window.Firebase;
  if (!FB) return;
  try {
    await FB.deletarFigurino(id, f?.fotoPath);
    toast(`"${f?.nome}" removido com sucesso`);
    adicionarHistorico('remocao_figurino', `Figurino "${f?.nome}" removido`);
  } catch(e) {
    toast('Erro ao remover: ' + e.message, 'error');
  }
}

function renderFigurinoForm(id) {
  const f = id ? state.figurinos.find(x => x.id === id) : null;
  const v = f || {};
  const tamanhosSel = v.tamanhos || (v.tamanho ? [v.tamanho] : []);

  return `
  <div style="max-width:640px">
    <button class="btn btn-secondary btn-sm" onclick="navigate('figurinos')" style="margin-bottom:20px">← Voltar</button>
    <div class="card">
      <h2 style="font-family:var(--font-head);margin-bottom:24px">${id ? 'Editar Figurino' : 'Novo Figurino'}</h2>
      <form id="figurino-form" onsubmit="salvarFigurinoForm(event,'${id||''}')">

        <div class="form-group">
          <label class="form-label">Foto do Figurino</label>
          <div class="upload-area" id="upload-area" onclick="document.getElementById('foto-input').click()" ondrop="handleDrop(event)" ondragover="event.preventDefault();this.classList.add('drag')" ondragleave="this.classList.remove('drag')">
            ${v.imagemUrl ? `<img src="${v.imagemUrl}" class="upload-preview" id="img-preview">` : `<div id="img-preview-placeholder"><div style="font-size:32px;margin-bottom:8px">📷</div><div style="color:var(--text2);font-size:14px">Clique ou arraste uma imagem<br><span style="font-size:12px">JPG, PNG, WebP — máx 5MB</span></div></div>`}
          </div>
          <input type="file" id="foto-input" accept="image/jpeg,image/png,image/webp" style="display:none" onchange="previewFoto(this)">
          <div class="form-error" id="foto-error"></div>
        </div>

        <div class="form-group">
          <label class="form-label">Nome *</label>
          <input class="form-input" name="nome" placeholder="Ex: Vestido Medieval Vermelho" value="${v.nome||''}" required>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Categoria *</label>
            <select class="form-select" name="categoria" required>
              <option value="">Selecione...</option>
              ${CATEGORIAS.map(c => `<option value="${c}" ${v.categoria===c?'selected':''}>${CATEGORIAS_LABEL[c]}</option>`).join('')}
            </select>
          </div>
          <div class="form-group">
            <label class="form-label">Estado de Conservação</label>
            <select class="form-select" name="estado">
              ${ESTADOS.map(e => `<option value="${e}" ${(v.estado||'bom')===e?'selected':''}>${ESTADOS_LABEL[e]}</option>`).join('')}
            </select>
          </div>
        </div>

        <div class="form-group">
          <label class="form-label">Tamanhos disponíveis</label>
          <div class="chips">
            ${TAMANHOS.map(t => `<div class="chip ${tamanhosSel.includes(t)?'selected':''}" onclick="toggleTamanho(this,'${t}')" data-val="${t}">${t}</div>`).join('')}
          </div>
          <input type="hidden" id="tamanhos-hidden" name="tamanhos" value="${tamanhosSel.join(',')}">
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Cor Principal</label>
            <select class="form-select" name="cor">
              <option value="">Selecione...</option>
              ${CORES.map(c => `<option value="${c}" ${v.cor===c?'selected':''}>${c.charAt(0).toUpperCase()+c.slice(1)}</option>`).join('')}
            </select>
          </div>
          <div class="form-group">
            <label class="form-label">Quantidade Total</label>
            <input class="form-input" type="number" name="quantidade" min="1" value="${v.quantidade||1}">
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Quantidade Disponível</label>
            <input class="form-input" type="number" name="quantidadeDisponivel" min="0" value="${v.quantidadeDisponivel??v.quantidade??1}">
          </div>
          <div class="form-group">
            <label class="form-label">Localização no Acervo</label>
            <input class="form-input" name="localizacao" placeholder="Ex: Prateleira A, Caixão 3" value="${v.localizacao||''}">
          </div>
        </div>

        <div class="form-group">
          <label class="form-label">Descrição</label>
          <textarea class="form-textarea" name="descricao" placeholder="Detalhes sobre o figurino...">${v.descricao||''}</textarea>
        </div>

        <div style="display:flex;gap:10px;margin-top:8px">
          <button type="button" class="btn btn-secondary" onclick="navigate('figurinos')">Cancelar</button>
          <button type="submit" class="btn btn-primary" id="submit-btn">💾 ${id ? 'Salvar Alterações' : 'Cadastrar Figurino'}</button>
        </div>
      </form>
    </div>
  </div>`;
}

function toggleTamanho(el, val) {
  el.classList.toggle('selected');
  const hidden = document.getElementById('tamanhos-hidden');
  const arr = hidden.value ? hidden.value.split(',').filter(Boolean) : [];
  const idx = arr.indexOf(val);
  if (idx >= 0) arr.splice(idx, 1); else arr.push(val);
  hidden.value = arr.join(',');
}

function previewFoto(input) {
  const file = input.files[0];
  if (!file) return;
  if (file.size > 5*1024*1024) { document.getElementById('foto-error').textContent = 'Imagem muito grande (máx 5MB)'; return; }
  const reader = new FileReader();
  reader.onload = e => {
    const area = document.getElementById('upload-area');
    area.innerHTML = `<img src="${e.target.result}" class="upload-preview" id="img-preview" style="display:block">`;
  };
  reader.readAsDataURL(file);
}

function handleDrop(e) {
  e.preventDefault();
  e.currentTarget.classList.remove('drag');
  const file = e.dataTransfer.files[0];
  if (file) {
    const dt = new DataTransfer();
    dt.items.add(file);
    document.getElementById('foto-input').files = dt.files;
    previewFoto(document.getElementById('foto-input'));
  }
}

async function salvarFigurinoForm(e, id) {
  e.preventDefault();
  const form = e.target;
  const btn = document.getElementById('submit-btn');
  btn.disabled = true; btn.textContent = '⏳ Salvando...';

  const dados = {
    nome: form.nome.value.trim(),
    categoria: form.categoria.value,
    estado: form.estado.value,
    tamanhos: form.tamanhos.value ? form.tamanhos.value.split(',').filter(Boolean) : [],
    cor: form.cor.value,
    quantidade: parseInt(form.quantidade.value) || 1,
    quantidadeDisponivel: parseInt(form.quantidadeDisponivel.value) ?? parseInt(form.quantidade.value) ?? 1,
    localizacao: form.localizacao.value.trim(),
    descricao: form.descricao.value.trim(),
  };

  const FB = window.Firebase;
  if (!FB) { toast('Firebase não inicializado', 'error'); btn.disabled=false; return; }

  try {
    // Upload foto se tiver
    const fotoInput = document.getElementById('foto-input');
    if (fotoInput?.files[0]) {
      toast('Fazendo upload da imagem...', 'info');
      if (FB.isConfigured()) {
        dados.imagemUrl = await FB.uploadImagem(fotoInput.files[0]);
      } else {
        // Modo local: converter para data URL
        dados.imagemUrl = await fileToDataUrl(fotoInput.files[0]);
      }
    } else if (id) {
      const prev = state.figurinos.find(f => f.id === id);
      if (prev?.imagemUrl) dados.imagemUrl = prev.imagemUrl;
    }

    if (id) {
      await FB.atualizarFigurino(id, dados);
      toast(`Figurino "${dados.nome}" atualizado!`);
      adicionarHistorico('edicao_figurino', `Figurino "${dados.nome}" atualizado`);
    } else {
      await FB.salvarFigurino(dados);
      toast(`Figurino "${dados.nome}" cadastrado!`);
      adicionarHistorico('cadastro_figurino', `Figurino "${dados.nome}" adicionado ao acervo`);
    }
    navigate('figurinos');
  } catch(err) {
    toast('Erro: ' + err.message, 'error');
    btn.disabled=false; btn.textContent = '💾 Salvar';
  }
}

function fileToDataUrl(file) {
  return new Promise((res, rej) => {
    const r = new FileReader();
    r.onload = e => res(e.target.result);
    r.onerror = rej;
    r.readAsDataURL(file);
  });
}

// ── ALUNOS ───────────────────────────────────────────────────
function renderAlunos() {
  const busca = state.filtros.busca;
  let lista = state.alunos;
  if (busca) lista = lista.filter(a => a.nome.toLowerCase().includes(busca.toLowerCase()) || (a.matricula||'').includes(busca));

  return `
  <div>
    <div class="card" style="margin-bottom:20px">
      <div class="search-wrap">
        <span class="search-icon">🔍</span>
        <input class="form-input" placeholder="Buscar aluno por nome ou matrícula..." value="${busca}" oninput="state.filtros.busca=this.value;render()">
      </div>
    </div>

    ${lista.length === 0 ? `
      <div class="empty">
        <div class="empty-icon">👥</div>
        <p class="empty-text">Nenhum aluno encontrado</p>
        <button class="btn btn-primary" onclick="navigate('aluno-novo')">＋ Cadastrar Aluno</button>
      </div>` : `
    <div class="card" style="padding:0">
      <div class="table-wrap">
        <table>
          <thead><tr>
            <th>Nome</th><th>Matrícula</th><th>Email</th><th>Telefone</th><th>Status</th><th style="text-align:right">Ações</th>
          </tr></thead>
          <tbody>
            ${lista.map(a => `<tr>
              <td><strong>${a.nome}</strong></td>
              <td>${a.matricula||'—'}</td>
              <td>${a.email||'—'}</td>
              <td>${a.telefone||'—'}</td>
              <td><span class="badge ${a.ativo!==false?'badge-green':'badge-gray'}">${a.ativo!==false?'Ativo':'Inativo'}</span></td>
              <td style="text-align:right">
                <div style="display:flex;gap:6px;justify-content:flex-end">
                  <button class="btn btn-secondary btn-sm" onclick="navigate('aluno-editar',{id:'${a.id}'})">✏️</button>
                  <button class="btn btn-danger btn-sm" onclick="confirmarDeletarAluno('${a.id}','${a.nome.replace(/'/g,"\\'")}')">🗑️</button>
                </div>
              </td>
            </tr>`).join('')}
          </tbody>
        </table>
      </div>
    </div>`}
  </div>`;
}

function confirmarDeletarAluno(id, nome) {
  const modal = document.createElement('div');
  modal.className = 'modal-bg';
  modal.innerHTML = `<div class="modal" style="max-width:400px">
    <div class="modal-header"><div class="modal-title">Confirmar exclusão</div></div>
    <div class="modal-body"><p>Remover aluno <strong>"${nome}"</strong>?</p></div>
    <div class="modal-footer">
      <button class="btn btn-secondary" onclick="this.closest('.modal-bg').remove()">Cancelar</button>
      <button class="btn btn-danger" onclick="deletarAluno('${id}','${nome}');this.closest('.modal-bg').remove()">Excluir</button>
    </div>
  </div>`;
  document.body.appendChild(modal);
}

async function deletarAluno(id, nome) {
  const FB = window.Firebase;
  try {
    await FB.deletarAluno(id);
    toast(`Aluno "${nome}" removido`);
    adicionarHistorico('remocao_aluno', `Aluno "${nome}" removido`);
  } catch(e) { toast('Erro: '+e.message, 'error'); }
}

function renderAlunoForm(id) {
  const a = id ? state.alunos.find(x => x.id === id) : null;
  const v = a || {};
  return `
  <div style="max-width:560px">
    <button class="btn btn-secondary btn-sm" onclick="navigate('alunos')" style="margin-bottom:20px">← Voltar</button>
    <div class="card">
      <h2 style="font-family:var(--font-head);margin-bottom:24px">${id ? 'Editar Aluno' : 'Novo Aluno'}</h2>
      <form onsubmit="salvarAlunoForm(event,'${id||''}')">
        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Nome Completo *</label>
            <input class="form-input" name="nome" value="${v.nome||''}" required placeholder="Nome do aluno">
          </div>
          <div class="form-group">
            <label class="form-label">Matrícula</label>
            <input class="form-input" name="matricula" value="${v.matricula||''}" placeholder="Ex: 2024001">
          </div>
        </div>
        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Email</label>
            <input class="form-input" type="email" name="email" value="${v.email||''}" placeholder="email@exemplo.com">
          </div>
          <div class="form-group">
            <label class="form-label">Telefone</label>
            <input class="form-input" name="telefone" value="${v.telefone||''}" placeholder="(22) 99999-9999">
          </div>
        </div>
        <div class="form-group">
          <label class="form-label">Turma / Observações</label>
          <input class="form-input" name="turma" value="${v.turma||''}" placeholder="Ex: Turma 3A, Teatro Avançado...">
        </div>
        ${id ? `<div class="form-group">
          <label class="form-label">Status</label>
          <select class="form-select" name="ativo">
            <option value="true" ${v.ativo!==false?'selected':''}>Ativo</option>
            <option value="false" ${v.ativo===false?'selected':''}>Inativo</option>
          </select>
        </div>` : ''}
        <div style="display:flex;gap:10px;margin-top:8px">
          <button type="button" class="btn btn-secondary" onclick="navigate('alunos')">Cancelar</button>
          <button type="submit" class="btn btn-primary">💾 ${id ? 'Salvar' : 'Cadastrar'}</button>
        </div>
      </form>
    </div>
  </div>`;
}

async function salvarAlunoForm(e, id) {
  e.preventDefault();
  const form = e.target;
  const dados = {
    nome: form.nome.value.trim(),
    matricula: form.matricula.value.trim(),
    email: form.email.value.trim(),
    telefone: form.telefone.value.trim(),
    turma: form.turma.value.trim(),
    ativo: form.ativo ? form.ativo.value === 'true' : true,
  };
  const FB = window.Firebase;
  try {
    if (id) {
      await FB.atualizarAluno(id, dados);
      toast(`Aluno "${dados.nome}" atualizado!`);
      adicionarHistorico('edicao_aluno', `Aluno "${dados.nome}" atualizado`);
    } else {
      await FB.salvarAluno(dados);
      toast(`Aluno "${dados.nome}" cadastrado!`);
      adicionarHistorico('cadastro_aluno', `Aluno "${dados.nome}" cadastrado`);
    }
    navigate('alunos');
  } catch(err) { toast('Erro: '+err.message, 'error'); }
}

// ── EMPRÉSTIMOS ───────────────────────────────────────────────
function renderEmprestimos() {
  const lista = state.emprestimos.filter(e => e.status !== 'devolvido');
  return `
  <div>
    ${lista.length === 0 ? `
      <div class="empty">
        <div class="empty-icon">🤝</div>
        <p class="empty-text">Nenhum empréstimo ativo</p>
        <button class="btn btn-primary" onclick="navigate('emprestimo-novo')">＋ Criar Empréstimo</button>
      </div>` : `
    <div class="card" style="padding:0">
      <div class="table-wrap">
        <table>
          <thead><tr><th>Aluno</th><th>Figurinos</th><th>Retirada</th><th>Devolução Prev.</th><th>Status</th><th style="text-align:right">Ações</th></tr></thead>
          <tbody>
            ${lista.map(e => `<tr>
              <td><strong>${e.alunoNome||'—'}</strong></td>
              <td>${(e.itens||[]).map(i=>`<span style="font-size:12px;background:var(--surface2);padding:2px 6px;border-radius:4px;margin-right:4px">${i.nome||i.figurinoId}</span>`).join('')}</td>
              <td>${formatDate(e.dataRetirada)}</td>
              <td>${formatDate(e.dataDevolucao)}</td>
              <td>${statusBadge(e.status)}</td>
              <td style="text-align:right">
                ${e.status !== 'devolvido' ? `<button class="btn btn-primary btn-sm" onclick="registrarDevolucao('${e.id}','${(e.alunoNome||'').replace(/'/g,"\\'")}')">↩️ Devolver</button>` : ''}
              </td>
            </tr>`).join('')}
          </tbody>
        </table>
      </div>
    </div>`}
  </div>`;
}

function renderEmprestimoForm() {
  const alunos = state.alunos.filter(a => a.ativo !== false);
  const figurinosDisp = state.figurinos.filter(f => (f.quantidadeDisponivel ?? f.quantidade ?? 1) > 0);

  const hoje = new Date().toISOString().split('T')[0];
  const doisSemanas = new Date(Date.now() + 14*86400000).toISOString().split('T')[0];

  return `
  <div style="max-width:640px">
    <button class="btn btn-secondary btn-sm" onclick="navigate('emprestimos')" style="margin-bottom:20px">← Voltar</button>
    <div class="card">
      <h2 style="font-family:var(--font-head);margin-bottom:24px">Novo Empréstimo</h2>
      <form onsubmit="salvarEmprestimoForm(event)">

        <div class="form-group">
          <label class="form-label">Aluno *</label>
          ${alunos.length === 0
            ? `<div style="color:var(--text2);font-size:14px">Nenhum aluno cadastrado. <button type="button" class="btn btn-secondary btn-sm" onclick="navigate('aluno-novo')">Cadastrar aluno</button></div>`
            : `<select class="form-select" name="alunoId" required>
                <option value="">Selecione o aluno...</option>
                ${alunos.map(a => `<option value="${a.id}" data-nome="${a.nome}">${a.nome} ${a.matricula?'('+a.matricula+')':''}</option>`).join('')}
              </select>`}
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Data de Retirada</label>
            <input class="form-input" type="date" name="dataRetirada" value="${hoje}">
          </div>
          <div class="form-group">
            <label class="form-label">Devolução Prevista</label>
            <input class="form-input" type="date" name="dataDevolucao" value="${doisSemanas}" required>
          </div>
        </div>

        <div class="form-group">
          <label class="form-label">Figurinos *</label>
          <div id="figurinos-sel" style="border:1px solid var(--border);border-radius:8px;overflow:hidden">
            ${figurinosDisp.length === 0
              ? `<div style="padding:20px;text-align:center;color:var(--text2)">Nenhum figurino disponível</div>`
              : figurinosDisp.map(f => `
              <div class="figurino-pick" style="display:flex;align-items:center;gap:12px;padding:12px;border-bottom:1px solid var(--border);cursor:pointer" onclick="toggleFigurinoPick('${f.id}',this)">
                <div style="width:32px;height:32px;border-radius:6px;overflow:hidden;background:var(--surface2);display:flex;align-items:center;justify-content:center;font-size:16px;flex-shrink:0">
                  ${f.imagemUrl ? `<img src="${f.imagemUrl}" style="width:100%;height:100%;object-fit:cover">` : '👗'}
                </div>
                <div style="flex:1">
                  <div style="font-size:14px;font-weight:500">${f.nome}</div>
                  <div style="font-size:12px;color:var(--text2)">${f.tamanhos?.join(', ')||f.tamanho||''} · ${f.quantidadeDisponivel??f.quantidade??1} disponível(is)</div>
                </div>
                <div class="pick-check" style="width:22px;height:22px;border-radius:50%;border:2px solid var(--border);display:flex;align-items:center;justify-content:center;flex-shrink:0">
                  <span class="check-icon" style="display:none;color:var(--accent)">✓</span>
                </div>
              </div>`).join('')}
          </div>
          <input type="hidden" id="figurinos-hidden" name="figurinosIds">
        </div>

        <div class="form-group">
          <label class="form-label">Observações</label>
          <textarea class="form-textarea" name="observacoes" placeholder="Alguma observação sobre o empréstimo..."></textarea>
        </div>

        <div style="display:flex;gap:10px">
          <button type="button" class="btn btn-secondary" onclick="navigate('emprestimos')">Cancelar</button>
          <button type="submit" class="btn btn-primary">💾 Registrar Empréstimo</button>
        </div>
      </form>
    </div>
  </div>`;
}

const figurinosPick = new Set();
function toggleFigurinoPick(id, el) {
  if (figurinosPick.has(id)) {
    figurinosPick.delete(id);
    el.style.background = '';
    el.querySelector('.check-icon').style.display = 'none';
    el.querySelector('.pick-check').style.borderColor = 'var(--border)';
  } else {
    figurinosPick.add(id);
    el.style.background = 'rgba(240,165,0,.08)';
    el.querySelector('.check-icon').style.display = 'block';
    el.querySelector('.pick-check').style.borderColor = 'var(--accent)';
  }
  document.getElementById('figurinos-hidden').value = [...figurinosPick].join(',');
}

async function salvarEmprestimoForm(e) {
  e.preventDefault();
  const form = e.target;
  const sel = form.alunoId;
  const alunoId = sel.value;
  const alunoNome = sel.options[sel.selectedIndex]?.dataset.nome || '';
  const figurinosIds = form.figurinosIds.value.split(',').filter(Boolean);

  if (!figurinosIds.length) { toast('Selecione ao menos um figurino', 'error'); return; }

  const itens = figurinosIds.map(id => {
    const f = state.figurinos.find(x => x.id === id);
    return { figurinoId: id, nome: f?.nome || id, quantidade: 1 };
  });

  const dados = {
    alunoId, alunoNome,
    dataRetirada: form.dataRetirada.value,
    dataDevolucao: form.dataDevolucao.value,
    observacoes: form.observacoes.value.trim(),
    itens,
    status: 'ativo',
  };

  const FB = window.Firebase;
  try {
    await FB.salvarEmprestimo(dados);
    // Decrementar disponibilidade
    for (const id of figurinosIds) {
      const f = state.figurinos.find(x => x.id === id);
      if (f) {
        const disp = (f.quantidadeDisponivel ?? f.quantidade ?? 1) - 1;
        await FB.atualizarFigurino(id, { quantidadeDisponivel: Math.max(0, disp) });
      }
    }
    toast(`Empréstimo para ${alunoNome} registrado!`);
    adicionarHistorico('emprestimo', `Empréstimo registrado para "${alunoNome}" (${itens.length} peça(s))`);
    navigate('emprestimos');
  } catch(err) { toast('Erro: '+err.message, 'error'); }
}

async function registrarDevolucao(id, alunoNome) {
  const e = state.emprestimos.find(x => x.id === id);
  if (!e) return;
  const FB = window.Firebase;
  try {
    await FB.atualizarEmprestimo(id, { status: 'devolvido', dataDevolucaoReal: new Date().toISOString() });
    // Incrementar disponibilidade
    for (const item of (e.itens||[])) {
      const f = state.figurinos.find(x => x.id === item.figurinoId);
      if (f) {
        const disp = (f.quantidadeDisponivel ?? 0) + (item.quantidade || 1);
        await FB.atualizarFigurino(item.figurinoId, { quantidadeDisponivel: Math.min(f.quantidade||1, disp) });
      }
    }
    toast(`Devolução de "${alunoNome}" registrada!`);
    adicionarHistorico('devolucao', `Devolução registrada de "${alunoNome}"`);
    render();
  } catch(err) { toast('Erro: '+err.message, 'error'); }
}

// ── DEVOLUÇÕES ────────────────────────────────────────────────
function renderDevolucoes() {
  const atrasados = state.emprestimos.filter(e => e.status === 'atrasado');
  const ativos = state.emprestimos.filter(e => e.status === 'ativo');
  const devolvidos = state.emprestimos.filter(e => e.status === 'devolvido').slice(0, 10);

  function rowDev(e, highlight) {
    const dias = diasAte(e.dataDevolucao);
    const info = highlight
      ? `<span style="color:var(--red);font-size:12px">⚠️ ${Math.abs(dias||0)} dia(s) atrasado</span>`
      : (dias !== null ? `<span style="color:var(--text2);font-size:12px">${dias} dia(s) restante(s)</span>` : '');
    return `<tr style="${highlight?'background:rgba(239,68,68,.05)':''}">
      <td><strong>${e.alunoNome||'—'}</strong></td>
      <td>${(e.itens||[]).map(i=>`<span style="font-size:12px;background:var(--surface2);padding:2px 6px;border-radius:4px;margin-right:4px">${i.nome||i.figurinoId}</span>`).join('')||'—'}</td>
      <td>${formatDate(e.dataDevolucao)}<br>${info}</td>
      <td>${statusBadge(e.status)}</td>
      <td style="text-align:right"><button class="btn btn-primary btn-sm" onclick="registrarDevolucao('${e.id}','${(e.alunoNome||'').replace(/'/g,"\\'")}')">↩️ Devolver</button></td>
    </tr>`;
  }

  return `
  <div>
    ${atrasados.length > 0 ? `
    <div class="card" style="border-color:rgba(239,68,68,.4);margin-bottom:20px">
      <h3 style="font-family:var(--font-head);color:var(--red);margin-bottom:16px">⚠️ Atrasados (${atrasados.length})</h3>
      <div class="table-wrap"><table>
        <thead><tr><th>Aluno</th><th>Figurinos</th><th>Prazo</th><th>Status</th><th></th></tr></thead>
        <tbody>${atrasados.map(e => rowDev(e, true)).join('')}</tbody>
      </table></div>
    </div>` : ''}

    ${ativos.length > 0 ? `
    <div class="card" style="margin-bottom:20px">
      <h3 style="font-family:var(--font-head);margin-bottom:16px">🕐 Em Andamento (${ativos.length})</h3>
      <div class="table-wrap"><table>
        <thead><tr><th>Aluno</th><th>Figurinos</th><th>Prazo</th><th>Status</th><th></th></tr></thead>
        <tbody>${ativos.map(e => rowDev(e, false)).join('')}</tbody>
      </table></div>
    </div>` : ''}

    ${devolvidos.length > 0 ? `
    <div class="card">
      <h3 style="font-family:var(--font-head);margin-bottom:16px">✅ Devolvidos Recentes</h3>
      <div class="table-wrap"><table>
        <thead><tr><th>Aluno</th><th>Figurinos</th><th>Data Devolução</th><th>Status</th><th></th></tr></thead>
        <tbody>${devolvidos.map(e => `<tr>
          <td>${e.alunoNome||'—'}</td>
          <td>${(e.itens||[]).map(i=>`<span style="font-size:12px;background:var(--surface2);padding:2px 6px;border-radius:4px;margin-right:4px">${i.nome||i.figurinoId}</span>`).join('')}</td>
          <td>${formatDate(e.dataDevolucaoReal||e.dataDevolucao)}</td>
          <td>${statusBadge(e.status)}</td>
          <td></td>
        </tr>`).join('')}</tbody>
      </table></div>
    </div>` : ''}

    ${atrasados.length===0 && ativos.length===0 && devolvidos.length===0 ? `
      <div class="empty"><div class="empty-icon">✅</div><p class="empty-text">Nenhum empréstimo registrado</p></div>` : ''}
  </div>`;
}

// ── HISTÓRICO DETALHADO ───────────────────────────────────────
function renderHistorico() {
  const ICONS = {
    cadastro_figurino:'👗', edicao_figurino:'✏️', remocao_figurino:'🗑️',
    cadastro_aluno:'👤', edicao_aluno:'✏️', remocao_aluno:'🗑️',
    emprestimo:'🤝', devolucao:'↩️',
  };
  const TIPOS_LABEL = {
    cadastro_figurino:'Cadastro de Figurino', edicao_figurino:'Edição de Figurino',
    remocao_figurino:'Remoção de Figurino', cadastro_aluno:'Cadastro de Aluno',
    edicao_aluno:'Edição de Aluno', remocao_aluno:'Remoção de Aluno',
    emprestimo:'Empréstimo Registrado', devolucao:'Devolução Registrada',
  };

  if (!state.histFiltros) state.histFiltros = { busca:'', tipo:'', dataIni:'', dataFim:'' };
  const hf = state.histFiltros;

  let lista = state.historico;
  if (hf.busca) lista = lista.filter(h => h.descricao.toLowerCase().includes(hf.busca.toLowerCase()));
  if (hf.tipo) lista = lista.filter(h => h.tipo === hf.tipo);
  if (hf.dataIni) lista = lista.filter(h => new Date(h.timestamp) >= new Date(hf.dataIni));
  if (hf.dataFim) lista = lista.filter(h => new Date(h.timestamp) <= new Date(hf.dataFim + 'T23:59:59'));

  const contagens = {};
  state.historico.forEach(h => { contagens[h.tipo] = (contagens[h.tipo]||0)+1; });

  return `
  <div>
    <!-- Filtros -->
    <div class="card" style="margin-bottom:20px">
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:12px">
        <div class="search-wrap" style="grid-column:span 2">
          <span class="search-icon">🔍</span>
          <input class="form-input" placeholder="Buscar no histórico..." value="${hf.busca}"
            oninput="state.histFiltros.busca=this.value;render()">
        </div>
        <div>
          <label class="form-label">Tipo de ação</label>
          <select class="form-select" onchange="state.histFiltros.tipo=this.value;render()">
            <option value="">Todos os tipos</option>
            ${Object.entries(TIPOS_LABEL).map(([k,v])=>`<option value="${k}" ${hf.tipo===k?'selected':''}>${v} ${contagens[k]?'('+contagens[k]+')':''}</option>`).join('')}
          </select>
        </div>
        <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px">
          <div>
            <label class="form-label">De</label>
            <input class="form-input" type="date" value="${hf.dataIni}" onchange="state.histFiltros.dataIni=this.value;render()">
          </div>
          <div>
            <label class="form-label">Até</label>
            <input class="form-input" type="date" value="${hf.dataFim}" onchange="state.histFiltros.dataFim=this.value;render()">
          </div>
        </div>
      </div>
      <div style="display:flex;align-items:center;justify-content:space-between">
        <span style="font-size:13px;color:var(--text2)">${lista.length} registro(s) encontrado(s)</span>
        <div style="display:flex;gap:8px">
          ${(hf.busca||hf.tipo||hf.dataIni||hf.dataFim)?`<button class="btn btn-secondary btn-sm" onclick="state.histFiltros={busca:'',tipo:'',dataIni:'',dataFim:''};render()">✕ Limpar filtros</button>`:''}
          <button class="btn btn-secondary btn-sm" onclick="exportarHistorico()">⬇️ Exportar CSV</button>
        </div>
      </div>
    </div>

    <!-- Resumo por tipo -->
    <div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:10px;margin-bottom:20px">
      ${Object.entries(TIPOS_LABEL).filter(([k])=>contagens[k]).map(([k,v])=>`
        <div class="stat-card" style="padding:14px;cursor:pointer" onclick="state.histFiltros.tipo='${k}';render()">
          <div style="font-size:20px">${ICONS[k]||'📌'}</div>
          <div><div style="font-family:var(--font-head);font-size:20px;font-weight:800">${contagens[k]||0}</div>
          <div style="font-size:11px;color:var(--text2)">${v}</div></div>
        </div>`).join('')}
    </div>

    <!-- Tabela -->
    ${lista.length === 0 ? `
      <div class="empty"><div class="empty-icon">📋</div><p class="empty-text">Nenhum registro encontrado</p></div>` : `
    <div class="card" style="padding:0">
      <div class="table-wrap"><table>
        <thead><tr>
          <th style="width:40px"></th>
          <th>Tipo</th>
          <th>Descrição</th>
          <th>Data</th>
          <th>Hora</th>
        </tr></thead>
        <tbody>
          ${lista.map(h => {
            const dt = new Date(h.timestamp);
            return `<tr>
              <td style="text-align:center;font-size:18px">${ICONS[h.tipo]||'📌'}</td>
              <td><span class="badge ${tipoBadgeClass(h.tipo)}">${TIPOS_LABEL[h.tipo]||h.tipo}</span></td>
              <td style="max-width:300px">${h.descricao}</td>
              <td style="color:var(--text2);font-size:13px;white-space:nowrap">${dt.toLocaleDateString('pt-BR')}</td>
              <td style="color:var(--text2);font-size:13px">${dt.toLocaleTimeString('pt-BR',{hour:'2-digit',minute:'2-digit'})}</td>
            </tr>`;
          }).join('')}
        </tbody>
      </table></div>
    </div>`}
  </div>`;
}

function tipoBadgeClass(tipo) {
  if (tipo.startsWith('cadastro')) return 'badge-green';
  if (tipo.startsWith('edicao')) return 'badge-blue';
  if (tipo.startsWith('remocao')) return 'badge-red';
  if (tipo === 'emprestimo') return 'badge-yellow';
  if (tipo === 'devolucao') return 'badge-purple';
  return 'badge-gray';
}

function exportarHistorico() {
  const linhas = [['Tipo','Descrição','Data','Hora']];
  state.historico.forEach(h => {
    const dt = new Date(h.timestamp);
    linhas.push([h.tipo, h.descricao, dt.toLocaleDateString('pt-BR'), dt.toLocaleTimeString('pt-BR')]);
  });
  const csv = linhas.map(l => l.map(c => `"${c}"`).join(',')).join('\n');
  const blob = new Blob(['\uFEFF'+csv], {type:'text/csv;charset=utf-8'});
  const a = document.createElement('a'); a.href=URL.createObjectURL(blob);
  a.download=`historico-${new Date().toISOString().split('T')[0]}.csv`; a.click();
  toast('Histórico exportado!');
}

// ── RELATÓRIOS SEMESTRAIS ────────────────────────────────────
function renderRelatorios() {
  if (!state.relSemestre) state.relSemestre = getSemestreAtual();
  const sem = state.relSemestre;

  const SEMESTRES = [
    { id:'2026-1', label:'1º Semestre 2026 (Maio–Outubro)', ini: new Date('2026-05-01'), fim: new Date('2026-10-31T23:59:59') },
    { id:'2026-2', label:'2º Semestre 2026 (Novembro–Abril 2027)', ini: new Date('2026-11-01'), fim: new Date('2027-04-30T23:59:59') },
    { id:'2025-2', label:'2º Semestre 2025 (Nov 2025–Abr 2026)', ini: new Date('2025-11-01'), fim: new Date('2026-04-30T23:59:59') },
  ];

  const semObj = SEMESTRES.find(s=>s.id===sem) || SEMESTRES[0];
  const { ini, fim } = semObj;

  const empSem = state.emprestimos.filter(e => {
    const d = e.dataRetirada ? new Date(e.dataRetirada) : (e.criadoEm?.toDate ? e.criadoEm.toDate() : new Date(e.criadoEm||0));
    return d >= ini && d <= fim;
  });
  const devSem = empSem.filter(e => e.status === 'devolvido');
  const atrasadosSem = empSem.filter(e => e.status === 'atrasado');
  const ativosSem = empSem.filter(e => e.status === 'ativo');

  // Top figurinos mais emprestados
  const contFig = {};
  empSem.forEach(e => (e.itens||[]).forEach(i => { contFig[i.nome||i.figurinoId] = (contFig[i.nome||i.figurinoId]||0)+1; }));
  const topFig = Object.entries(contFig).sort((a,b)=>b[1]-a[1]).slice(0,5);

  // Top alunos
  const contAluno = {};
  empSem.forEach(e => { contAluno[e.alunoNome||'?'] = (contAluno[e.alunoNome||'?']||0)+1; });
  const topAlunos = Object.entries(contAluno).sort((a,b)=>b[1]-a[1]).slice(0,5);

  // Empréstimos por mês
  const porMes = {};
  empSem.forEach(e => {
    const d = e.dataRetirada ? new Date(e.dataRetirada) : new Date(e.criadoEm||0);
    const k = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}`;
    porMes[k] = (porMes[k]||0)+1;
  });
  const meses = Object.entries(porMes).sort();

  // Taxa de devolução no prazo
  const taxaDevolucao = empSem.length > 0 ? Math.round((devSem.length/empSem.length)*100) : 0;

  return `
  <div style="max-width:1000px">
    <!-- Seletor de semestre -->
    <div class="card" style="margin-bottom:24px">
      <div style="display:flex;align-items:center;gap:16px;flex-wrap:wrap">
        <strong style="font-family:var(--font-head);font-size:15px">Período:</strong>
        <div style="display:flex;gap:8px;flex-wrap:wrap">
          ${SEMESTRES.map(s=>`<button class="btn ${sem===s.id?'btn-primary':'btn-secondary'} btn-sm" onclick="state.relSemestre='${s.id}';render()">${s.label}</button>`).join('')}
        </div>
        <button class="btn btn-secondary btn-sm" onclick="exportarRelatorio('${sem}')" style="margin-left:auto">⬇️ Exportar PDF</button>
      </div>
    </div>

    <!-- Título do relatório -->
    <div style="margin-bottom:24px">
      <h2 style="font-family:var(--font-head);font-size:22px;font-weight:800">${semObj.label}</h2>
      <p style="color:var(--text2);font-size:14px">Relatório gerado em ${new Date().toLocaleDateString('pt-BR')} às ${new Date().toLocaleTimeString('pt-BR',{hour:'2-digit',minute:'2-digit'})}</p>
    </div>

    <!-- KPIs -->
    <div class="card-grid" style="grid-template-columns:repeat(auto-fill,minmax(180px,1fr));margin-bottom:24px">
      ${statCard('🤝','Empréstimos no Período', empSem.length, 'rgba(59,130,246,.2)', 'emprestimos')}
      ${statCard('✅','Devolvidos', devSem.length, 'rgba(34,197,94,.2)', 'devolucoes')}
      ${statCard('⚠️','Atrasados', atrasadosSem.length, 'rgba(239,68,68,.2)', 'devolucoes')}
      ${statCard('🕐','Em Andamento', ativosSem.length, 'rgba(240,165,0,.2)', 'emprestimos')}
      ${statCard('📊','Taxa de Devolução', taxaDevolucao+'%', 'rgba(168,85,247,.2)', 'relatorios')}
    </div>

    <div style="display:grid;grid-template-columns:1fr 1fr;gap:20px;margin-bottom:24px">
      <!-- Top figurinos -->
      <div class="card">
        <h3 style="font-family:var(--font-head);margin-bottom:16px">👗 Figurinos Mais Emprestados</h3>
        ${topFig.length===0 ? `<p style="color:var(--text2);font-size:14px">Sem dados no período</p>` : `
        <div style="display:flex;flex-direction:column;gap:10px">
          ${topFig.map(([nome,qtd],i)=>`
            <div style="display:flex;align-items:center;gap:10px">
              <div style="width:24px;height:24px;background:${i===0?'var(--accent)':i===1?'var(--text2)':'var(--border)'};border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:11px;font-weight:700;color:${i===0?'#0c0c10':'var(--text)'};">${i+1}</div>
              <div style="flex:1;font-size:14px">${nome}</div>
              <div style="display:flex;align-items:center;gap:8px">
                <div style="height:6px;background:var(--accent);border-radius:3px;width:${Math.round((qtd/topFig[0][1])*80)}px"></div>
                <span style="font-family:var(--font-head);font-weight:700">${qtd}</span>
              </div>
            </div>`).join('')}
        </div>`}
      </div>

      <!-- Top alunos -->
      <div class="card">
        <h3 style="font-family:var(--font-head);margin-bottom:16px">👥 Alunos Mais Ativos</h3>
        ${topAlunos.length===0 ? `<p style="color:var(--text2);font-size:14px">Sem dados no período</p>` : `
        <div style="display:flex;flex-direction:column;gap:10px">
          ${topAlunos.map(([nome,qtd],i)=>`
            <div style="display:flex;align-items:center;gap:10px">
              <div style="width:24px;height:24px;background:${i===0?'var(--blue)':'var(--border)'};border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:11px;font-weight:700;color:${i===0?'#fff':'var(--text)'};">${i+1}</div>
              <div style="flex:1;font-size:14px">${nome}</div>
              <div style="display:flex;align-items:center;gap:8px">
                <div style="height:6px;background:var(--blue);border-radius:3px;width:${Math.round((qtd/topAlunos[0][1])*80)}px"></div>
                <span style="font-family:var(--font-head);font-weight:700">${qtd}</span>
              </div>
            </div>`).join('')}
        </div>`}
      </div>
    </div>

    <!-- Empréstimos por mês -->
    <div class="card" style="margin-bottom:24px">
      <h3 style="font-family:var(--font-head);margin-bottom:20px">📅 Empréstimos por Mês</h3>
      ${meses.length===0 ? `<p style="color:var(--text2);font-size:14px">Sem dados no período</p>` : `
      <div style="display:flex;align-items:flex-end;gap:12px;height:120px;padding-bottom:8px">
        ${meses.map(([mes,qtd])=>{
          const max = Math.max(...meses.map(m=>m[1]));
          const h = Math.max(12, Math.round((qtd/max)*100));
          const [ano,m] = mes.split('-');
          const nomes = ['Jan','Fev','Mar','Abr','Mai','Jun','Jul','Ago','Set','Out','Nov','Dez'];
          return `<div style="flex:1;display:flex;flex-direction:column;align-items:center;gap:6px">
            <span style="font-family:var(--font-head);font-size:12px;font-weight:700">${qtd}</span>
            <div style="width:100%;background:var(--accent);border-radius:4px 4px 0 0;height:${h}px;opacity:.85"></div>
            <span style="font-size:11px;color:var(--text2)">${nomes[parseInt(m)-1]}</span>
          </div>`;
        }).join('')}
      </div>`}
    </div>

    <!-- Lista completa de empréstimos do semestre -->
    <div class="card" style="padding:0">
      <div style="padding:16px 20px;border-bottom:1px solid var(--border)">
        <h3 style="font-family:var(--font-head)">📋 Todos os Empréstimos do Período (${empSem.length})</h3>
      </div>
      ${empSem.length===0 ? `<div class="empty" style="padding:40px"><p class="empty-text">Nenhum empréstimo neste período</p></div>` : `
      <div class="table-wrap"><table>
        <thead><tr><th>Aluno</th><th>Figurinos</th><th>Retirada</th><th>Devolução</th><th>Status</th></tr></thead>
        <tbody>
          ${empSem.map(e=>`<tr>
            <td><strong>${e.alunoNome||'—'}</strong></td>
            <td>${(e.itens||[]).map(i=>`<span style="font-size:12px;background:var(--surface2);padding:2px 6px;border-radius:4px;margin-right:4px">${i.nome||i.figurinoId}</span>`).join('')||'—'}</td>
            <td style="font-size:13px">${formatDate(e.dataRetirada)}</td>
            <td style="font-size:13px">${formatDate(e.dataDevolucao)}</td>
            <td>${statusBadge(e.status)}</td>
          </tr>`).join('')}
        </tbody>
      </table></div>`}
    </div>
  </div>`;
}

function getSemestreAtual() {
  const m = new Date().getMonth()+1;
  return (m >= 5 && m <= 10) ? '2026-1' : '2026-2';
}

function exportarRelatorio(semId) {
  const semLabels = {'2026-1':'1S2026_Mai-Out','2026-2':'2S2026_Nov-Abr','2025-2':'2S2025'};
  const label = semLabels[semId]||semId;
  const ini = semId==='2026-1' ? new Date('2026-05-01') : semId==='2026-2' ? new Date('2026-11-01') : new Date('2025-11-01');
  const fim = semId==='2026-1' ? new Date('2026-10-31') : semId==='2026-2' ? new Date('2027-04-30') : new Date('2026-04-30');
  const empSem = state.emprestimos.filter(e => {
    const d = e.dataRetirada ? new Date(e.dataRetirada) : new Date(e.criadoEm||0);
    return d >= ini && d <= fim;
  });
  const linhas = [['Aluno','Figurinos','Data Retirada','Data Devolução','Status']];
  empSem.forEach(e => linhas.push([
    e.alunoNome||'—',
    (e.itens||[]).map(i=>i.nome||i.figurinoId).join('; '),
    formatDate(e.dataRetirada), formatDate(e.dataDevolucao), e.status
  ]));
  const csv = linhas.map(l=>l.map(c=>`"${c}"`).join(',')).join('\n');
  const blob = new Blob(['\uFEFF'+csv],{type:'text/csv;charset=utf-8'});
  const a = document.createElement('a'); a.href=URL.createObjectURL(blob);
  a.download=`relatorio-${label}.csv`; a.click();
  toast('Relatório exportado!');
}

// ── ANÁLISES ─────────────────────────────────────────────────
function renderAnalises() {
  const totalFig = state.figurinos.length;
  const totalAlunos = state.alunos.length;
  const totalEmp = state.emprestimos.length;
  const totalDev = state.emprestimos.filter(e=>e.status==='devolvido').length;
  const totalAtrasados = state.emprestimos.filter(e=>e.status==='atrasado').length;
  const totalAtivos = state.emprestimos.filter(e=>e.status==='ativo').length;
  const taxaGeral = totalEmp > 0 ? Math.round((totalDev/totalEmp)*100) : 0;

  // Distribuição por categoria
  const porCat = {};
  state.figurinos.forEach(f => { const c = f.categoria||'outro'; porCat[c]=(porCat[c]||0)+1; });

  // Distribuição por estado de conservação
  const porEstado = {};
  state.figurinos.forEach(f => { const e = f.estado||'bom'; porEstado[e]=(porEstado[e]||0)+1; });

  // Figurinos sem empréstimos (nunca usados)
  const usados = new Set(state.emprestimos.flatMap(e=>(e.itens||[]).map(i=>i.figurinoId)));
  const semUso = state.figurinos.filter(f=>!usados.has(f.id));

  // Alunos com empréstimos atrasados
  const alunosAtrasados = [...new Set(state.emprestimos.filter(e=>e.status==='atrasado').map(e=>e.alunoNome))];

  // Média de dias de empréstimo
  const duracoes = state.emprestimos.filter(e=>e.dataRetirada&&e.dataDevolucao).map(e=>{
    return Math.ceil((new Date(e.dataDevolucao)-new Date(e.dataRetirada))/86400000);
  });
  const mediaDias = duracoes.length > 0 ? Math.round(duracoes.reduce((a,b)=>a+b,0)/duracoes.length) : 0;

  // Ocupação do acervo
  const totalPecas = state.figurinos.reduce((s,f)=>s+(f.quantidade||1),0);
  const totalDisp = state.figurinos.reduce((s,f)=>s+(f.quantidadeDisponivel??f.quantidade??1),0);
  const taxaOcup = totalPecas > 0 ? Math.round(((totalPecas-totalDisp)/totalPecas)*100) : 0;

  function barChart(items, cor) {
    if (!items.length) return `<p style="color:var(--text2);font-size:14px">Sem dados</p>`;
    const max = Math.max(...items.map(i=>i[1]));
    return items.map(([label,val]) => `
      <div style="display:flex;align-items:center;gap:10px;margin-bottom:10px">
        <div style="width:110px;font-size:13px;color:var(--text2);text-align:right;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">${label}</div>
        <div style="flex:1;height:22px;background:var(--surface2);border-radius:4px;overflow:hidden">
          <div style="height:100%;background:${cor};width:${Math.round((val/max)*100)}%;border-radius:4px;transition:width .5s;display:flex;align-items:center;padding-left:8px">
            <span style="font-size:12px;font-weight:700;color:#fff">${val}</span>
          </div>
        </div>
      </div>`).join('');
  }

  const catItems = Object.entries(porCat).map(([k,v])=>[CATEGORIAS_LABEL[k]||k,v]).sort((a,b)=>b[1]-a[1]);
  const estItems = Object.entries(porEstado).map(([k,v])=>[ESTADOS_LABEL[k]||k,v]).sort((a,b)=>b[1]-a[1]);

  return `
  <div style="max-width:1000px">
    <!-- Indicadores gerais -->
    <h3 style="font-family:var(--font-head);margin-bottom:16px">📊 Indicadores Gerais</h3>
    <div class="card-grid" style="grid-template-columns:repeat(auto-fill,minmax(180px,1fr));margin-bottom:28px">
      <div class="stat-card"><div class="stat-icon" style="background:rgba(59,130,246,.2)">📦</div><div><div class="stat-value">${totalFig}</div><div class="stat-label">Figurinos Cadastrados</div></div></div>
      <div class="stat-card"><div class="stat-icon" style="background:rgba(168,85,247,.2)">👥</div><div><div class="stat-value">${totalAlunos}</div><div class="stat-label">Alunos Cadastrados</div></div></div>
      <div class="stat-card"><div class="stat-icon" style="background:rgba(240,165,0,.2)">🤝</div><div><div class="stat-value">${totalEmp}</div><div class="stat-label">Total de Empréstimos</div></div></div>
      <div class="stat-card"><div class="stat-icon" style="background:rgba(34,197,94,.2)">%</div><div><div class="stat-value">${taxaGeral}%</div><div class="stat-label">Taxa de Devolução</div></div></div>
      <div class="stat-card"><div class="stat-icon" style="background:rgba(59,130,246,.2)">📅</div><div><div class="stat-value">${mediaDias}d</div><div class="stat-label">Média de Empréstimo</div></div></div>
      <div class="stat-card"><div class="stat-icon" style="background:rgba(255,107,53,.2)">🔄</div><div><div class="stat-value">${taxaOcup}%</div><div class="stat-label">Taxa de Ocupação</div></div></div>
    </div>

    <div style="display:grid;grid-template-columns:1fr 1fr;gap:20px;margin-bottom:24px">
      <!-- Distribuição por categoria -->
      <div class="card">
        <h3 style="font-family:var(--font-head);margin-bottom:18px">👗 Acervo por Categoria</h3>
        ${barChart(catItems,'var(--accent)')}
      </div>
      <!-- Estado de conservação -->
      <div class="card">
        <h3 style="font-family:var(--font-head);margin-bottom:18px">🔍 Estado de Conservação</h3>
        ${barChart(estItems,'var(--blue)')}
      </div>
    </div>

    <!-- Status dos empréstimos -->
    <div class="card" style="margin-bottom:24px">
      <h3 style="font-family:var(--font-head);margin-bottom:20px">🤝 Status dos Empréstimos</h3>
      <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:16px">
        ${[
          ['✅ Devolvidos', totalDev, 'var(--green)', totalEmp],
          ['🕐 Ativos', totalAtivos, 'var(--blue)', totalEmp],
          ['⚠️ Atrasados', totalAtrasados, 'var(--red)', totalEmp],
        ].map(([label,val,cor,tot]) => `
          <div style="text-align:center;padding:20px;background:var(--surface2);border-radius:var(--radius)">
            <div style="font-family:var(--font-head);font-size:32px;font-weight:800;color:${cor}">${val}</div>
            <div style="font-size:13px;color:var(--text2);margin-top:4px">${label}</div>
            <div style="margin-top:12px;height:4px;background:var(--border);border-radius:2px">
              <div style="height:100%;background:${cor};width:${tot>0?Math.round((val/tot)*100):0}%;border-radius:2px"></div>
            </div>
            <div style="font-size:11px;color:var(--text2);margin-top:4px">${tot>0?Math.round((val/tot)*100):0}% do total</div>
          </div>`).join('')}
      </div>
    </div>

    <div style="display:grid;grid-template-columns:1fr 1fr;gap:20px;margin-bottom:24px">
      <!-- Alunos com atrasos -->
      <div class="card" style="border-color:${alunosAtrasados.length?'rgba(239,68,68,.4)':'var(--border)'}">
        <h3 style="font-family:var(--font-head);margin-bottom:16px;color:${alunosAtrasados.length?'var(--red)':'var(--text)'}">
          ⚠️ Alunos com Atraso (${alunosAtrasados.length})
        </h3>
        ${alunosAtrasados.length===0 ? `<p style="color:var(--green);font-size:14px">✅ Nenhum aluno com atraso!</p>` : `
        <div style="display:flex;flex-direction:column;gap:8px">
          ${alunosAtrasados.map(n=>`
            <div style="display:flex;align-items:center;gap:10px;padding:8px 12px;background:rgba(239,68,68,.08);border-radius:8px;border:1px solid rgba(239,68,68,.2)">
              <span style="font-size:16px">👤</span>
              <span style="font-size:14px">${n}</span>
            </div>`).join('')}
        </div>`}
      </div>

      <!-- Figurinos sem uso -->
      <div class="card">
        <h3 style="font-family:var(--font-head);margin-bottom:16px">😴 Figurinos Sem Uso (${semUso.length})</h3>
        ${semUso.length===0 ? `<p style="color:var(--green);font-size:14px">✅ Todos os figurinos já foram emprestados!</p>` : `
        <div style="display:flex;flex-direction:column;gap:6px;max-height:200px;overflow-y:auto">
          ${semUso.map(f=>`
            <div style="display:flex;align-items:center;gap:10px;padding:6px 10px;background:var(--surface2);border-radius:6px">
              <span style="font-size:14px">👗</span>
              <span style="font-size:13px">${f.nome}</span>
              <span class="badge badge-gray" style="margin-left:auto">${CATEGORIAS_LABEL[f.categoria]||f.categoria||'—'}</span>
            </div>`).join('')}
        </div>`}
      </div>
    </div>

    <!-- Insights automáticos -->
    <div class="card" style="border-color:rgba(240,165,0,.3)">
      <h3 style="font-family:var(--font-head);margin-bottom:16px">💡 Insights</h3>
      <div style="display:flex;flex-direction:column;gap:10px">
        ${gerarInsights(taxaGeral, taxaOcup, atrasadosSem=totalAtrasados, semUso, mediaDias, totalEmp).map(ins=>`
          <div style="display:flex;gap:10px;padding:12px;background:var(--surface2);border-radius:8px;font-size:14px">
            <span style="font-size:18px">${ins.icon}</span>
            <div>
              <strong>${ins.titulo}</strong><br>
              <span style="color:var(--text2)">${ins.texto}</span>
            </div>
          </div>`).join('')}
      </div>
    </div>
  </div>`;
}

function gerarInsights(taxaDev, taxaOcup, atrasados, semUso, mediaDias, totalEmp) {
  const ins = [];
  if (taxaDev >= 90) ins.push({icon:'🏆', titulo:'Excelente taxa de devolução!', texto:`${taxaDev}% dos empréstimos foram devolvidos. O acervo está sendo bem gerenciado.`});
  else if (taxaDev >= 70) ins.push({icon:'✅', titulo:'Boa taxa de devolução', texto:`${taxaDev}% de devoluções. Mantenha o acompanhamento dos prazos.`});
  else if (taxaDev > 0) ins.push({icon:'⚠️', titulo:'Taxa de devolução abaixo do ideal', texto:`Apenas ${taxaDev}% foram devolvidos. Considere reforçar a comunicação com os alunos.`});

  if (taxaOcup >= 80) ins.push({icon:'🔥', titulo:'Acervo muito ocupado', texto:`${taxaOcup}% das peças estão emprestadas. Considere ampliar o acervo.`});
  else if (taxaOcup >= 50) ins.push({icon:'📦', titulo:'Boa utilização do acervo', texto:`${taxaOcup}% do acervo em uso.`});
  else ins.push({icon:'💤', titulo:'Acervo subutilizado', texto:`Apenas ${taxaOcup}% das peças estão em uso. Divulgue mais o acervo!`});

  if (atrasados > 0) ins.push({icon:'⏰', titulo:`${atrasados} empréstimo(s) em atraso`, texto:'Entre em contato com os alunos para regularizar a devolução.'});

  if (semUso.length > 0) ins.push({icon:'🔎', titulo:`${semUso.length} figurino(s) nunca emprestado(s)`, texto:'Considere divulgar esses itens ou verificar se estão em boas condições.'});

  if (mediaDias > 0) ins.push({icon:'📅', titulo:`Média de ${mediaDias} dias por empréstimo`, texto: mediaDias > 21 ? 'Os empréstimos estão demorando muito. Revise os prazos.' : 'Prazo médio saudável para os empréstimos.'});

  if (totalEmp === 0) ins.push({icon:'🚀', titulo:'Comece a registrar empréstimos!', texto:'Nenhum empréstimo registrado ainda. Use o sistema para controlar o acervo.'});

  return ins;
}

// ── CONFIGURAÇÕES ─────────────────────────────────────────────
function renderConfig() {
  const configured = window.Firebase?.isConfigured();
  return `
  <div style="max-width:700px">
    <div class="card" style="margin-bottom:20px;border-color:${configured?'rgba(34,197,94,.4)':'rgba(240,165,0,.4)'}">
      <div style="display:flex;align-items:center;gap:12px;margin-bottom:12px">
        <span style="font-size:24px">${configured?'✅':'⚠️'}</span>
        <div>
          <strong style="font-family:var(--font-head)">${configured?'Firebase Conectado':'Firebase Não Configurado'}</strong>
          <div style="font-size:14px;color:var(--text2);">${configured?'Dados sendo salvos na nuvem em tempo real':'Dados salvos apenas localmente (perdem com limpeza do navegador)'}</div>
        </div>
      </div>
    </div>

    <div class="card" style="margin-bottom:20px">
      <h3 style="font-family:var(--font-head);margin-bottom:20px">🚀 Como configurar Firebase (gratuito)</h3>

      <div class="step">
        <div class="step-num">1</div>
        <div>Acesse <a href="https://console.firebase.google.com" target="_blank" style="color:var(--accent)">console.firebase.google.com</a> e crie um projeto (conta Google necessária)</div>
      </div>
      <div class="step">
        <div class="step-num">2</div>
        <div>No projeto, vá em <strong>Project Settings (⚙️) → Adicionar app → Web</strong>. Copie as configurações <code style="background:var(--surface2);padding:2px 6px;border-radius:4px">firebaseConfig</code></div>
      </div>
      <div class="step">
        <div class="step-num">3</div>
        <div>Vá em <strong>Firestore Database → Criar banco de dados → Modo de teste</strong></div>
      </div>
      <div class="step">
        <div class="step-num">4</div>
        <div>Vá em <strong>Storage → Começar → Modo de teste</strong></div>
      </div>
      <div class="step">
        <div class="step-num">5</div>
        <div>Neste arquivo HTML, localize o bloco <code style="background:var(--surface2);padding:2px 6px;border-radius:4px">const firebaseConfig = {</code> e substitua com suas configurações</div>
      </div>

      <div class="code-block">// Exemplo de como ficará:
const firebaseConfig = {
  apiKey: "AIzaSyAbc123...",
  authDomain: "meu-acervo.firebaseapp.com",
  projectId: "meu-acervo",
  storageBucket: "meu-acervo.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123..."
};</div>

      <div class="step" style="margin-top:16px">
        <div class="step-num">6</div>
        <div>Hospede o arquivo em qualquer serviço gratuito:
          <ul style="margin-top:8px;padding-left:20px;color:var(--text2);font-size:14px;line-height:1.8">
            <li><a href="https://pages.github.com" target="_blank" style="color:var(--accent)">GitHub Pages</a> — gratuito, apenas faça upload do arquivo</li>
            <li><a href="https://firebase.google.com/docs/hosting" target="_blank" style="color:var(--accent)">Firebase Hosting</a> — integrado ao projeto Firebase</li>
            <li><a href="https://netlify.com" target="_blank" style="color:var(--accent)">Netlify</a> — arraste e solte o arquivo HTML</li>
          </ul>
        </div>
      </div>
    </div>

    <div class="card" style="margin-bottom:20px">
      <h3 style="font-family:var(--font-head);margin-bottom:16px">📱 Acesso em Qualquer Dispositivo</h3>
      <p style="color:var(--text2);font-size:14px;line-height:1.7">
        Uma vez hospedado, qualquer pessoa com o link pode acessar o sistema — celular, tablet, computador.
        Os dados são sincronizados em <strong style="color:var(--text)">tempo real</strong>: se alguém cadastrar um figurino em um celular, aparece imediatamente em todos os outros dispositivos.
      </p>
    </div>

    <div class="card">
      <h3 style="font-family:var(--font-head);margin-bottom:16px">💾 Exportar Dados (Backup)</h3>
      <p style="color:var(--text2);font-size:14px;margin-bottom:16px">Baixe todos os dados como arquivo JSON:</p>
      <button class="btn btn-secondary" onclick="exportarDados()">⬇️ Exportar Tudo como JSON</button>
    </div>
  </div>`;
}

function exportarDados() {
  const dados = {
    figurinos: state.figurinos,
    alunos: state.alunos,
    emprestimos: state.emprestimos,
    exportadoEm: new Date().toISOString(),
  };
  const blob = new Blob([JSON.stringify(dados, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = `acervo-figurinos-${new Date().toISOString().split('T')[0]}.json`;
  a.click(); URL.revokeObjectURL(url);
  toast('Dados exportados!');
}

// ── HISTÓRICO LOCAL ───────────────────────────────────────────
function adicionarHistorico(tipo, descricao) {
  state.historico.unshift({ tipo, descricao, timestamp: new Date().toISOString() });
  if (state.historico.length > 200) state.historico.pop();
}

// ── INIT ──────────────────────────────────────────────────────
render();

// Responsive: mostrar botão menu em mobile
if (window.innerWidth < 768) {
  document.getElementById('menu-btn').style.display = 'flex';
  document.getElementById('sidebar').classList.add('collapsed');
}
</script>
</body>
</html>
