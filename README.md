[chaya2.index.html](https://github.com/user-attachments/files/26270979/chaya2.index.html)
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8"><!-- 文字コード指定 -->
<title>食材保管庫</title><!-- ページタイトル -->
<meta name="viewport" content="width=device-width, initial-scale=1"><!-- レスポンシブ対応 -->
<style>
/* ===== 色設定ここから（ここだけ触ればOK） ===== */
:root {
  --bg-color: #ffffff;        /* ページ背景色 */
  --text-color: #000000;      /* 通常文字色 */
  --accent-color: #808080;    /* ボタン色 */
  --result-color: #696969;    /* 結果文字色 */
  --box-bg-color: #C0C0C0;    /* 履歴背景 */

  --result-border: #808080;  /* 通常時の結果枠線 */
  --result-bg: #f5f5f5;      /* 通常時の結果背景 */
  --error-color: #8b3a3a;    /* 誤入力色 */

  /* ▼レア演出用カラー（ここを変えるとレア時の見た目が変わる） */
  --rare-border: #d4af37;    /* レア時の枠線色（金色） */
  --rare-bg: #fff8dc;        /* レア時の背景色 */
  --rare-text: #ff0000;      /* レア時の文字色（赤） */
}
/* ===== 色設定ここまで ===== */

body {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
  padding: 20px;
  background: var(--bg-color);
  color: var(--text-color);
}

h1 { font-size: 1.4em; }

input {
  font-size: 1em;
  padding: 10px;
  margin-top: 10px;
  width: 70%;
}

button {
  font-size: 1em;
  padding: 10px 14px;
  margin-top: 10px;
  margin-left: 5px;
  background: var(--accent-color);
  color: #fff;
  border: none;
  border-radius: 6px;
}

button:hover { opacity: 0.9; }

/* ===== 結果表示エリア ===== */
#result-box {
  margin-top: 20px;
  padding: 10px;
  border: 4px double var(--result-border);
  border-radius: 10px;
  background: var(--result-bg);
  transition: all 0.5s ease;
}

#result-title {
  font-size: 0.9em;
  color: #555;
  margin-bottom: 6px;
}

#result {
  font-size: 1.2em;
  font-weight: bold;
  color: var(--result-color);
  opacity: 0;
  transition: all 0.5s ease;
}

/* ===== 履歴ボックス ===== */
#history-box {
  margin-top: 30px;
  background: var(--box-bg-color);
  padding: 15px;
  border-radius: 8px;
}

#history-box h2 {
  font-size: 1.1em;
  margin-bottom: 10px;
}

ul { padding-left: 20px; }

.reare { color: var(--rare-text); font-weight: bold; }
</style>
</head>
<body>

<h1>【部屋の名前】</h1>
<b>食材</b>とだけ発言すると食材が一種類選ばれるよ<br><br>

<!-- 入力欄 -->
<input id="word" placeholder="入力"
  onkeydown="if(event.key==='Enter') run();">
<button onclick="run()">発動</button>

<!-- 結果表示 -->
<div id="result-box">
  <div id="result-title"><b>結果</b></div>
  <div id="result"></div>
</div>

<!-- 履歴表示 -->
<div id="history-box">
  <h2>発動履歴</h2>
  <ul id="history"></ul>
</div>

<!-- Firebase SDK -->
<script src="www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
/* ===== Firebase 設定 ===== */
const firebaseConfig = {
  apiKey: "AIzaSyBtjo76to-489vhWVmZtCIJGF5TEY7Cnsg",
  authDomain: "chaya-nintama.firebaseapp.com",
  databaseURL: "chaya-nintama-default-rtdb.firebaseio.com",
  projectId: "chaya-nintama",
  storageBucket: "chaya-nintama.firebasestorage.app",
  messagingSenderId: "28953695885",
  appId: "1:28953695885:web:7f0474b19eb8c66f3074c4"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

/* ===== 管理者設定 ===== */
const ADMIN_PASSWORD = "豆腐ラブ"; // ←ここを変更
let isAdmin = false;

/* ===== 基本設定 ===== */
const triggerWord = "食材";
const MAX_HISTORY = 50;

/* ===== 食材リスト（通常） ===== */
let items = [
  "団子とお茶",
  "団子３本、持ち帰りで",
  "お茶をこぼした",
  "何となく欲しくなる、せんべい"
];

/* ===== レア食材リスト ===== */
let rareItems = [
  "【大当たり】今日は豆腐パーティー確定！",
  "【激レア】謎の高級食材が届いた…！"
];

/* ===== 直前結果保持 ===== */
let lastResult = null;

/* ===== 結果処理 ===== */
function run() {
  const input = document.getElementById("word").value.trim();
  const resultEl = document.getElementById("result");
  const resultBox = document.getElementById("result-box");
  resultEl.style.opacity = 0;

  setTimeout(() => {

    /* 管理者ログイン */
    if (input === `管理者:${ADMIN_PASSWORD}`) {
      isAdmin = true;
      resultEl.textContent = "管理者モードに入りました";
      resultEl.style.color = "var(--result-color)";
      resultEl.style.opacity = 1;
      return;
    }

    /* 管理者終了 */
    if (input === "管理者終了") {
      isAdmin = false;
      resultEl.textContent = "管理者モードを終了しました";
      resultEl.style.opacity = 1;
      return;
    }

    /* 管理者操作 */
    if (isAdmin) {
      if (input.startsWith("追加:")) {
        items.push(input.replace("追加:", ""));
        resultEl.textContent = "通常アイテムを追加しました";
        resultEl.style.opacity = 1;
        return;
      }
      if (input.startsWith("レア追加:")) {
        rareItems.push(input.replace("レア追加:", ""));
        resultEl.textContent = "レアアイテムを追加しました";
        resultEl.style.opacity = 1;
        return;
      }
    }

    /* 通常ユーザー */
    if (input !== triggerWord) {
      resultEl.style.color = "var(--error-color)";
      resultEl.textContent = "「食材」と発言した時だけ発動するよ";
      resultEl.style.opacity = 1;
      return;
    }

    let result;
    let isRare = false;

    /* レア抽選 25% */
    if (Math.random() < 0.25) {
      result = rareItems[Math.floor(Math.random() * rareItems.length)];
      isRare = true;
    } else {
      do {
        result = items[Math.floor(Math.random() * items.length)];
        isRare = false;
      } while (result === lastResult);
    }

    /* 見た目処理 */
    if (isRare) {
      resultBox.style.border = "4px double var(--rare-border)";
      resultBox.style.background = "var(--rare-bg)";
      resultEl.style.color = "var(--rare-text)";
      resultEl.style.transform = "scale(1.1) rotate(-5deg)";
      setTimeout(()=>{ resultEl.style.transform="scale(1) rotate(0deg)"; }, 300);
    } else {
      resultBox.style.border = "4px double var(--result-border)";
      resultBox.style.background = "var(--result-bg)";
      resultEl.style.color = "var(--result-color)";
    }

    lastResult = result;
    resultEl.textContent = result;
    resultEl.style.opacity = 1;

    saveHistory(result); // Firebase 保存
    renderHistory();     // 履歴描画

  }, 50);
}

/* ===== Firebase に履歴保存 ===== */
function saveHistory(text) {
  const historyRef = db.ref("history");
  const entry = { text, time: new Date().toLocaleString() };
  historyRef.push(entry);
  historyRef.limitToLast(MAX_HISTORY);
}

/* ===== 履歴描画 ===== */
function renderHistory() {
  const ul = document.getElementById("history");
  ul.innerHTML = "";
  db.ref("history").limitToLast(MAX_HISTORY).once("value", snapshot => {
    snapshot.forEach(child => {
      const li = document.createElement("li");
      li.textContent = `${child.val().time}：${child.val().text}`;
      if (rareItems.includes(child.val().text)) li.classList.add("reare");
      ul.appendChild(li);
    });
  });
}

/* ページロード時に履歴表示 */
renderHistory();
</script>
</body>
</html>
