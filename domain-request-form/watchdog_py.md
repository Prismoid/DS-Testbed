
## Python スクリプトをバックグラウンドで実行する方法

## Python スクリプトをバックグラウンドで実行する方法

### 起動

ログを `watchdog.log` に保存し、PID を `watchdog.pid` に記録する。

```bash
nohup python3 watchdog.py > watchdog.log 2>&1 &
echo $! > watchdog.pid
```

### ログ確認

```bash
tail -f watchdog.log
```

### 実行中プロセスの確認

```bash
ps aux | grep watchdog.py
```

### 終了

```bash
kill $(cat watchdog.pid)
```

### 強制終了

通常の `kill` で止まらない場合のみ実行する。

```bash
kill -9 $(cat watchdog.pid)
```

### PID ファイルを使わずに終了する場合

```bash
pkill -f watchdog.py
```


## メール送付用のプログラム(`./watchdog`ディレクトリ内)

`credentials.json `: クライアントID、シークレットなどの情報
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

`watchdog.py`: DBを監視し、アップデート(追加)があったら、`notifier.py`を呼び出すプログラム。
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
