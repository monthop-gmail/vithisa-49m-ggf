# ระบบรวมข้อมูล 300 สาขา ผ่าน Google Form → PostgreSQL

## สถาปัตยกรรมที่แนะนำ

```
สาขา 1  ──┐
สาขา 2  ──┼──→  Google Form 1 ตัว  ──→  Google Sheet 1 ตัว  ──→  PostgreSQL
สาขา 3  ──┤                                                        │
...        │                                                        ▼
สาขา 300 ──┘                                                   Dashboard
```

- ใช้ Google Form **1 ตัว** ให้ทุกสาขากรอกที่เดียว
- ใน Form เพิ่ม dropdown **"ชื่อสาขา"** เพื่อแยกข้อมูล
- Response ไหลเข้า Google Sheet 1 ตัวอัตโนมัติ
- Python script (cron) ดึงข้อมูลจาก Sheet เข้า PostgreSQL วันละครั้ง
- Dashboard (Metabase / Grafana / Looker Studio) ดูข้อมูลแยกตามสาขา

---

## ทำไมไม่ใช้ 300 Forms / 300 Sheets

| | 300 Forms | 1 Form |
|---|---|---|
| **ดูแล** | แก้ 300 ที่ | แก้ที่เดียว |
| **ดึงข้อมูล** | เรียก API 300 ครั้ง | เรียกครั้งเดียว |
| **Rate limit** | เสี่ยงโดน | ไม่มีปัญหา |
| **Permission** | จัดการ 300 ชุด | จัดการชุดเดียว |
| **Sync → Postgres** | script ซับซ้อน | script ง่ายมาก |

> สาขาเห็นแค่หน้า Form ตอนกรอก ไม่เห็น Sheet response ของสาขาอื่น

---

## ขีดจำกัดของ Google Sheets

| รายการ | Limit |
|--------|-------|
| จำนวน cells ต่อ workbook | 10,000,000 cells |
| จำนวน columns สูงสุด | 18,278 columns |
| จำนวน rows สูงสุด | 10,000,000 rows (ถ้ามี 1 column) |

### ตัวอย่างจริง

| Columns | Rows ที่ได้สูงสุด |
|---------|-----------------|
| 5 | 2,000,000 |
| 10 | 1,000,000 |
| 20 | 500,000 |
| 50 | 200,000 |

> ในทางปฏิบัติ เกิน ~100,000 แถว Sheet จะเริ่มช้า/ค้าง
> ดังนั้น Google Sheet เหมาะเป็น **ทางเข้า** (input) แต่ไม่เหมาะเป็น **ที่เก็บ** (storage) ระยะยาว

---

## Database Schema

### ตารางหลัก

```sql
CREATE TABLE branch_submissions (
    id SERIAL PRIMARY KEY,
    branch_name VARCHAR(100) NOT NULL,
    report_date DATE NOT NULL,
    sale_amount NUMERIC(12,2),
    customer_count INTEGER,
    product_name VARCHAR(200),
    note TEXT,

    -- revision management
    version INT NOT NULL DEFAULT 1,
    status VARCHAR(20) DEFAULT 'active',  -- active, superseded
    submitted_at TIMESTAMP DEFAULT NOW(),
    submitted_by VARCHAR(100),

    -- correction tracking
    correction_reason TEXT,
    corrects_id INT REFERENCES branch_submissions(id)
);

CREATE INDEX idx_branch_date ON branch_submissions (branch_name, report_date);

CREATE UNIQUE INDEX idx_unique_entry
    ON branch_submissions (sync_date, branch_name, product_name)
    WHERE status = 'active';
```

### View ดึงข้อมูลล่าสุด

```sql
CREATE VIEW branch_data_latest AS
SELECT DISTINCT ON (branch_name, report_date, product_name)
    *
FROM branch_submissions
ORDER BY branch_name, report_date, product_name, version DESC;
```

---

## การแก้ไขข้อมูลย้อนหลัง

### Flow

```
วันที่ 1: สาขาส่ง Google Form ตามปกติ
          → INSERT version = 1, status = 'active'

วันที่ 3: สาขาแจ้งว่าข้อมูลผิด
          → สาขากรอก "แบบฟอร์มแก้ไข"
          → UPDATE ตัวเก่า status = 'superseded'
          → INSERT ตัวใหม่ version = 2, status = 'active'
                   + correction_reason + corrects_id
```

### แบบฟอร์มแก้ไข (สร้างอีก 1 Google Form)

```
แบบฟอร์มแก้ไขข้อมูล
├── ชื่อสาขา (dropdown)
├── วันที่ของข้อมูลที่ต้องการแก้ (date)
├── สาเหตุที่แก้ไข (text)
├── ข้อมูลที่ถูกต้อง:
│   ├── ยอดขาย
│   ├── จำนวนลูกค้า
│   └── ...
└── ผู้แจ้งแก้ไข (text)
```

### แนวทางเปรียบเทียบ

| | Versioning | Audit Log | Approval Workflow |
|---|---|---|---|
| **ความซับซ้อน** | ต่ำ | ปานกลาง | สูง |
| **ดูประวัติ** | ดูได้จากตารางเดียว | ต้อง join audit | ดูได้ครบ |
| **ควบคุม** | สาขาแก้ได้เอง | สาขาแก้ได้เอง | ต้องอนุมัติ |
| **เหมาะกับ** | ข้อมูลทั่วไป | การเงิน/compliance | ข้อมูลสำคัญมาก |

> แนะนำเริ่มจาก **Versioning** เพราะง่ายสุด และเพิ่ม approval ทีหลังได้

---

## Python Script: Sync Google Sheet → PostgreSQL

### ติดตั้ง Dependencies

```bash
pip install gspread google-auth psycopg2-binary
```

### Script หลัก

```python
import gspread
import psycopg2
from psycopg2.extras import execute_values
from google.oauth2.service_account import Credentials
from datetime import date
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# --- Config ---
SCOPES = ["https://www.googleapis.com/auth/spreadsheets.readonly"]
SERVICE_ACCOUNT_FILE = "service_account.json"

DB_CONFIG = {
    "host": "localhost",
    "port": 5432,
    "dbname": "branch_db",
    "user": "app_user",
    "password": "your_password",  # ควรใช้ env var
}

MASTER_SHEET_ID = "YOUR_MASTER_SHEET_ID"


def get_gspread_client():
    creds = Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES
    )
    return gspread.authorize(creds)


def read_sheet(gc, sheet_id):
    """อ่านข้อมูลจาก Google Sheet"""
    try:
        sh = gc.open_by_key(sheet_id)
        ws = sh.sheet1
        return ws.get_all_records()
    except Exception as e:
        logger.error(f"อ่าน Sheet ไม่ได้: {e}")
        return []


def save_to_postgres(rows):
    """เขียนข้อมูลลง PostgreSQL"""
    conn = psycopg2.connect(**DB_CONFIG)
    try:
        today = date.today()
        data = [
            (
                today,
                row.get("ชื่อสาขา", ""),
                row.get("ยอดขาย", 0),
                row.get("จำนวนลูกค้า", 0),
                row.get("สินค้า", ""),
                row.get("หมายเหตุ", ""),
            )
            for row in rows
        ]

        with conn.cursor() as cur:
            execute_values(
                cur,
                """
                INSERT INTO branch_submissions
                    (report_date, branch_name, sale_amount,
                     customer_count, product_name, note)
                VALUES %s
                ON CONFLICT (sync_date, branch_name, product_name)
                    WHERE status = 'active'
                DO UPDATE SET
                    sale_amount = EXCLUDED.sale_amount,
                    customer_count = EXCLUDED.customer_count,
                    note = EXCLUDED.note
                """,
                data,
                page_size=500,
            )
        conn.commit()
        logger.info(f"บันทึกสำเร็จ {len(data)} แถว")
    finally:
        conn.close()


def process_correction(conn, correction):
    """รับข้อมูลแก้ไขจาก Correction Form"""
    with conn.cursor() as cur:
        # หา record เดิม
        cur.execute("""
            SELECT id, version FROM branch_submissions
            WHERE branch_name = %s
              AND report_date = %s
              AND status = 'active'
            ORDER BY version DESC LIMIT 1
        """, (correction["branch_name"], correction["report_date"]))

        old = cur.fetchone()
        if not old:
            logger.warning(f"ไม่พบข้อมูลเดิมของ {correction['branch_name']}")
            return

        old_id, old_version = old

        # mark ตัวเก่าเป็น superseded
        cur.execute("""
            UPDATE branch_submissions
            SET status = 'superseded'
            WHERE id = %s
        """, (old_id,))

        # insert ตัวใหม่
        cur.execute("""
            INSERT INTO branch_submissions
                (branch_name, report_date, sale_amount,
                 customer_count, product_name,
                 version, status, submitted_by,
                 correction_reason, corrects_id)
            VALUES (%s, %s, %s, %s, %s, %s, 'active', %s, %s, %s)
        """, (
            correction["branch_name"],
            correction["report_date"],
            correction["sale_amount"],
            correction["customer_count"],
            correction["product_name"],
            old_version + 1,
            correction["submitted_by"],
            correction["reason"],
            old_id,
        ))
    conn.commit()


def main():
    gc = get_gspread_client()
    rows = read_sheet(gc, MASTER_SHEET_ID)
    if rows:
        save_to_postgres(rows)
    logger.info(f"เสร็จ: {len(rows)} แถว")


if __name__ == "__main__":
    main()
```

### ตั้ง Cron รันทุกวัน

```bash
# crontab -e
0 6 * * * cd /opt/scripts && /usr/bin/python3 sync_sheets.py >> /var/log/sync.log 2>&1
```

---

## Google Sheets API Rate Limits

| Quota | Limit |
|-------|-------|
| Read requests | 300/นาที per project |
| Per user | 60/นาที |

---

## Query ที่ใช้บ่อย

```sql
-- ข้อมูลล่าสุดทุกสาขา (ไม่สนว่าแก้กี่ครั้ง)
SELECT * FROM branch_data_latest
WHERE report_date = '2026-03-15';

-- ประวัติการแก้ไขของสาขาหนึ่ง
SELECT * FROM branch_submissions
WHERE branch_name = 'สาขาสยาม'
  AND report_date = '2026-03-15'
ORDER BY version;

-- สาขาไหนแก้ไขบ่อย
SELECT branch_name, COUNT(*) AS correction_count
FROM branch_submissions
WHERE status = 'superseded'
GROUP BY branch_name
ORDER BY correction_count DESC;

-- เทียบ before/after ของการแก้ไข
SELECT
    s1.branch_name,
    s1.sale_amount AS เดิม,
    s2.sale_amount AS แก้เป็น,
    s2.correction_reason
FROM branch_submissions s1
JOIN branch_submissions s2 ON s2.corrects_id = s1.id
WHERE s2.submitted_at >= NOW() - INTERVAL '7 days';
```

---

## ทางเลือก Database

| | Google Sheet รวม | PostgreSQL | BigQuery |
|---|---|---|---|
| **ค่าใช้จ่าย** | ฟรี | ฟรี-ถูก (self-host) | จ่ายตาม query |
| **รองรับข้อมูล** | ~5M cells | ไม่จำกัด | ไม่จำกัด |
| **Query** | ช้า, จำกัด | SQL เต็ม, เร็ว | SQL เต็ม, เร็วมาก |
| **Dashboard** | ทำเองใน Sheet | Metabase/Grafana | Looker Studio |
| **Setup** | ง่ายสุด | ปานกลาง | ต้อง GCP |

> ถ้าไม่อยาก self-host ใช้ **Supabase** (Postgres as a service) ได้ฟรี 500MB
