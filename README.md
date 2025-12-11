<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>画像5枚横並び + 役名表示（修正版・単一ファイル）</title>
  <style>
    :root{
      --bg:#f7f8fa;
      --card-bg:#fff;
      --accent:#2b7cff;
      --muted:#6b7280;
      --danger:#e53e3e;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }

    *{box-sizing:border-box}
    body{
      margin:0;
      background:var(--bg);
      color:#111827;
      padding:20px;
      line-height:1.4;
    }

    header h1{margin:0 0 6px 0}
    header p{margin:0 0 16px 0; color:var(--muted)}

    .input-area, .list-area{
      background:var(--card-bg);
      border-radius:10px;
      padding:14px;
      box-shadow:0 1px 3px rgba(16,24,40,0.04);
      margin-bottom:18px;
    }

    /* 入力エリア */
    .role-input{
      display:flex;
      flex-direction:column;
      margin-bottom:12px;
    }
    .role-input label{font-size:13px; color:var(--muted); margin-bottom:6px}
    .role-input input[type="text"]{padding:8px; border-radius:6px; border:1px solid #e6e9ee}

    .slots{
      display:grid;
      grid-template-columns:repeat(auto-fit,minmax(140px,1fr));
      gap:10px;
      margin-bottom:12px;
    }
    .slot{
      border:1px solid #e6e9ee;
      padding:10px;
      border-radius:8px;
      background:#fff;
      display:flex;
      flex-direction:column;
      gap:8px;
      align-items:stretch;
    }
    .slot .thumb{
      width:100%;
      aspect-ratio:4/3;
      background:#f1f5f9;
      border-radius:6px;
      display:flex;
      align-items:center;
      justify-content:center;
      overflow:hidden;
    }
    .slot .thumb img{width:100%; height:100%; object-fit:cover; display:block}
    .slot label{font-size:13px; color:var(--muted); margin-bottom:4px}
    .slot input[type="file"]{padding:6px}

    .controls{
      display:flex;
      gap:8px;
      justify-content:flex-start;
      flex-wrap:wrap;
      margin-top:6px;
    }
    button{
      padding:8px 12px;
      border-radius:8px;
      border:0;
      background:var(--accent);
      color:#fff;
      cursor:pointer;
      font-weight:600;
    }
    button#clear-btn{background:var(--danger)}
    button[disabled]{opacity:0.6; cursor:not-allowed}

    /* プレビュー：画像5枚を横並びに表示 */
    .cards-list{
      padding:12px 8px;
      background:#fff;
      border-radius:8px;
      border:1px solid #e6e9ee;
    }
    .cards-row{
      display:flex;
      gap:12px;
      align-items:flex-start;
      justify-content:center;
      flex-wrap:wrap; /* 小さい画面では折り返す */
      margin-bottom:12px;
    }
    .card{
      width:140px;
      background:#fff;
      border-radius:6px;
      overflow:hidden;
      text-align:center;
      border:1px solid #eef2f6;
    }
    .card .image{
      width:100%;
      aspect-ratio:4/3;
      background:#eef2ff;
      display:flex;
      align-items:center;
      justify-content:center;
      overflow:hidden;
    }
    .card .image img{width:100%; height:100%; object-fit:cover; display:block}
    .card .placeholder{padding:12px;color:var(--muted)}

    /* 役名（画像群の下、中央寄せ） */
    .role-display{
      text-align:center;
      font-weight:700;
      padding:8px 6px;
      color:#111827;
      background:linear-gradient(180deg, rgba(255,255,255,0.6), rgba(255,255,255,0.9));
      border-radius:6px;
    }

    /* 小さい画面用 */
    @media (max-width:520px){
      .card{ width:100px; }
    }
    footer { margin-top:8px; color:var(--muted); font-size:13px; }
  </style>
</head>
<body>
  <header>
    <h1>画像5枚横並び + 役名表示（修正版）</h1>
    <p>画像を5枚アップロードし、画像群の下に役名を1つ表示します。修正版では「復元後も再アップロードせずに保存できる」など、アップロードの使い勝手を改善しました。</p>
  </header>

  <main>
    <section class="input-area" aria-labelledby="input-heading">
      <h2 id="input-heading">入力</h2>

      <form id="cards-form" onsubmit="return false;">
        <div class="role-input">
          <label for="role-name">役名（全体に表示）</label>
          <input type="text" id="role-name" placeholder="例：リーチ" maxlength="100" />
        </div>

        <div class="slots" id="slots" aria-label="画像アップロードスロット">
          <!-- スロットは JavaScript で生成 -->
        </div>

        <div class="controls">
          <button type="button" id="preview-btn">一覧を表示</button>
          <button type="button" id="save-btn">保存</button>
          <button type="button" id="clear-btn">全てクリア</button>
        </div>
      </form>
    </section>

    <section class="list-area" aria-labelledby="list-heading">
      <h2 id="list-heading">一覧プレビュー</h2>
      <div id="cards-list" class="cards-list" role="region" aria-live="polite">
        <!-- プレビュー（横並びの画像群 + 役名） -->
      </div>
    </section>
  </main>

  <footer>
    <small>作成: 単一HTMLファイル版 — 画像はブラウザのファイルを使用します</small>
  </footer>

  <script>
    (function(){
      const SLOT_COUNT = 5;
      const slotsContainer = document.getElementById('slots');
      const cardsList = document.getElementById('cards-list');
      const previewBtn = document.getElementById('preview-btn');
      const saveBtn = document.getElementById('save-btn');
      const clearBtn = document.getElementById('clear-btn');
      const roleInput = document.getElementById('role-name');

      const STORAGE_KEY = 'five_images_role_under_singlefile_v2';

      // 一時的にプレビュー用の objectURL を保持して revoke できるようにする
      const previewObjectUrls = new Map();

      // スロットを作成
      function createSlots(){
        for(let i=0;i<SLOT_COUNT;i++){
          const slot = document.createElement('div');
          slot.className = 'slot';
          slot.dataset.index = i;

          slot.innerHTML = `
            <div class="thumb" aria-hidden="true">
              <span class="thumb-placeholder">画像を選択</span>
            </div>
            <label for="file-${i}">画像 (${i+1})</label>
            <input type="file" accept="image/*" id="file-${i}" />
          `;

          const fileInput = slot.querySelector(`#file-${i}`);
          const thumb = slot.querySelector('.thumb');

          fileInput.addEventListener('change', (ev) => {
            const file = ev.target.files && ev.target.files[0];
            // revoke previous objectURL for this slot
            if(previewObjectUrls.has(i)){
              URL.revokeObjectURL(previewObjectUrls.get(i));
              previewObjectUrls.delete(i);
            }

            if(!file){
              thumb.innerHTML = `<span class="thumb-placeholder">画像を選択</span>`;
              return;
            }

            // preview via object URL (fast). We'll still convert to DataURL when saving.
            const objUrl = URL.createObjectURL(file);
            previewObjectUrls.set(i, objUrl);
            thumb.innerHTML = '';
            const img = document.createElement('img');
            img.src = objUrl;
            img.alt = `選択画像 ${i+1}`;
            thumb.appendChild(img);
          });

          slotsContainer.appendChild(slot);
        }
      }

      // ファイルをdataURLに変換（保存用）
      function fileToDataURL(file){
        return new Promise((res, rej) => {
          const fr = new FileReader();
          fr.onload = () => res(fr.result);
          fr.onerror = rej;
          fr.readAsDataURL(file);
        });
      }

      // 入力を集める（roleは1つ）
      // 重要な修正点：
      //  - 「復元したプレビュー画像があるが input.files が空」の場合、プレビュー中の img.src（DataURL）を使えるようにする。
      //  - ファイルが選択されている場合は fileToDataURL(file) を使って DataURL を作成する。
      async function gatherInputs(){
        const images = [];
        const slotEls = Array.from(document.querySelectorAll('.slot'));
        for(const slot of slotEls){
          const idx = Number(slot.dataset.index);
          const fileInput = slot.querySelector(`#file-${idx}`);
          let dataUrl = '';

          // 1) ファイル入力にファイルがある場合はそれをDataURL化
          if(fileInput.files && fileInput.files[0]){
            try {
              dataUrl = await fileToDataURL(fileInput.files[0]);
            } catch(e){
              console.error('fileToDataURL error', e);
            }
          } else {
            // 2) fileInput が空でも、プレビュー用の img がある（復元済み）なら、その src を使う
            const img = slot.querySelector('.thumb img');
            if(img && img.src){
              dataUrl = img.src;
            } else {
              dataUrl = '';
            }
          }

          images.push({ image: dataUrl });
        }
        return {
          role: roleInput.value.trim(),
          images
        };
      }

      // レンダリング：画像を横並びにし、下に役名を1つ表示
      function renderList(payload){
        cardsList.innerHTML = '';

        const row = document.createElement('div');
        row.className = 'cards-row';
        // 画像カード（空のスロットはプレースホルダ表示）
        payload.images.forEach((it, idx) => {
          const card = document.createElement('div');
          card.className = 'card';
          if(it.image){
            // dataURL or objectURL are fine for src
            card.innerHTML = `<div class="image"><img src="${it.image}" alt="画像 ${idx+1}" /></div>`;
          } else {
            card.innerHTML = `<div class="image"><div class="placeholder">画像なし</div></div>`;
          }
          row.appendChild(card);
        });

        cardsList.appendChild(row);

        const roleDiv = document.createElement('div');
        roleDiv.className = 'role-display';
        roleDiv.textContent = payload.role || '（役名未設定）';
        cardsList.appendChild(roleDiv);
      }

      // 保存 / 読み込み（localStorage）
      function saveToStorage(payload){
        try{
          // 注意: localStorage にはサイズ制限があります。大量の大きな画像を保存すると失敗します。
          localStorage.setItem(STORAGE_KEY, JSON.stringify(payload));
          saveBtn.textContent = '保存しました';
          saveBtn.disabled = true;
          setTimeout(()=>{ saveBtn.textContent = '保存'; saveBtn.disabled = false; }, 1200);
        }catch(e){
          console.error('保存失敗', e);
          alert('保存に失敗しました（容量制限の可能性があります）：' + e.message);
        }
      }

      function loadFromStorage(){
        try{
          const raw = localStorage.getItem(STORAGE_KEY);
          if(!raw) return null;
          return JSON.parse(raw);
        }catch(e){
          console.error('読み込み失敗', e);
          return null;
        }
      }

      // 復元：画像はDataURLでプレビューを表示する（file input 自体には戻せない）
      function restoreToForm(payload){
        if(!payload) return;
        roleInput.value = payload.role || '';
        const slotEls = Array.from(document.querySelectorAll('.slot'));
        slotEls.forEach((slot, i) => {
          const thumb = slot.querySelector('.thumb');
          const fileInput = slot.querySelector(`#file-${i}`);
          // revoke previous objectURL if any
          if(previewObjectUrls.has(i)){
            URL.revokeObjectURL(previewObjectUrls.get(i));
            previewObjectUrls.delete(i);
          }

          if(payload.images?.[i]?.image){
            thumb.innerHTML = '';
            const img = document.createElement('img');
            img.src = payload.images[i].image; // DataURL saved previously
            img.alt = `復元画像 ${i+1}`;
            thumb.appendChild(img);
            fileInput.value = ''; // cannot set File object
          } else {
            thumb.innerHTML = `<span class="thumb-placeholder">画像を選択</span>`;
            fileInput.value = '';
          }
        });
      }

      // クリア
      function clearAll(){
        if(!confirm('本当に全ての入力と保存をクリアしますか？')) return;
        roleInput.value = '';
        const slotEls = Array.from(document.querySelectorAll('.slot'));
        slotEls.forEach((slot, i) => {
          const fileInput = slot.querySelector(`#file-${i}`);
          const thumb = slot.querySelector('.thumb');
          // revoke objectURL if any
          if(previewObjectUrls.has(i)){
            URL.revokeObjectURL(previewObjectUrls.get(i));
            previewObjectUrls.delete(i);
          }
          fileInput.value = '';
          thumb.innerHTML = `<span class="thumb-placeholder">画像を選択</span>`;
        });
        localStorage.removeItem(STORAGE_KEY);
        cardsList.innerHTML = '';
      }

      // 初期化
      function init(){
        createSlots();

        const saved = loadFromStorage();
        if(saved){
          restoreToForm(saved);
          renderList(saved);
        }

        previewBtn.addEventListener('click', async () => {
          const payload = await gatherInputs();
          renderList(payload);
        });

        saveBtn.addEventListener('click', async () => {
          const payload = await gatherInputs();
          saveToStorage(payload);
        });

        clearBtn.addEventListener('click', clearAll);
      }

      // run
      init();

      // ページを離れるときに objectURL を確実に解放
      window.addEventListener('beforeunload', () => {
        previewObjectUrls.forEach((url) => {
          try { URL.revokeObjectURL(url); } catch(e){ /* ignore */ }
        });
        previewObjectUrls.clear();
      });
    })();
  </script>
</body>
</html>
