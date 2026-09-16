# Startup Checklist App — ILD Coffee Vietnam

Ứng dụng web 1 file (`index.html`) dùng để chấm checklist Shutdown/Start-up,
theo dõi Highlight, Kế hoạch vệ sinh và lưu trữ Lịch sử các phiên đã Submit. Không cần build,
không cần server — mở thẳng file HTML là chạy được (hoặc host tĩnh qua GitHub Pages).

## 1. Lưu trữ code lên GitHub

Repo đã được khởi tạo cục bộ (`git init` + commit đầu tiên). Để đẩy lên GitHub:

```bash
git remote add origin https://github.com/<tài-khoản-của-bạn>/<tên-repo>.git
git branch -M main
git push -u origin main
```

Sau khi push, có thể bật **GitHub Pages** (Settings → Pages → Deploy from branch → `main` /
`root`) để có 1 đường link truy cập app từ mọi thiết bị mà không cần mở file cục bộ.

> Lưu ý: file HTML publish lên GitHub Pages là **công khai** với bất kỳ ai có link. App không có
> cơ chế đăng nhập, và cấu hình Firebase (bước dưới) không nằm trong file này — mỗi máy tự nhập
> cấu hình Firebase riêng, lưu trong `localStorage` của trình duyệt đó, không đẩy lên GitHub.

## 2. Đồng bộ dữ liệu qua Firebase (Cloud)

Trước đây app chỉ đồng bộ giữa các máy qua 1 thư mục dùng chung (OneDrive/Google Drive). Bản này
bổ sung thêm 1 cách đồng bộ **qua Internet** dùng **Firebase Firestore**, ở tab **Đồng bộ** trong
app, mục "☁️ Đồng bộ Cloud (Firebase)".

### 2.1. Tạo project Firebase (miễn phí)

1. Vào https://console.firebase.google.com → **Add project** → đặt tên (vd `ild-startup-checklist`) → tạo project (không cần bật Google Analytics).
2. Trong project, vào **Build → Firestore Database → Create database** → chọn chế độ **Production mode** → chọn khu vực gần nhất (vd `asia-southeast1`) → Enable.
3. Vào **Project settings** (biểu tượng bánh răng) → tab **General** → mục **Your apps** → bấm biểu tượng **Web `</>`** → đặt tên app bất kỳ → **Register app**.
4. Firebase sẽ hiện đoạn cấu hình dạng:
   ```js
   const firebaseConfig = {
     apiKey: "...",
     authDomain: "...",
     projectId: "...",
     storageBucket: "...",
     messagingSenderId: "...",
     appId: "..."
   };
   ```
   Copy **toàn bộ object `{ ... }` này** (chỉ phần trong dấu ngoặc nhọn, dạng JSON).

### 2.2. Thiết lập quyền truy cập (Firestore Rules)

Vào **Firestore Database → Rules**, dán đúng luật sau rồi **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /checklist_sync/{docId} {
      allow read, write: if true;
    }
  }
}
```

> ⚠️ **Lưu ý bảo mật**: luật trên cho phép **bất kỳ ai có `firebaseConfig`** (vốn nằm trong
> mã nguồn/`localStorage` — không phải bí mật tuyệt đối) đọc/ghi được document đồng bộ này. Phù hợp
> cho công cụ nội bộ, ít người biết link. Nếu cần chặt chẽ hơn, có thể bật **Firebase
> Authentication** (Email/Password hoặc Google) và đổi luật thành
> `allow read, write: if request.auth != null;` — phần này ngoài phạm vi thiết lập hiện tại,
> nói với Claude nếu muốn bổ sung sau.

### 2.3. Kết nối app với Firebase

1. Mở app (`index.html` hoặc link GitHub Pages) → tab **Đồng bộ**.
2. Ở mục "☁️ Đồng bộ Cloud (Firebase)", dán nguyên object `firebaseConfig` đã copy ở bước 2.1 vào ô **firebaseConfig (dạng JSON)**.
3. Bấm **💾 Lưu cấu hình**.
4. Bấm **⬆️ Đẩy lên Firebase** để đẩy dữ liệu hiện có trên máy này lên Cloud lần đầu.
5. Trên các máy khác: lặp lại bước 1–3 (dán **đúng cùng 1 firebaseConfig**), rồi bấm **⬇️ Tải về từ Firebase** để nhận dữ liệu.

Từ đó, nút **"Lưu"** và **"Submit"** ở tab Checklist sẽ tự động đẩy dữ liệu lên Firebase (âm thầm,
không cần bấm gì thêm) nếu máy đó đã cấu hình. Khi mở app trên máy khác, bấm **"Tải về từ
Firebase"** để nhận bản mới nhất — việc gộp dữ liệu dùng lại đúng cơ chế "gộp an toàn, không mất
dữ liệu" đã có sẵn của tính năng đồng bộ thư mục (không ghi đè nhầm, không hồi sinh dữ liệu đã xoá).

### 2.4. Phạm vi dữ liệu đồng bộ lên Firebase

Đồng bộ lên Firebase bao gồm: **Lịch sử phiên đã Submit**, **Danh mục Checklist/Cài đặt đã chỉnh
sửa**, **Nhật ký hoạt động (Data Log)**, Highlight, Kế hoạch vệ sinh, và phiên đang chấm dở — tức
là **toàn bộ dữ liệu dạng chữ/số**, giống hệt phạm vi của tính năng đồng bộ thư mục hiện có.

**Không đồng bộ**: ảnh bằng chứng (Evidence) đính kèm trực tiếp trong checklist — các ảnh này bị
loại bỏ trước khi đẩy lên để tránh tốn dung lượng/chi phí Firestore (giới hạn 1MB/document). Ảnh
vẫn đồng bộ được như cũ qua thư mục dùng chung (OneDrive/Google Drive).

## 3. Cấu trúc dữ liệu trên Firestore

Toàn bộ dữ liệu được lưu trong **1 document duy nhất**:
`checklist_sync/ild_startup_checklist_shared`, dạng:

```json
{
  "data": { "history": [...], "items": {...}, "settings": {...}, "logs": [...], "...": "..." },
  "updatedAt": "<server timestamp>",
  "updatedBy": "<device id>"
}
```

Đây là thiết kế "1 bản ghi JSON dùng chung" (giống file JSON đồng bộ qua thư mục trước đây), phù
hợp vì checklist chỉ dùng cho 1 nhà máy/1 bộ dữ liệu chung — không cần thiết kế bảng quan hệ phức
tạp.
