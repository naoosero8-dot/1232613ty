<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>画像5枚横並び + 役名表示（スマホ保存対応版）</title>

  <style>
    :root{
      --bg:#f7f8fa;
      --card-bg:#fff;
      --accent:#2b7cff;
      --muted:#6b7280;
      --danger:#e53e3e;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, Arial;
    }
    *{box-sizing:border-box}

    body{
      margin:0;
      background:var(--bg);
      color:#111827;
      padding:20px;
      line-height:1.4;
    }

    header h1{margin:0 0 6px}
    header p{margin:0 0 16px; color:var(--muted)}

    .input-area, .list-area{
      background:var(--card-bg);
      border-radius:10px;
      padding:14px;
      box-shadow:0 1px 3px rgba(0,0,0,0.05);
      margin-bottom:18px;
    }

    .role-input{display:flex; flex-direction:column; margin-bottom:14px}
    .role-input label{font-size:13px; color:var(--muted); margin-bottom:6px}
    .role-input input{padding:8px; border-radius:6px; border:1px solid #ddd}

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
    .slot .thumb img{
      width:100%; height:100%; object-fit:cover;
    }

    .controls{display:flex; gap:8px; flex-wrap:wrap; margin-top:6px}
    button{
      padding:8px 12px;
      border-radius:8px;
      border:0;
      background:var(--accent);
      color:#fff;
      cursor:pointer;
      font-weight:600;
    }
    #clear-btn{background:var(--danger)}
    button[disabled]{opacity:0.6; cursor:not-allowed}

    /* ▼ 横スクロール可能な横一列表示（スマホ対応） */
    .cards-list{
      padding:12px 8px;
      background:#fff;
      border-radius:8px;
      border:1px solid #e6e9ee;
    }
    .cards-row{
      display:flex;
      flex-wrap:nowrap;       /* 折り返し禁止（常に横一列） */
      gap:12px;
      overflow-x:auto;        /* 横スクロール許可（スマホ対応） */
      padding-bottom:10px;
    }
    .cards-row::-webkit-scrollbar{
      height:6px;
    }

    .card{
      width:140px;
      min-width:140px;
      background:#fff;
      border-radius:6px;
      overflow:hidden;
      border:1px solid #eef2f6;
    }
    .card .image{
      width:100%;
      aspect-ratio:4/3;
      background:#eef2ff;
      display:flex;
      align-items:center;
      justify-content:center;
    }
    .card .image img{
      width:100%; height:100%; object-fit:cover;
    }
    .placeholder{color:var(--muted); padding:12px; text-align:center}

    .role-display{
      margin-top:10px;
      text-align:center;
      font-weight:700;
      padding:8px 6px;
      background:#fff;
      border-radius:6px;
    }

    @media (max-width:520px){
      .card{width:120px; min-width:120px;}
    }
  </style>
</head>
<body>

<header>
  <h1>画像5枚横並び + 役名表示（スマホ保存対応）</h1>
  <p>スマホでも横スクロールで5枚が横並び。画像をスマホの写真に保存できます。</p>
</header>

<main>
  <section class="input-area">
    <h2>入力</h2>

    <div class="role-input">
      <label for="role-name">役名</label>
      <input id="role-name" type="text" placeholder="例：リーチ" maxlength="100">
    </div>

    <div id="slots" class="slots"></div>

    <div class="controls">
      <button type="button" id="preview-btn">一覧を表示</button>
      <button type="button" id="save-btn">保存</button>
      <button type="button" id="download-image-btn">画像として保存</button>
      <button type="button" id="clear-btn">全てクリア</button>
    </div>
  </section>

  <section class="list-area">
    <h2>一覧プレビュー</h2>
    <div id="cards-list" class="cards-list"></div>
  </section>
</main>

<footer>
  <small>スマホ対応保存機能つき</small>
</footer>


<!-- html2canvas -->
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>

<script>
(function(){
  const SLOT_COUNT = 5;
  const STORAGE_KEY = "five_images_role_under_singlefile_v3";

  const slotsContainer = document.getElementById("slots");
  const cardsList = document.getElementById("cards-list");
  const previewBtn = document.getElementById("preview-btn");
  const saveBtn = document.getElementById("save-btn");
  const clearBtn = document.getElementById("clear-btn");
  const roleInput = document.getElementById("role-name");
  const downloadBtn = document.getElementById("download-image-btn");

  const previewObjectUrls = new Map();

  /*------------ スロット作成 ------------*/
  function createSlots(){
    for(let i=0;i<SLOT_COUNT;i++){
      const slot = document.createElement("div");
      slot.className = "slot";
      slot.dataset.index = i;

      slot.innerHTML = `
        <div class="thumb"><span class="thumb-placeholder">画像を選択</span></div>
        <label for="file-${i}">画像 (${i+1})</label>
        <input type="file" id="file-${i}" accept="image/*">
      `;

      const fileInput = slot.querySelector(`#file-${i}`);
      const thumb = slot.querySelector(".thumb");

      fileInput.addEventListener("change", e => {
        const file = e.target.files[0];

        if(previewObjectUrls.has(i)){
          URL.revokeObjectURL(previewObjectUrls.get(i));
          previewObjectUrls.delete(i);
        }

        if(!file){
          thumb.innerHTML = `<span class="thumb-placeholder">画像を選択</span>`;
          return;
        }

        const objUrl = URL.createObjectURL(file);
        previewObjectUrls.set(i, objUrl);
        thumb.innerHTML = `<img src="${objUrl}" alt="">`;
      });

      slotsContainer.appendChild(slot);
    }
  }

  function fileToDataURL(file){
    return new Promise((res,rej)=>{
      const fr = new FileReader();
      fr.onload = ()=>res(fr.result);
      fr.onerror = rej;
      fr.readAsDataURL(file);
    });
  }

  /*------------ 入力収集 ------------*/
  async function gatherInputs(){
    const images=[];
    const slotEls = [...document.querySelectorAll(".slot")];

    for(const slot of slotEls){
      const idx = Number(slot.dataset.index);
      const input = slot.querySelector(`#file-${idx}`);
      let dataUrl = "";

      if(input.files[0]){
        dataUrl = await fileToDataURL(input.files[0]);
      } else {
        const img = slot.querySelector(".thumb img");
        if(img) dataUrl = img.src;
      }
      images.push({ image:dataUrl });
    }

    return {
      role: roleInput.value.trim(),
      images
    };
  }

  /*------------ プレビュー描画 ------------*/
  function renderList(payload){
    cardsList.innerHTML = "";

    const row = document.createElement("div");
    row.className = "cards-row";

    payload.images.forEach((it,i)=>{
      const card = document.createElement("div");
      card.className = "card";

      if(it.image){
        card.innerHTML = `<div class="image"><img src="${it.image}" alt=""></div>`;
      } else {
        card.innerHTML = `<div class="image"><div class="placeholder">画像なし</div></div>`;
      }
      row.appendChild(card);
    });

    cardsList.appendChild(row);

    const role = document.createElement("div");
    role.className = "role-display";
    role.textContent = payload.role || "（役名未設定）";
    cardsList.appendChild(role);
  }

  /*------------ 保存/読込 ------------*/
  function saveToStorage(payload){
    localStorage.setItem(STORAGE_KEY, JSON.stringify(payload));
    saveBtn.textContent = "保存しました";
    saveBtn.disabled = true;
    setTimeout(()=>{saveBtn.textContent="保存"; saveBtn.disabled=false;},1200);
  }
  function loadFromStorage(){
    try{
      const d = localStorage.getItem(STORAGE_KEY);
      return d ? JSON.parse(d) : null;
    }catch(e){return null;}
  }

  function restoreToForm(payload){
    if(!payload) return;

    roleInput.value = payload.role || "";
    const slotEls=[...document.querySelectorAll(".slot")];

    slotEls.forEach((slot,i)=>{
      const thumb = slot.querySelector(".thumb");
      const input = slot.querySelector("input");
      if(payload.images[i]?.image){
        thumb.innerHTML = `<img src="${payload.images[i].image}" alt="">`;
        input.value="";
      }else{
        thumb.innerHTML=`<span class="thumb-placeholder">画像を選択</span>`;
      }
    });
  }

  /*------------ クリア ------------*/
  function clearAll(){
    if(!confirm("本当に全てクリアしますか？")) return;
    localStorage.removeItem(STORAGE_KEY);
    roleInput.value="";
    const slotEls=[...document.querySelectorAll(".slot")];
    slotEls.forEach(slot=>{
      const thumb = slot.querySelector(".thumb");
      thumb.innerHTML=`<span class="thumb-placeholder">画像を選択</span>`;
      slot.querySelector("input").value="";
    });
    cardsList.innerHTML="";
  }


  /*-----------------------------------------
      ★ スマホ対応：画像として保存
      iPhone → 新しいタブに画像 → 長押し保存
      Android/PC → 自動ダウンロード
  -----------------------------------------*/
  downloadBtn.addEventListener("click", async () => {
    if(!cardsList.innerHTML.trim()){
      alert("先に一覧を表示してください");
      return;
    }

    const target = cardsList;

    const prevShadow = target.style.boxShadow;
    const prevBorder = target.style.border;
    target.style.boxShadow="none";
    target.style.border="none";

    const canvas = await html2canvas(target, {scale:2, useCORS:true});

    target.style.boxShadow=prevShadow;
    target.style.border=prevBorder;

    const dataUrl = canvas.toDataURL("image/png");
    const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);

    if(isIOS){
      const tab = window.open();
      tab.document.write(`<img src="${dataUrl}" style="width:100%;">`);
      return;
    }

    const link = document.createElement("a");
    link.download = "cards-list.png";
    link.href = dataUrl;
    link.click();
  });

  /*------------ 初期化 ------------*/
  function init(){
    createSlots();

    const saved = loadFromStorage();
    if(saved){
      restoreToForm(saved);
      renderList(saved);
    }

    previewBtn.addEventListener("click", async ()=>{
      const p = await gatherInputs();
      renderList(p);
    });

    saveBtn.addEventListener("click", async ()=>{
      const p = await gatherInputs();
      saveToStorage(p);
    });

    clearBtn.addEventListener("click", clearAll);
  }

  init();
})();
</script>

</body>
</html>
