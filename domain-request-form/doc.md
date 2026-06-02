# 必要なこと

## Dockerの設定(Caddyの設定込み)


```Dockerfile
FROM node:22-alpine

WORKDIR /app
COPY package.json .
RUN npm install

COPY app.js .
CMD ["node", "app.js"]
```


## メール送付用のプログラム

`notifier.py`

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

