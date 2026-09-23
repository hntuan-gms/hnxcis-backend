# HNX-CIS Backend

API service của **Hệ thống Quản lý Niêm yết, Trái phiếu & Công bố thông tin HNX**.

- Express +  TypeScript, chạy trên **Cloud Run** (service `hnxcis-backend`)
- **Cloud SQL for PostgreSQL** qua Unix socket `/cloudsql/<INSTANCE_CONNECTION_NAME>`
- Drizzle ORM + drizzle-kit cho schema & migration
- Gemini API (`@google/genai`) cho các nghiệp vụ AI: FR-032, FR-063, FR-064 và FR-065
- Xác thực Firebase ID token (bật/tắt bằng `AUTH_REQUIRED`)

Frontend nằm ở repo riêng (`hnxcis-frontend`) và gọi service này qua HTTPS.

---

## 1. Cấu trúc
     
```
backend/
├── src/
│   ├── index.ts              # bootstrap: listen 0.0.0.0:$PORT, graceful shutdown
│   ├── app.ts                # express app: CORS, JSON, routes, error handler
│   ├── config/env.ts         # đọc & validate biến môi trường
│   ├── db/
│   │   ├── index.ts          # pool Postgres (Cloud SQL socket hoặc TCP) + getDb()
│   │   └── schema.ts         # Drizzle schema (users, organizations, securities, ...)
│   ├── lib/
│   │   ├── firebase-admin.ts # verify ID token bằng ADC
│   │   └── gemini.ts         # client Gemini + parse JSON an toàn
│   ├── middleware/           # auth (Firebase), errorHandler
│   ├── routes/               # health, gemini
│   └── types/hnx.ts          # domain types dùng chung với frontend
├── drizzle/                  # SQL migration sinh bởi drizzle-kit
├── drizzle.config.ts
├── Dockerfile                # multi-stage, chạy bằng user `node`, PORT=8080
└── .github/workflows/        # ci.yml, deploy-cloudrun.yml, db-migrate.yml
```

## 2. API

| Method | Endpoint                | Mô tả                                              |
| ------ | ----------------------- | -------------------------------------------------- |
| GET    | `/api/health`           | Liveness (không chạm DB) — dùng cho Cloud Run probe |
| GET    | `/api/health/db`        | Readiness: `SELECT 1` tới Cloud SQL                 |
| POST   | `/api/gemini/nl2query`  | FR-032 — Tra cứu báo cáo bằng ngôn ngữ tự nhiên     |
| POST   | `/api/gemini/datascan`  | FR-064 — Quét & trích xuất dữ liệu BCTC             |
| POST   | `/api/gemini/translate` | FR-065 — Dịch Việt → Anh theo glossary HNX          |
| POST   | `/api/gemini/chatbot`   | FR-063 — Chatbot FAQ HNX                            |

Khi `AUTH_REQUIRED=true`, mọi endpoint `/api/gemini/*` yêu cầu header
`Authorization: Bearer <Firebase ID token>`.

## 3. Chạy local

```bash
cp .env.example .env      # điền GEMINI_API_KEY, thông tin Cloud SQL
npm install
npm run dev               # http://localhost:8080
```

Kết nối Cloud SQL từ máy local qua Cloud SQL Auth Proxy:

```bash
cloud-sql-proxy --port 5432 "$PROJECT_ID:asia-southeast1:hnxcis-pg"
# .env: SQL_HOST=127.0.0.1, SQL_PORT=5432, INSTANCE_CONNECTION_NAME để trống
```

## 4. Biến môi trường

| Biến                       | Bắt buộc | Mô tả                                                              |
| -------------------------- | -------- | ------------------------------------------------------------------ |
| `PORT`                     | không    | Cloud Run tự inject (8080)                                         |
| `CORS_ORIGINS`             | nên có   | Danh sách origin được phép, phân tách bằng dấu phẩy                |
| `GEMINI_API_KEY`           | có       | Lấy từ Secret Manager (`hnxcis-gemini-api-key`)                     |
| `GEMINI_MODEL`             | không    | Mặc định `gemini-3.6-flash`                                        |
| `FIREBASE_PROJECT_ID`      | không    | Mặc định lấy `GOOGLE_CLOUD_PROJECT`                                |
| `AUTH_REQUIRED`            | không    | `true` → bắt buộc Firebase ID token                                |
| `INSTANCE_CONNECTION_NAME` | có (RUN) | `project:region:instance` → kết nối qua `/cloudsql/...`             |
| `SQL_HOST` / `SQL_PORT`    | local    | Dùng khi chạy qua Cloud SQL Auth Proxy                             |
| `SQL_DB_NAME`              | có       | Tên database (mặc định `hnxcis`)                                   |
| `SQL_USER` / `SQL_PASSWORD`| có       | User runtime (quyền hạn chế). Password lấy từ Secret Manager        |
| `SQL_ADMIN_USER` / `SQL_ADMIN_PASSWORD` | migrate | Chỉ dùng cho drizzle-kit, không dùng lúc chạy service |

## 5. Migration

```bash
npm run db:generate   # sinh SQL từ src/db/schema.ts vào ./drizzle
npm run db:migrate    # áp dụng migration (qua Cloud SQL Auth Proxy)
```

Trên CI: chạy thủ công workflow **Migrate Cloud SQL (Drizzle)** (`workflow_dispatch`).
Migration **không** chạy tự động khi deploy, để tránh sửa schema ngoài ý muốn.

### 5.1. Migration thủ công trong `drizzle/manual/`

RLS policy, hàm PL/pgSQL và trigger nằm ngoài khả năng biểu diễn của schema
Drizzle, nên chúng được viết tay trong `drizzle/manual/` và **drizzle-kit không
tự đọc thư mục này**. Chạy `db:generate` + `db:migrate` mà bỏ qua chúng sẽ cho
ra một database trông có vẻ đúng nhưng **không có phân quyền, không chặn xoá
cứng, audit sửa được** — nguy hiểm hơn là lỗi rõ ràng.

| File | Nội dung |
|---|---|
| `0001_v12_rls_and_audit_guard.sql` | RLS + `FORCE ROW LEVEL SECURITY` cho 30/30 bảng, chặn DELETE vật lý, audit append-only |
| `0002_v12_authz_bootstrap.sql` | Policy bootstrap danh tính theo `app.uid`, hàm `app_can_perform` (trục Chức năng × Trạng thái) |
| `0003_v12_versioning.sql` | `app_create_version` (version-on-approved-edit), `app_soft_delete` |

Thứ tự áp dụng, sau khi `db:generate` đã sinh migration schema:

```bash
# Tạo runtime role một lần cho mỗi database (chạy bằng SQL_ADMIN_USER):
#   CREATE ROLE hnxcis_app LOGIN PASSWORD '<SQL_PASSWORD>' NOBYPASSRLS;

npx drizzle-kit generate --custom --name=v12_rls_and_audit_guard
npx drizzle-kit generate --custom --name=v12_authz_bootstrap
npx drizzle-kit generate --custom --name=v12_versioning
# dán nội dung từng file trong drizzle/manual/ vào migration trống tương ứng
npm run db:migrate
```

**Runtime role `hnxcis_app` không được là chủ sở hữu bảng và không được có
`BYPASSRLS`.** PostgreSQL bỏ qua RLS với chủ sở hữu; `FORCE ROW LEVEL SECURITY`
bịt lỗ đó, nhưng `BYPASSRLS` thì không gì bịt được. Migration `0001` sẽ dừng và
báo lỗi nếu phát hiện role có thuộc tính này.

**Đừng áp dụng `0001` trước khi backend đã đặt session context.** Chưa có
`SET LOCAL app.*` (xem `src/db/session.ts`) thì mọi truy vấn trả về 0 dòng.

## 6. Deploy

Push lên nhánh `main` → workflow `.github/workflows/deploy-cloudrun.yml` sẽ:
build image → push Artifact Registry → `gcloud run deploy` (kèm `--add-cloudsql-instances`)
→ kiểm tra `/api/health`.

Cấu hình cần khai báo trong repo (Settings → Secrets and variables → Actions):

**Variables**

| Tên                     | Ví dụ                                    |
| ----------------------- | ---------------------------------------- |
| `GCP_PROJECT_ID`        | `dkquoc-sandbox-cob`                     |
| `GCP_REGION`            | `asia-southeast1`                        |
| `AR_REPOSITORY`         | `hnxcis`                                 |
| `CLOUD_RUN_SERVICE`     | `hnxcis-backend`                         |
| `CLOUD_RUN_RUNTIME_SA`  | `hnxcis-backend-sa@<project>.iam.gserviceaccount.com` |
| `CLOUD_SQL_INSTANCE`    | `dkquoc-sandbox-cob:asia-southeast1:hnxcis-pg` |
| `SQL_DB_NAME`           | `hnxcis`                                 |
| `SQL_USER`              | `hnxcis_app`                             |
| `CORS_ORIGINS`          | `https://hnxcis-frontend-xxxx.a.run.app` |
| `FIREBASE_PROJECT_ID`   | `dkquoc-sandbox-cob` (chỉ cần khi Firebase khác project Cloud Run) |
| `AUTH_REQUIRED`         | `false`                                  |

**Secrets**

| Tên                                | Mô tả                                                     |
| ---------------------------------- | --------------------------------------------------------- |
| `GCP_WORKLOAD_IDENTITY_PROVIDER`   | WIF provider (khuyến nghị, không cần key)                  |
| `GCP_DEPLOYER_SA`                  | Service account deploy dùng cùng WIF                       |
| `GCP_SA_KEY`                       | *Chỉ khi không dùng WIF* — JSON key của service account     |

Giá trị nhạy cảm (mật khẩu DB, Gemini API key) nằm ở **Secret Manager**, không đưa vào
GitHub Secrets — Cloud Run đọc trực tiếp qua `--set-secrets`.

Hướng dẫn tạo hạ tầng GCP (Artifact Registry, Cloud SQL, Secret Manager, WIF):
xem `docs/gcp-setup.md`.
