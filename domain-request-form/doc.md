# 必要なこと

## Dockerの設定(Caddyの設定込み)

`Dockerfile`
```Dockerfile
FROM node:22-alpine

WORKDIR /app
COPY package.json .
RUN npm install

COPY app.js .
CMD ["node", "app.js"]
```

`docker-compose.yml`
```
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "31515:31515/tcp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./certs:/certs:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app
    networks:
      - internal
      
  app:
    build: .
    ports:
      - "8080:3000"
    volumes:
      - ./data:/app/data
    environment:
      ADMIN_USER: admin
      ADMIN_PASS: change-me
    networks:
      - internal

networks:
  internal:

volumes:
  caddy_data:
  caddy_config: 
```

`Caddyfile`
```
https://domain-request.koshizukalab.dataspace.internal:31515 {
    tls /certs/drf.crt /certs/drf.key
    reverse_proxy app:3000 {
        header_up Host {host}
        header_up X-Forwarded-Proto https
        header_up X-Forwarded-Host {host}
        header_up X-Forwarded-Port 32443
    }
}
```

./certsのディレクトリに入れる秘密鍵と証明書(.crt)ファイルについて

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
-keyout drf.key \
-out drf.crt \
-subj "/CN=domain-request.koshizukalab.dataspace.internal" \
-addext "subjectAltName=DNS:domain-request.koshizukalab.dataspace.internal"
```

## Dockerコンテナ内で動かすnpmプログラム周り

`app.js`
```
const express = require("express");
const Database = require("better-sqlite3");

const app = express();
const db = new Database("/app/data/requests.db");

app.use(express.urlencoded({ extended: true }));

db.exec(`
CREATE TABLE IF NOT EXISTS requests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  requester TEXT NOT NULL,
  domain TEXT NOT NULL,
  ip_address TEXT NOT NULL,
  note TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
`);

function escapeHtml(value) {
  return String(value || "")
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#039;");
}

function adminAuth(req, res, next) {
  const auth = req.headers.authorization || "";
  const token = auth.split(" ")[1] || "";
  const [user, pass] = Buffer.from(token, "base64").toString().split(":");

  if (user === process.env.ADMIN_USER && pass === process.env.ADMIN_PASS) {
    return next();
  }

  res.setHeader("WWW-Authenticate", 'Basic realm="Admin Page"');
  return res.status(401).send("Authentication required");
}

app.get("/", (req, res) => {
  res.send(`
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>DNS登録申請フォーム</title>
  <style>
    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      background: linear-gradient(135deg, #eef2ff, #f8fafc);
      color: #111827;
    }
    .container {
      max-width: 760px;
      margin: 64px auto;
      padding: 24px;
    }
    .card {
      background: rgba(255,255,255,0.94);
      border-radius: 28px;
      padding: 44px;
      box-shadow: 0 24px 60px rgba(15, 23, 42, 0.12);
    }
    .badge {
      display: inline-block;
      padding: 8px 14px;
      border-radius: 999px;
      background: #eff6ff;
      color: #2563eb;
      font-size: 14px;
      font-weight: 700;
      margin-bottom: 18px;
    }
    h1 {
      margin: 0;
      font-size: 36px;
      letter-spacing: -0.04em;
    }
    .description {
      color: #6b7280;
      margin: 16px 0 36px;
      line-height: 1.7;
    }
    label {
      display: block;
      font-weight: 700;
      margin-bottom: 10px;
    }
    input, textarea {
      width: 100%;
      padding: 16px 18px;
      font-size: 17px;
      border: 1px solid #d1d5db;
      border-radius: 16px;
      box-sizing: border-box;
      margin-bottom: 24px;
    }
    input:focus, textarea:focus {
      outline: none;
      border-color: #2563eb;
      box-shadow: 0 0 0 5px rgba(37, 99, 235, 0.14);
    }
    textarea {
      min-height: 140px;
      resize: vertical;
    }
    .hint {
      display: block;
      margin-top: -16px;
      margin-bottom: 24px;
      color: #6b7280;
      font-size: 14px;
    }
    button {
      width: 100%;
      padding: 18px;
      border: none;
      border-radius: 18px;
      background: linear-gradient(135deg, #2563eb, #1d4ed8);
      color: white;
      font-size: 18px;
      font-weight: 800;
      cursor: pointer;
    }
    button:hover {
      filter: brightness(0.96);
    }
    .footer {
      margin-top: 24px;
      text-align: center;
      color: #9ca3af;
      font-size: 13px;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <div class="badge">Internal DNS Request</div>
      <h1>DNS登録申請フォーム</h1>
      <p class="description">
        dataspace.internal 用のドメイン名と IP アドレスの登録申請を行います。
      </p>

      <form method="POST" action="/submit">
        <label>申請者</label>
        <input name="requester" required placeholder="例：山田 太郎">

        <label>ドメイン名</label>
        <input
          name="domain"
          required
          placeholder="host.site.dataspace.internal"
          pattern="^[a-z0-9-]+\\.[a-z0-9-]+\\.dataspace\\.internal$">

        <span class="hint">
          形式：&lt;ホスト名&gt;.&lt;サイト名&gt;.dataspace.internal
        </span>

        <label>IPアドレス</label>
        <input
          name="ip_address"
          required
          placeholder="例：10.250.0.10"
          pattern="^((25[0-5]|2[0-4][0-9]|1?[0-9]{1,2})\\.){3}(25[0-5]|2[0-4][0-9]|1?[0-9]{1,2})$">

        <label>追記事項</label>
        <textarea name="note" placeholder="用途や補足事項を記入してください"></textarea>

        <button type="submit">申請する</button>
      </form>
    </div>
    <div class="footer">Internal DNS Request System</div>
  </div>
</body>
</html>
`);
});

app.post("/submit", (req, res) => {
  const { requester, domain, ip_address, note } = req.body;

  const domainRegex = /^[a-z0-9-]+\.[a-z0-9-]+\.dataspace\.internal$/;
  const ipRegex =
    /^((25[0-5]|2[0-4][0-9]|1?[0-9]{1,2})\.){3}(25[0-5]|2[0-4][0-9]|1?[0-9]{1,2})$/;

  if (!requester || !domain || !ip_address) {
    return res.status(400).send("申請者、ドメイン名、IPアドレスは必須です。");
  }

  if (!domainRegex.test(domain)) {
    return res
      .status(400)
      .send("ドメイン名は <ホスト名>.<サイト名>.dataspace.internal の形式で入力してください。");
  }

  if (!ipRegex.test(ip_address)) {
    return res.status(400).send("IPアドレスの形式が正しくありません。");
  }

  db.prepare(`
    INSERT INTO requests (requester, domain, ip_address, note)
    VALUES (?, ?, ?, ?)
  `).run(requester, domain, ip_address, note || "");

  res.send(`
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>申請完了</title>
  <style>
    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      background: linear-gradient(135deg, #ecfdf5, #f8fafc);
      color: #111827;
    }
    .container {
      max-width: 680px;
      margin: 80px auto;
      padding: 24px;
    }
    .card {
      background: white;
      border-radius: 28px;
      padding: 44px;
      box-shadow: 0 24px 60px rgba(15, 23, 42, 0.12);
      text-align: center;
    }
    .icon {
      width: 64px;
      height: 64px;
      border-radius: 50%;
      margin: 0 auto 20px;
      background: #dcfce7;
      color: #16a34a;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 34px;
      font-weight: 800;
    }
    h1 {
      font-size: 34px;
      margin: 0 0 12px;
      letter-spacing: -0.03em;
    }
    .description {
      color: #6b7280;
      margin-bottom: 28px;
    }
    .result {
      background: #f9fafb;
      border: 1px solid #e5e7eb;
      border-radius: 18px;
      padding: 22px;
      margin: 24px 0;
      text-align: left;
      word-break: break-word;
    }
    .label {
      color: #6b7280;
      font-size: 14px;
      margin-bottom: 6px;
    }
    .value {
      font-size: 18px;
      font-weight: 700;
      margin-bottom: 16px;
    }
    a {
      display: inline-block;
      margin-top: 12px;
      padding: 14px 22px;
      border-radius: 14px;
      background: #2563eb;
      color: white;
      text-decoration: none;
      font-weight: 700;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <div class="icon">✓</div>
      <h1>申請を受け付けました</h1>
      <p class="description">以下の内容で DNS 登録申請を保存しました。</p>

      <div class="result">
        <div class="label">申請者</div>
        <div class="value">${escapeHtml(requester)}</div>

        <div class="label">ドメイン名</div>
        <div class="value">${escapeHtml(domain)}</div>

        <div class="label">IPアドレス</div>
        <div class="value">${escapeHtml(ip_address)}</div>
      </div>

      <a href="/">別の申請を行う</a>
    </div>
  </div>
</body>
</html>
`);
});

app.get("/admin", adminAuth, (req, res) => {
  const rows = db.prepare("SELECT * FROM requests ORDER BY id DESC").all();

  res.send(`
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>DNS申請管理ページ</title>
  <style>
    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      background: #f8fafc;
      color: #111827;
    }
    .container {
      max-width: 1400px;
      margin: 48px auto;
      padding: 24px;
    }
    .header {
      margin-bottom: 28px;
    }
    .badge {
      display: inline-block;
      padding: 8px 14px;
      border-radius: 999px;
      background: #eff6ff;
      color: #2563eb;
      font-size: 14px;
      font-weight: 700;
      margin-bottom: 14px;
    }
    h1 {
      margin: 0;
      font-size: 36px;
      letter-spacing: -0.04em;
    }
    .description {
      color: #6b7280;
      margin-top: 10px;
    }
    .card {
      background: white;
      border-radius: 24px;
      box-shadow: 0 18px 40px rgba(15, 23, 42, 0.08);
      overflow: hidden;
    }
    table {
      width: 100%;
      border-collapse: collapse;
    }
    th {
      background: #f9fafb;
      color: #374151;
      text-align: left;
      font-size: 14px;
      padding: 16px;
      border-bottom: 1px solid #e5e7eb;
    }
    td {
      padding: 16px;
      border-bottom: 1px solid #f1f5f9;
      vertical-align: top;
      font-size: 15px;
      word-break: break-word;
    }
    tr:hover {
      background: #f8fafc;
    }
    .domain {
      font-weight: 700;
      color: #1d4ed8;
    }
    .ip {
      font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
      background: #f3f4f6;
      padding: 4px 8px;
      border-radius: 8px;
      display: inline-block;
    }
    .empty {
      padding: 40px;
      text-align: center;
      color: #6b7280;
    }
    .nav {
      margin-top: 24px;
    }
    .nav a {
      display: inline-block;
      padding: 12px 18px;
      border-radius: 12px;
      background: #2563eb;
      color: white;
      text-decoration: none;
      font-weight: 700;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div class="badge">Admin</div>
      <h1>DNS申請管理ページ</h1>
      <p class="description">登録申請されたドメイン名と IP アドレスの一覧です。</p>
    </div>

    <div class="card">
      ${
        rows.length === 0
          ? `<div class="empty">まだ申請はありません。</div>`
          : `
      <table>
        <tr>
          <th>ID</th>
          <th>申請者</th>
          <th>ドメイン名</th>
          <th>IPアドレス</th>
          <th>追記事項</th>
          <th>申請日時</th>
        </tr>
        ${rows.map(row => `
          <tr>
            <td>${escapeHtml(row.id)}</td>
            <td>${escapeHtml(row.requester)}</td>
            <td class="domain">${escapeHtml(row.domain)}</td>
            <td><span class="ip">${escapeHtml(row.ip_address)}</span></td>
            <td>${escapeHtml(row.note)}</td>
            <td>${escapeHtml(row.created_at)}</td>
          </tr>
        `).join("")}
      </table>
      `
      }
    </div>

    <div class="nav">
      <a href="/">申請フォームへ戻る</a>
    </div>
  </div>
</body>
</html>
`);
});

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

`package.json`
```
{
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "better-sqlite3": "^11.9.1",
    "express": "^4.18.3"
  }
}
```

## メール送付用のプログラム(`./watchdog`ディレクトリ内)

``クライアントID、シークレット情報
```
{
  "installed": {
    "client_id": "xxx",
    "project_id": "my-project-gmail-498203",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "yyy",
    "redirect_uris": [
      "http://localhost"
    ]
  }
}
```


`notifier.py`: メールを送信するプログラム

```
import base64
import json
import os
import sqlite3
from email.mime.text import MIMEText

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build


SCOPES = ["https://www.googleapis.com/auth/gmail.send"]

DB_PATH = "../data/requests.db"
STATE_PATH = "./notifier_state.json"

MAIL_TO = "seike.hirotsugu@gmail.com"
MAIL_SUBJECT = "DNS申請フォームに新しい申請がありました"


def get_gmail_service():
    creds = None

    if os.path.exists("token.json"):
        creds = Credentials.from_authorized_user_file("token.json", SCOPES)

    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                "credentials.json",
                SCOPES,
            )
            creds = flow.run_local_server(port=0)

        with open("token.json", "w", encoding="utf-8") as token:
            token.write(creds.to_json())

    return build("gmail", "v1", credentials=creds)


def send_message(to, subject, body):
    service = get_gmail_service()

    message = MIMEText(body, "plain", "utf-8")
    message["to"] = to
    message["from"] = "me"
    message["subject"] = subject

    raw = base64.urlsafe_b64encode(message.as_bytes()).decode("utf-8")

    sent = service.users().messages().send(
        userId="me",
        body={"raw": raw},
    ).execute()

    print("sent:", sent["id"])


def load_last_seen_id():
    if not os.path.exists(STATE_PATH):
        return 0

    with open(STATE_PATH, "r", encoding="utf-8") as f:
        state = json.load(f)

    return state.get("last_seen_id", 0)


def save_last_seen_id(last_seen_id):
    with open(STATE_PATH, "w", encoding="utf-8") as f:
        json.dump(
            {"last_seen_id": last_seen_id},
            f,
            ensure_ascii=False,
            indent=2,
        )


def get_new_requests():
    last_seen_id = load_last_seen_id()

    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row

    cur = conn.cursor()

    cur.execute(
        """
        SELECT rowid AS internal_id, *
        FROM requests
        WHERE rowid > ?
        ORDER BY rowid ASC
        """,
        (last_seen_id,),
    )

    rows = cur.fetchall()
    conn.close()

    return rows


def format_request_body(rows):
    lines = []
    lines.append("DNS申請フォームに新しい申請がありました。")
    lines.append("")
    lines.append(f"新規申請数: {len(rows)}")
    lines.append("")

    for row in rows:
        data = dict(row)

        lines.append("------------------------------")
        lines.append(f"申請ID: {data.get('internal_id')}")

        for key, value in data.items():
            if key == "internal_id":
                continue
            lines.append(f"{key}: {value}")

        lines.append("")

    return "\n".join(lines)


def main():
    rows = get_new_requests()

    if not rows:
        print("新しい申請はありません。")
        return

    body = format_request_body(rows)

    send_message(
        to=MAIL_TO,
        subject=MAIL_SUBJECT,
        body=body,
    )

    last_seen_id = rows[-1]["internal_id"]
    save_last_seen_id(last_seen_id)

    print(f"{len(rows)} 件の新しい申請を通知しました。")


if __name__ == "__main__":
    main()
```

`watchdog.py`: DBを監視し、アップデート(追加)があったら、呼び出すプログラム。
```
import json
import sqlite3
import subprocess
import sys
import time
from pathlib import Path


BASE_DIR = Path(__file__).resolve().parent

DB_PATH = BASE_DIR.parent / "data" / "requests.db"
STATE_PATH = BASE_DIR / "watchdog_state.json"
NOTIFIER_PATH = BASE_DIR / "notifier.py"

CHECK_INTERVAL_SECONDS = 60


def load_last_seen_id():
    if not STATE_PATH.exists():
        return 0

    with STATE_PATH.open("r", encoding="utf-8") as f:
        state = json.load(f)

    return state.get("last_seen_id", 0)


def save_last_seen_id(last_seen_id):
    with STATE_PATH.open("w", encoding="utf-8") as f:
        json.dump(
            {"last_seen_id": last_seen_id},
            f,
            ensure_ascii=False,
            indent=2,
        )


def get_latest_request_id():
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()

    cur.execute("SELECT MAX(rowid) FROM requests")
    latest_id = cur.fetchone()[0]

    conn.close()

    return latest_id or 0


def run_notifier():
    print("更新を検知しました。notifier.py を実行します。")

    result = subprocess.run(
        [sys.executable, str(NOTIFIER_PATH)],
        cwd=str(BASE_DIR),
        text=True,
        capture_output=True,
    )

    if result.stdout:
        print(result.stdout)

    if result.stderr:
        print(result.stderr)

    if result.returncode != 0:
        raise RuntimeError(f"notifier.py failed: exit code {result.returncode}")


def check_once():
    last_seen_id = load_last_seen_id()
    latest_id = get_latest_request_id()

    print(f"last_seen_id={last_seen_id}, latest_id={latest_id}")

    if latest_id > last_seen_id:
        run_notifier()
        save_last_seen_id(latest_id)
    else:
        pass 
        # print("更新はありません。")


def main():
    print("DB更新監視を開始します。")
    print(f"DB_PATH: {DB_PATH}")
    print(f"NOTIFIER_PATH: {NOTIFIER_PATH}")

    while True:
        try:
            check_once()
        except Exception as e:
            print("エラーが発生しました:", e)

        time.sleep(CHECK_INTERVAL_SECONDS)


if __name__ == "__main__":
    main()
```

