# Startup Checklist App — ILD Coffee Vietnam

Ứng dụng web 1 file (`index.html`) dùng để chấm checklist Shutdown/Start-up theo từng Đợt
(Tháng/Năm), theo dõi Highlight và lưu trữ/chỉnh sửa Lịch sử các phiên đã Submit. Không cần build,
không cần server — mở thẳng file HTML là chạy được (hoặc host tĩnh qua GitHub Pages).

## 1. Lưu trữ code lên GitHub

Repo đã được khởi tạo cục bộ. Để đẩy lên GitHub:

```bash
git remote add origin https://github.com/<tài-khoản-của-bạn>/<tên-repo>.git
git branch -M main
git push -u origin main
```

Sau khi push, có thể bật **GitHub Pages** (Settings → Pages → Deploy from branch → `main` /
`root`) để có 1 đường link truy cập app từ mọi thiết bị mà không cần mở file cục bộ.

## 2. Đăng nhập

Mở app sẽ luôn gặp màn hình đăng nhập trước — dùng **Firebase Authentication** (Email/Password)
thật của project `ild-startup-checklist`, không phải mật khẩu viết cứng trong code. Đăng nhập 1
lần trên máy nào thì máy đó tự nhớ (Firebase Auth tự lưu phiên đăng nhập trong trình duyệt), không
hỏi lại trừ khi đăng xuất hoặc xoá dữ liệu site của trang.

**Thêm/xoá/đổi mật khẩu người dùng** (không cần sửa code): vào
[Firebase Console](https://console.firebase.google.com/project/ild-startup-checklist/authentication/users)
→ Authentication → Users → **Add user** (hoặc bấm vào 1 user có sẵn để đổi mật khẩu/xoá).

> ⚠️ Lưu ý: đây là xác thực thật (server-side), khác với mật khẩu "1234" ở tab Chỉnh sửa/Cài đặt
> (vẫn giữ nguyên, chỉ là rào chắn nhẹ viết cứng trong code cho 2 tab đó).

## 3. Đồng bộ dữ liệu qua Firebase (Cloud)

Dữ liệu đồng bộ **qua Internet** dùng **Firebase Firestore**, ở tab **Đồng bộ** trong app, mục
"☁️ Đồng bộ Cloud (Firebase)" — đây là cách đồng bộ duy nhất giữa nhiều máy (đã bỏ cách đồng bộ qua
thư mục dùng chung OneDrive/Google Drive trước đây). Cấu hình Firebase đã **gắn sẵn trong code**
(project `ild-startup-checklist`) — dùng được ngay, không cần thiết lập gì thêm trên từng máy. Mục
"⚙️ Nâng cao" trong tab Đồng bộ chỉ dùng khi muốn trỏ sang 1 project Firebase khác.

**Hoàn toàn tự động, gần như realtime**: đăng nhập xong app tự tải dữ liệu mới nhất từ Firebase về (1
lần mỗi khi mở app). Từ đó, **mọi thay đổi dữ liệu đều tự đẩy lên Firebase** — không riêng gì nút
"Lưu"/"Submit": tick Đạt/Không đạt từng hạng mục, gõ ghi chú, thêm/sửa/xoá Highlight, sửa Cài đặt hay
danh mục Checklist... tất cả tự đẩy lên sau khi tạm ngừng thao tác ~2-3 giây (gộp các thay đổi liên
tiếp lại thành 1 lượt ghi thay vì đẩy từng phím gõ, tránh spam Firestore). Riêng "Lưu"/"Submit" và các
thao tác rõ ràng khác (thêm/xoá Highlight, sửa phiên Lịch sử) vẫn đẩy lên **ngay lập tức**, không chờ.
App cũng cố gắng đẩy nốt phần đang chờ ngay khi bạn chuyển tab/đóng trình duyệt, để giảm tối đa rủi ro
mất dữ liệu nếu tắt máy đột ngột. 2 nút **"⬆️ Đẩy lên"** / **"⬇️ Tải về"** ở tab Đồng bộ chỉ còn dùng
khi muốn đồng bộ lại thủ công. Việc gộp dữ liệu dùng cơ chế "gộp an toàn, không mất dữ liệu" (không ghi
đè nhầm, không hồi sinh dữ liệu đã xoá).

**Phạm vi đồng bộ**: Lịch sử phiên đã Submit (kể cả khi sửa lại), Danh mục Checklist/Cài đặt đã
chỉnh sửa, Nhật ký hoạt động (Data Log), Highlight, phiên đang chấm dở — toàn bộ dữ liệu dạng chữ/số.
**Không đồng bộ**: ảnh bằng chứng đính kèm trực tiếp (base64) — bị loại bỏ trước khi đẩy lên để
tránh tốn dung lượng/chi phí Firestore (giới hạn 1MB/document). Ảnh chỉ lưu trên ổ đĩa/thư mục đã
chọn ở tab **Cài đặt** ("Thư mục lưu ảnh bằng chứng") — nếu muốn ảnh cũng đồng bộ được giữa nhiều
máy, tự chọn 1 thư mục OneDrive/Google Drive dùng chung khi kết nối thư mục này (tính năng riêng,
độc lập với Firebase).

## 4. Bảo mật dữ liệu (Firestore Rules)

Firestore Rules yêu cầu **đã đăng nhập** (`request.auth != null`) mới được đọc/ghi document đồng bộ
`checklist_sync/ild_startup_checklist_shared` — xem/sửa tại
[Firebase Console → Firestore → Rules](https://console.firebase.google.com/project/ild-startup-checklist/firestore/databases/-default-/rules):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /checklist_sync/{docId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

apiKey trong `firebaseConfig` (gắn sẵn trong `index.html`) **không phải bí mật** — đây là thiết kế
của Firebase, apiKey chỉ định danh project, không dùng để xác thực. Lớp bảo vệ dữ liệu thật nằm ở
Rules (yêu cầu đăng nhập) + Firebase Authentication (Mục 2), không nằm ở việc giấu config.

## 5. Cấu trúc dữ liệu trên Firestore

Toàn bộ dữ liệu được lưu trong **1 document duy nhất**:
`checklist_sync/ild_startup_checklist_shared`, dạng:

```json
{
  "data": { "history": [...], "items": {...}, "settings": {...}, "logs": [...], "...": "..." },
  "updatedAt": "<server timestamp>",
  "updatedBy": "<device id>"
}
```

Đây là thiết kế "1 bản ghi JSON dùng chung", phù hợp vì checklist chỉ dùng cho 1 nhà máy/1 bộ dữ
liệu chung — không cần thiết kế bảng quan hệ phức tạp.
