# Thiết kế hệ thống web quản lý bài giảng đa giáo viên

## 1) Sơ đồ kiến trúc hệ thống

```mermaid
flowchart TB
    subgraph FE[Frontend - Next.js/React]
      UI[Lecture UI\nUpload/Review/Admin]
      OAUTH[Google OAuth Login]
    end

    subgraph BE[Backend - NestJS (gợi ý)]
      APIGW[API Gateway\nREST/GraphQL]
      AUTH[Auth Service\nGoogle token verify\nRBAC]
      LEC[Lecture Service\nHierarchy\nSharing\nReview]
      FILE[File Service\nDrive/YouTube Upload]
      AUDIT[Audit Log Service]
      JOB[Queue Worker\nBullMQ]
    end

    subgraph DATA[Data Layer]
      PG[(PostgreSQL)]
      REDIS[(Redis)]
    end

    subgraph GOOGLE[Google Platform]
      GDRIVE[Google Drive API\nService Account / delegated account]
      YT[YouTube Data API\nFixed Channel OAuth Refresh Token]
    end

    USER[User/Admin\nGoogle Account] --> FE
    FE -->|OAuth code + API calls| APIGW

    APIGW --> AUTH
    APIGW --> LEC
    APIGW --> FILE
    LEC --> PG
    AUTH --> PG
    FILE --> PG
    AUDIT --> PG
    FILE -->|enqueue video upload| JOB
    JOB --> REDIS
    JOB --> YT
    FILE --> GDRIVE
    APIGW --> AUDIT
```

### Tách service rõ ràng

- **Auth service**:
  - Đăng nhập Google OAuth 2.0 (không local password).
  - Verify ID token, mapping user theo email/subject.
  - RBAC: `ADMIN`, `TEACHER`.
- **Lecture service**:
  - Quản lý khối → nhóm môn → bài giảng.
  - Sharing visibility (`PUBLIC`, `PRIVATE`) + ACL theo email.
  - Review: rating/comment.
- **File service**:
  - Nhận file từ frontend qua backend.
  - Tài liệu `.doc/.docx/.ppt/.pptx` upload Google Drive bằng tài khoản hệ thống.
  - Video upload YouTube về 1 channel cố định (token cấu hình sẵn).

---

## 2) Database schema chi tiết (PostgreSQL)

> Có thể dùng MongoDB, nhưng với yêu cầu quan hệ + ACL + review thì PostgreSQL dễ maintain hơn.

### 2.1 ERD logic

- users 1-n lectures (created_by / updated_by)
- lectures 1-n lecture_attachments
- lectures 1-n lecture_reviews
- lectures n-n users qua lecture_shares
- grade_levels 1-n subject_groups 1-n lectures

### 2.2 SQL schema mẫu

```sql
-- ENUMs
CREATE TYPE user_role AS ENUM ('ADMIN', 'TEACHER');
CREATE TYPE visibility AS ENUM ('PUBLIC', 'PRIVATE');
CREATE TYPE attachment_type AS ENUM ('DOCUMENT', 'VIDEO');
CREATE TYPE upload_status AS ENUM ('PENDING', 'PROCESSING', 'DONE', 'FAILED');

CREATE TABLE users (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  google_sub          TEXT UNIQUE NOT NULL,
  email               CITEXT UNIQUE NOT NULL,
  full_name           TEXT,
  avatar_url          TEXT,
  role                user_role NOT NULL DEFAULT 'TEACHER',
  is_active           BOOLEAN NOT NULL DEFAULT TRUE,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE grade_levels (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code                TEXT UNIQUE NOT NULL,   -- e.g. "10", "11", "12"
  name                TEXT NOT NULL,
  display_order       INT NOT NULL DEFAULT 0,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subject_groups (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  grade_level_id      UUID NOT NULL REFERENCES grade_levels(id) ON DELETE CASCADE,
  code                TEXT NOT NULL,
  name                TEXT NOT NULL,
  display_order       INT NOT NULL DEFAULT 0,
  UNIQUE (grade_level_id, code)
);

CREATE TABLE lectures (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  grade_level_id      UUID NOT NULL REFERENCES grade_levels(id),
  subject_group_id    UUID NOT NULL REFERENCES subject_groups(id),
  title               TEXT NOT NULL,
  description         TEXT,
  visibility          visibility NOT NULL DEFAULT 'PRIVATE',
  created_by          UUID NOT NULL REFERENCES users(id),
  updated_by          UUID NOT NULL REFERENCES users(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lectures_subject ON lectures(subject_group_id);
CREATE INDEX idx_lectures_visibility ON lectures(visibility);

CREATE TABLE lecture_shares (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lecture_id          UUID NOT NULL REFERENCES lectures(id) ON DELETE CASCADE,
  shared_email        CITEXT NOT NULL,
  granted_by          UUID NOT NULL REFERENCES users(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (lecture_id, shared_email)
);

CREATE TABLE lecture_attachments (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lecture_id          UUID NOT NULL REFERENCES lectures(id) ON DELETE CASCADE,
  type                attachment_type NOT NULL,
  original_filename   TEXT NOT NULL,
  mime_type           TEXT,
  file_size_bytes     BIGINT,

  -- Google Drive fields
  drive_file_id       TEXT,
  drive_web_view_link TEXT,
  drive_web_content_link TEXT,

  -- YouTube fields
  youtube_video_id    TEXT,
  youtube_url         TEXT,

  upload_status       upload_status NOT NULL DEFAULT 'PENDING',
  upload_error        TEXT,
  uploaded_by         UUID NOT NULL REFERENCES users(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

  CHECK (
    (type = 'DOCUMENT' AND drive_file_id IS NOT NULL)
    OR
    (type = 'VIDEO' AND youtube_video_id IS NOT NULL)
    OR
    (upload_status IN ('PENDING', 'PROCESSING', 'FAILED'))
  )
);

CREATE TABLE lecture_reviews (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lecture_id          UUID NOT NULL REFERENCES lectures(id) ON DELETE CASCADE,
  reviewer_id         UUID NOT NULL REFERENCES users(id),
  rating              SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  comment             TEXT,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (lecture_id, reviewer_id)
);

CREATE TABLE audit_logs (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_user_id       UUID REFERENCES users(id),
  actor_email         CITEXT,
  action              TEXT NOT NULL, -- LECTURE_CREATE, FILE_UPLOAD, REVIEW_UPDATE...
  entity_type         TEXT NOT NULL,
  entity_id           TEXT NOT NULL,
  metadata            JSONB,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at DESC);
```

---

## 3) Flow upload file và video

## 3.1 Upload tài liệu (.doc/.docx/.ppt/.pptx) -> Google Drive

1. Frontend gửi `multipart/form-data` đến `POST /lectures/:id/attachments/document`.
2. Backend kiểm tra:
   - user đã login bằng Google.
   - user có quyền `ADMIN/TEACHER` và có quyền sửa lecture.
   - validate mime type, max size.
3. File Service dùng **service account / delegated account cố định** upload vào Drive folder cấu hình sẵn.
4. Nhận `driveFileId`, `webViewLink`.
5. Lưu metadata vào `lecture_attachments`.
6. Ghi `audit_logs`.

## 3.2 Upload video -> YouTube

1. Frontend gửi file video đến `POST /lectures/:id/attachments/video`.
2. Backend tạo bản ghi `lecture_attachments` trạng thái `PENDING`.
3. Đẩy job vào queue (BullMQ): tránh timeout request dài.
4. Worker dùng **OAuth refresh token của channel cố định** upload YouTube.
5. Khi xong, update DB: `youtube_video_id`, `youtube_url`, `DONE`.
6. Nếu lỗi: `FAILED` + `upload_error`.
7. Frontend poll hoặc websocket để xem trạng thái.

> Lưu ý: YouTube API không hỗ trợ service account để upload channel như Drive. Cách chuẩn là OAuth consent một lần cho tài khoản channel, lưu refresh token ở backend (KMS/Secret Manager).

---

## 4) Code mẫu thực tế

## 4.1 Google OAuth login (NestJS + Passport)

```ts
// src/auth/google.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';
import { Injectable } from '@nestjs/common';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor() {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: process.env.GOOGLE_CALLBACK_URL!,
      scope: ['profile', 'email'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const email = profile.emails?.[0]?.value;
    const user = {
      googleSub: profile.id,
      email,
      fullName: profile.displayName,
      avatarUrl: profile.photos?.[0]?.value,
    };
    done(null, user);
  }
}
```

```ts
// src/auth/auth.controller.ts
import { Controller, Get, Req, Res, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
  @Get('google')
  @UseGuards(AuthGuard('google'))
  async googleLogin() {
    // redirect to Google
  }

  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleCallback(@Req() req, @Res() res) {
    // 1) upsert user by googleSub/email
    // 2) issue JWT access/refresh token for frontend
    // 3) redirect về FE with token/cookie
    return res.redirect(`${process.env.FRONTEND_URL}/auth/callback`);
  }
}
```

## 4.2 Upload tài liệu lên Google Drive bằng service account

```ts
// src/file/google-drive.service.ts
import { Injectable } from '@nestjs/common';
import { google } from 'googleapis';
import { Readable } from 'stream';

@Injectable()
export class GoogleDriveService {
  private drive;

  constructor() {
    const auth = new google.auth.GoogleAuth({
      credentials: {
        client_email: process.env.GDRIVE_SERVICE_ACCOUNT_EMAIL,
        private_key: process.env.GDRIVE_SERVICE_ACCOUNT_PRIVATE_KEY?.replace(/\\n/g, '\n'),
      },
      scopes: ['https://www.googleapis.com/auth/drive'],
    });

    this.drive = google.drive({ version: 'v3', auth });
  }

  async uploadDocument(params: {
    buffer: Buffer;
    filename: string;
    mimeType: string;
    parentFolderId: string;
  }) {
    const { buffer, filename, mimeType, parentFolderId } = params;

    const response = await this.drive.files.create({
      requestBody: {
        name: filename,
        parents: [parentFolderId],
      },
      media: {
        mimeType,
        body: Readable.from(buffer),
      },
      fields: 'id,name,webViewLink,webContentLink',
    });

    return {
      fileId: response.data.id,
      webViewLink: response.data.webViewLink,
      webContentLink: response.data.webContentLink,
    };
  }
}
```

## 4.3 Upload video lên YouTube API (channel cố định)

```ts
// src/file/youtube.service.ts
import { Injectable } from '@nestjs/common';
import { google } from 'googleapis';
import { Readable } from 'stream';

@Injectable()
export class YouTubeService {
  private youtube;

  constructor() {
    const oauth2Client = new google.auth.OAuth2(
      process.env.YT_CLIENT_ID,
      process.env.YT_CLIENT_SECRET,
      process.env.YT_REDIRECT_URI,
    );

    oauth2Client.setCredentials({
      refresh_token: process.env.YT_REFRESH_TOKEN,
    });

    this.youtube = google.youtube({
      version: 'v3',
      auth: oauth2Client,
    });
  }

  async uploadVideo(params: {
    buffer: Buffer;
    title: string;
    description?: string;
    privacyStatus?: 'private' | 'unlisted' | 'public';
  }) {
    const { buffer, title, description = '', privacyStatus = 'unlisted' } = params;

    const response = await this.youtube.videos.insert({
      part: ['snippet', 'status'],
      requestBody: {
        snippet: {
          title,
          description,
        },
        status: {
          privacyStatus,
          selfDeclaredMadeForKids: false,
        },
      },
      media: {
        body: Readable.from(buffer),
      },
    });

    const videoId = response.data.id!;
    return {
      videoId,
      url: `https://www.youtube.com/watch?v=${videoId}`,
    };
  }
}
```

---

## 5) Thiết kế API endpoints

## 5.1 Auth

- `GET /auth/google` -> redirect login Google
- `GET /auth/google/callback` -> xử lý callback, issue JWT
- `GET /auth/me` -> thông tin user hiện tại

## 5.2 Lecture hierarchy

- `GET /grade-levels`
- `POST /grade-levels` (ADMIN)
- `GET /grade-levels/:gradeId/subject-groups`
- `POST /subject-groups` (ADMIN)

## 5.3 Lectures

- `GET /lectures?gradeLevelId=&subjectGroupId=&q=&visibility=`
- `POST /lectures` (TEACHER/ADMIN)
- `GET /lectures/:id`
- `PATCH /lectures/:id`
- `DELETE /lectures/:id` (ADMIN hoặc owner)

## 5.4 Sharing (private lecture)

- `POST /lectures/:id/share`
  - body: `{ "emails": ["a@gmail.com", "b@gmail.com"] }`
- `DELETE /lectures/:id/share/:email`
- `GET /lectures/:id/share`

## 5.5 Attachments

- `POST /lectures/:id/attachments/document` (multipart)
- `POST /lectures/:id/attachments/video` (multipart, async)
- `GET /lectures/:id/attachments`
- `GET /attachments/:attachmentId/status`

## 5.6 Review

- `POST /lectures/:id/reviews`
- `PATCH /lectures/:id/reviews/:reviewId`
- `DELETE /lectures/:id/reviews/:reviewId`
- `GET /lectures/:id/reviews`

## 5.7 Authorization rule gợi ý

- Lecture `PUBLIC`: mọi user đăng nhập xem được.
- Lecture `PRIVATE`:
  - owner, ADMIN, hoặc email có trong `lecture_shares` mới xem được.
- Chỉ `ADMIN`/owner mới sửa sharing.

Pseudo-check:

```ts
function canViewLecture(user, lecture, shareEmails: string[]) {
  if (user.role === 'ADMIN') return true;
  if (lecture.visibility === 'PUBLIC') return true;
  if (lecture.createdBy === user.id) return true;
  return shareEmails.includes(user.email.toLowerCase());
}
```

---

## 6) Gợi ý deploy (Docker + Cloud)

## Option A: Docker Compose (MVP, VPS)

- Services:
  - `frontend` (Next.js)
  - `backend` (NestJS)
  - `postgres`
  - `redis`
  - `nginx` reverse proxy
- SSL qua Let's Encrypt (Caddy/Nginx certbot).
- Secrets inject qua `.env` + file permission chặt.

## Option B: Cloud production (khuyến nghị)

- **Frontend**: Vercel / Cloud Run static+SSR.
- **Backend API**: Cloud Run / GKE.
- **Database**: Cloud SQL PostgreSQL.
- **Queue**: Redis (Memorystore) + BullMQ.
- **Secrets**: Secret Manager (không hardcode key).
- **Storage external**: Google Drive + YouTube API.
- **Observability**: OpenTelemetry + Cloud Logging + alert.

## CI/CD gợi ý

1. PR -> chạy lint/test/build.
2. Merge main -> build Docker image.
3. Deploy rolling/canary.
4. Run migration bằng job riêng.

---

## 7) Bảo mật & best practices quan trọng

1. **Không bao giờ upload bằng token Google của user** cho Drive/YouTube.
2. Drive dùng service account hoặc delegated account trung tâm.
3. YouTube upload dùng OAuth refresh token của **channel hệ thống duy nhất**.
4. Refresh token và private key lưu trong Secret Manager/KMS.
5. Validate file type bằng cả extension + mime + magic bytes.
6. Giới hạn size, scan antivirus (ClamAV) nếu cần compliance.
7. Dùng signed JWT ngắn hạn + refresh token rotation.
8. Log đầy đủ `actor`, `action`, `entity` để audit.

---

## 8) Stack khuyến nghị chốt

- Frontend: Next.js + TanStack Query + NextAuth (Google provider, session nhẹ).
- Backend: NestJS + Prisma + PostgreSQL + Redis + BullMQ.
- APIs: REST (dễ tách service, dễ tích hợp upload multipart).
- Infra: Docker + Cloud Run + Cloud SQL + Secret Manager.

Thiết kế này đáp ứng đúng ràng buộc:
- Multi-teacher dùng đồng thời.
- Google login để nhận diện và audit.
- Upload luôn qua backend với tài khoản cấu hình sẵn.
- Phân quyền rõ (`ADMIN`, `TEACHER`) + chia sẻ private theo email.
