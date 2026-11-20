<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>我的百科</title>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <!-- 引入 Markdown 解析器 -->
  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: -apple-system, "Microsoft JhengHei", sans-serif; background: #f8f9fa; color: #202122; display: flex; height: 100vh; overflow: hidden; }
    .hidden { display: none !important; }

    /* 登入畫面 */
    .auth-view { position: fixed; top: 0; left: 0; width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; background: #f8f9fa; z-index: 1000; }
    .auth-card { background: #fff; border: 1px solid #a2a9b1; border-radius: 4px; padding: 32px; width: 100%; max-width: 400px; display: flex; flex-direction: column; gap: 16px; }
    .auth-card h1 { text-align: center; font-family: "Times New Roman", serif; }
    .auth-card input { padding: 10px; border: 1px solid #a2a9b1; border-radius: 2px; }
    .auth-card button { padding: 10px; background: #3366cc; color: #fff; border: none; cursor: pointer; }

    /* 側邊欄 */
    .sidebar { width: 250px; background: #f8f9fa; border-right: 1px solid #a2a9b1; display: flex; flex-direction: column; }
    .sidebar-header { padding: 15px; border-bottom: 1px solid #a2a9b1; background: #fff; display: flex; justify-content: space-between; align-items: center; }
    .nav-list { flex: 1; overflow-y: auto; padding: 10px; }
    .nav-item { padding: 8px 10px; cursor: pointer; font-size: 0.95rem; color: #3366cc; border-bottom: 1px solid #eaecf0; text-decoration: none; display: block; }
    .nav-item:hover { background: #eaecf0; text-decoration: underline; }

    /* 主內容區 */
    .main-content { flex: 1; background: #fff; overflow-y: auto; padding: 40px; }
    .article { max-width: 900px; margin: 0 auto; }
    
    /* 百科排版樣式 */
    .article h1.title { font-family: "Times New Roman", serif; font-size: 2.5rem; font-weight: normal; border-bottom: 2px solid #a2a9b1; padding-bottom: 10px; margin-bottom: 20px; }
    .article-content { font-size: 1rem; line-height: 1.8; }
    
    /* Markdown 渲染出來的樣式 */
    .article-content h1, .article-content h2, .article-content h3 { font-family: "Times New Roman", serif; border-bottom: 1px solid #a2a9b1; padding-bottom: 5px; margin: 20px 0 10px 0; font-weight: normal; }
    .article-content h2 { font-size: 1.6rem; }
    .article-content h3 { font-size: 1.3rem; border-bottom: none; }
    .article-content ul, .article-content ol { margin-left: 25px; margin-bottom: 15px; }
    .article-content li { margin-bottom: 5px; }
    .article-content a { color: #3366cc; text-decoration: none; }
    .article-content a:hover { text-decoration: underline; }
    .article-content p { margin-bottom: 15px; }

    /* 編輯器 */
    .editor-box { background: #f8f9fa; border: 1px solid #a2a9b1; padding: 20px; margin-top: 20px; border-radius: 2px; }
    .editor-box input, .editor-box select, .editor-box textarea { width: 100%; padding: 10px; margin-bottom: 15px; border: 1px solid #a2a9b1; font-family: monospace; font-size: 1rem; }
    .editor-box textarea { min-height: 400px; line-height: 1.5; resize: vertical; }
    
    .btn { padding: 8px 16px; border: none; cursor: pointer; border-radius: 2px; font-size: 0.95rem; }
    .btn-primary { background: #3366cc; color: #fff; }
    .btn-danger { background: #d33; color: #fff; }
    .btn-outline { background: #fff; color: #333; border: 1px solid #a2a9b1; }
  </style>
</head>
<body>

  <div id="authView" class="auth-view hidden">
    <div class="auth-card">
      <h1>我的百科</h1>
      <input type="email" id="email" placeholder="Email">
      <input type="password" id="password" placeholder="密碼">
      <button id="loginBtn" class="btn-primary">登入</button>
      <button id="signupBtn" class="btn-outline">註冊</button>
      <p id="authMsg" style="color:#d33; font-size:0.85rem; text-align:center;"></p>
    </div>
  </div>

  <div id="mainView" class="hidden" style="display: flex; width: 100%; height: 100%;">
    <div class="sidebar">
      <div class="sidebar-header">
        <h2 style="font-family: serif;">目錄</h2>
        <button id="newEntryBtn" class="btn btn-primary" style="padding: 4px 8px; font-size: 0.8rem;">+ 新增</button>
      </div>
      <div class="nav-list" id="navList"></div>
      <div style="padding: 10px; border-top: 1px solid #a2a9b1;">
        <button id="logoutBtn" class="btn btn-outline" style="width: 100%;">登出</button>
      </div>
    </div>

    <div class="main-content" id="mainContent">
      <div id="welcomeScreen" style="text-align: center; margin-top: 100px;">
        <h1 style="font-family: serif; font-size: 3rem; color: #a2a9b1;">我的百科全書</h1>
        <p style="color: #666; margin-top: 10px;">請從左側目錄選擇條目，或建立新條目。</p>
      </div>

      <!-- 閱讀區 -->
      <div id="viewer" class="article hidden">
        <h1 class="title" id="viewTitle"></h1>
        <div id="viewContent" class="article-content"></div>
        <div style="margin-top: 40px; border-top: 1px solid #eaecf0; padding-top: 20px;">
          <button id="editBtn" class="btn btn-outline">編輯</button>
          <button id="deleteBtn" class="btn btn-danger" style="margin-left: 10px;">刪除</button>
        </div>
      </div>

      <!-- 編輯區 -->
      <div id="editor" class="article hidden">
        <h1 class="title" id="editorTitle">建立新條目</h1>
        <div class="editor-box">
          <input type="text" id="editTitle" placeholder="條目名稱 (例：作品簡介)">
          <select id="editCategory">
            <option value="未分類">未分類</option>
            <option value="世界觀">世界觀</option>
            <option value="角色">角色</option>
            <option value="劇情">劇情</option>
            <option value="設定">設定</option>
          </select>
          <p style="font-size: 0.85rem; color: #666; margin-bottom: 5px;">支援 Markdown 語法 (例：`# 大標題`、`## 小標題`、`- 列表`、`**粗體**`)</p>
          <textarea id="editContent" placeholder="# 第一章&#10;這裡寫內容...&#10;&#10;## 第一節&#10;- 重點一&#10;- 重點二"></textarea>
          <button id="saveBtn" class="btn btn-primary">儲存</button>
          <button id="cancelBtn" class="btn btn-outline" style="margin-left: 10px;">取消</button>
        </div>
      </div>
    </div>
  </div>

  <script>
    const SUPABASE_URL = 'https://vshfxcygviqsxxnanxzd.supabase.co';
    const SUPABASE_ANON_KEY = 'sb_publishable_KMI9aX76T4fxrrwOCoErlA_7ybOkPq0'; 

    const db = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
      auth: { persistSession: true, autoRefreshToken: true }
    });

    const authView = document.getElementById('authView');
    const mainView = document.getElementById('mainView');
    const navList = document.getElementById('navList');
    const viewer = document.getElementById('viewer');
    const editor = document.getElementById('editor');
    const welcomeScreen = document.getElementById('welcomeScreen');

    let records = [];
    let currentId = null;

    function showAuth() { authView.classList.remove('hidden'); mainView.classList.add('hidden'); }
    function showMain() { authView.classList.add('hidden'); mainView.classList.remove('hidden'); }

    document.getElementById('loginBtn').onclick = async () => {
      const email = document.getElementById('email').value.trim();
      const password = document.getElementById('password').value;
      const { error } = await db.auth.signInWithPassword({ email, password });
      if (error) document.getElementById('authMsg').textContent = error.message;
      else { showMain(); loadRecords(); }
    };
    document.getElementById('signupBtn').onclick = async () => {
      const email = document.getElementById('email').value.trim();
      const password = document.getElementById('password').value;
      const { error } = await db.auth.signUp({ email, password });
      if (error) document.getElementById('authMsg').textContent = error.message;
      else document.getElementById('authMsg').textContent = '註冊成功，請登入。';
    };
    document.getElementById('logoutBtn').onclick = async () => { await db.auth.signOut(); showAuth(); };

    async function loadRecords() {
      const { data, error } = await db.from('records').select('*').order('created_at', { ascending: true });
      if (!error) { records = data; renderNav(); }
    }

    function renderNav() {
      navList.innerHTML = '';
      records.forEach(r => {
        const a = document.createElement('a');
        a.className = 'nav-item';
        a.textContent = r.title;
        a.onclick = () => viewRecord(r.id);
        navList.appendChild(a);
      });
    }

    function viewRecord(id) {
      const r = records.find(x => x.id === id);
      if (!r) return;
      currentId = id;
      
      document.getElementById('viewTitle').textContent = r.title;
      // 使用 Markdown 解析內容
      document.getElementById('viewContent').innerHTML = marked.parse(r.content);

      welcomeScreen.classList.add('hidden');
      editor.classList.add('hidden');
      viewer.classList.remove('hidden');
    }

    document.getElementById('newEntryBtn').onclick = () => {
      currentId = null;
      document.getElementById('editorTitle').textContent = '建立新條目';
      document.getElementById('editTitle').value = '';
      document.getElementById('editContent').value = '';
      welcomeScreen.classList.add('hidden');
      viewer.classList.add('hidden');
      editor.classList.remove('hidden');
    };

    document.getElementById('editBtn').onclick = () => {
      const r = records.find(x => x.id === currentId);
      if (!r) return;
      document.getElementById('editorTitle').textContent = '編輯條目：' + r.title;
      document.getElementById('editTitle').value = r.title;
      document.getElementById('editCategory').value = r.category;
      document.getElementById('editContent').value = r.content;
      viewer.classList.add('hidden');
      editor.classList.remove('hidden');
    };

    document.getElementById('cancelBtn').onclick = () => {
      if (currentId) viewRecord(currentId);
      else { welcomeScreen.classList.remove('hidden'); editor.classList.add('hidden'); }
    };

    document.getElementById('saveBtn').onclick = async () => {
      const title = document.getElementById('editTitle').value.trim();
      const category = document.getElementById('editCategory').value;
      const content = document.getElementById('editContent').value.trim();
      if (!title || !content) { alert('請填寫標題與內容'); return; }

      if (currentId) {
        const { error } = await db.from('records').update({ title, category, content }).eq('id', currentId);
        if (error) { alert('更新失敗：' + error.message); return; }
        const idx = records.findIndex(x => x.id === currentId);
        records[idx] = { ...records[idx], title, category, content };
      } else {
        const { data: { user } } = await db.auth.getUser();
        const { data, error } = await db.from('records').insert({ user_id: user.id, title, category, content }).select().single();
        if (error) { alert('新增失敗：' + error.message); return; }
        records.push(data);
        currentId = data.id;
      }
      renderNav();
      viewRecord(currentId);
    };

    document.getElementById('deleteBtn').onclick = async () => {
      if (!confirm('確定要刪除嗎？')) return;
      const { error } = await db.from('records').delete().eq('id', currentId);
      if (error) { alert('刪除失敗：' + error.message); return; }
      records = records.filter(x => x.id !== currentId);
      currentId = null;
      renderNav();
      viewer.classList.add('hidden');
      welcomeScreen.classList.remove('hidden');
    };

    (async () => {
      const { data: { session } } = await db.auth.getSession();
      if (session) { showMain(); loadRecords(); }
      else showAuth();
    })();
  </script>
</body>
</html>
