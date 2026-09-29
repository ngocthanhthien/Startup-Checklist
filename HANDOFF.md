# HANDOFF — Startup Checklist App (ILD Coffee Vietnam)

Tài liệu này tổng hợp toàn bộ ngữ cảnh dự án để một AI/dev khác có thể tiếp tục chỉnh sửa
`index.html` mà không cần dò lại từ đầu. Đọc file này trước, sau đó đọc thẳng `index.html`
(1 file duy nhất, ~4000 dòng, không build) khi cần chi tiết code.

## 1. Dự án là gì

Web app 1 file (`index.html`) dùng nội bộ tại nhà máy ILD Coffee Vietnam để chấm checklist
Shutdown/Start-up hàng tháng, theo dõi Highlight (việc cần chú ý), và lưu trữ/chỉnh sửa Lịch sử
các phiên đã Submit. Không có build step — mở thẳng file hoặc host tĩnh.

**Deploy hiện tại**: GitHub Pages — `https://ngocthanhthien.github.io/Startup-Checklist/`.
Repo git cục bộ tại thư mục này **chưa gắn remote** (`git remote -v` trống) — người dùng tự
đẩy/tải file lên GitHub bằng cách riêng của họ (không qua `git push` từ máy đang code), nên
**đừng cho rằng commit ở đây tự động lên production** — luôn nhắc người dùng tự deploy sau khi
sửa xong, và khi debug lỗi trên bản live, nhớ khả năng họ đang xem bản cache/cũ.

## 2. Firebase (đã cấu hình sẵn, dùng chung)

- Project: **`ild-startup-checklist`** (Google Cloud project cùng tên).
- `firebaseConfig` **gắn cứng trong code** (biến `FIREBASE_CONFIG_BUILTIN`, gần đầu file, cạnh
  `LS_FIREBASE_CFG`) — apiKey của Firebase Web App **không phải bí mật** (thiết kế của Firebase),
  không cần giấu.
- **Firestore**: 1 document duy nhất `checklist_sync/ild_startup_checklist_shared` chứa toàn bộ
  dữ liệu dạng `{ data: {...snapshotData()...}, updatedAt, updatedBy }`. Không dùng bảng quan hệ.
  Rules yêu cầu đã đăng nhập: `allow read, write: if request.auth != null;` (xem Firebase Console
  → Firestore → Rules). Authorized domain đã thêm: `ngocthanhthien.github.io` (Authentication →
  Settings → Authorized domains) — nếu đổi domain deploy, phải thêm domain mới vào đây.
- **Authentication**: Email/Password, KHÔNG phải mật khẩu viết cứng. Tài khoản hiện có:
  `qaline@ild-coffee.com` (mật khẩu do người dùng tự đặt, tôi không lưu). Thêm/xoá/đổi mật khẩu
  user: Firebase Console → Authentication → Users → Add user (không cần sửa code).
- **API key restrictions**: key đang giới hạn theo danh sách "25 APIs" (Google Cloud Console →
  APIs & Services → Credentials) — đã xác nhận **Identity Toolkit API** và **Token Service API**
  nằm trong danh sách cho phép. Nếu sau này gặp lỗi `auth/api-key-not-valid`, đây là chỗ đầu tiên
  cần kiểm tra (dù lần gặp lỗi thực tế trước đó hoá ra là do cache trình duyệt cũ, không phải do
  key — xem mục 6).

## 3. Xác thực / bảo mật trong app (2 lớp riêng biệt, đừng nhầm)

1. **Màn hình đăng nhập toàn app** (`#loginGate`, hiện ngay khi mở trang) — dùng Firebase
   Authentication thật (mục 2). Code: tìm `initLoginGate` (IIFE gần đầu script). Đăng nhập xong,
   Firebase Auth tự nhớ phiên (persistence mặc định), không dùng localStorage tự chế nữa.
2. **Mật khẩu "1234"** (biến `APP_EDIT_PASSWORD`) — chỉ là rào chắn nhẹ viết cứng trong code, gọi
   qua `requirePassword(actionLabel)`, dùng cho: vào tab **Chỉnh sửa**, vào tab **Cài đặt**, xoá
   hoặc sửa 1 phiên trong **Lịch sử**. KHÔNG phải bảo mật thật — ai xem source đều thấy. Giữ
   nguyên theo yêu cầu người dùng trước đây (đã hỏi, họ chọn giữ).

## 4. Cấu trúc tab (thứ tự trong thanh tab)

`Checklist | Highlight | Evidence | Chỉnh sửa | Lịch sử | Truy xuất | Đồng bộ | Data Log | Cài đặt | Hướng dẫn`

Tab **"Plan"** (Kế hoạch Cleaning) đã bị **xoá hẳn** theo yêu cầu — nếu thấy code cũ/tài liệu cũ
nhắc tới nó thì đó là tàn dư, bỏ qua.

Mỗi tab có `render<TenTab>()` riêng, gọi từ `switchTab(name)` (cuối file). Tab bị gate mật khẩu:
`edit`, `settings`. Các tab khác mở tự do.

## 5. Data model

### localStorage keys chính (prefix `ild_startup_checklist_`)
`draft_v2` (STATE đang nhập), `history_v2` (mảng phiên đã Submit), `items_v2` (override danh mục
Checklist), `settings_v1`, `log_v1` (Data Log), `highlight_v1`, `device_id_v1`, `evidence_tree_v1`
(+ `_ts`), `deleted_ids_v1` (tombstone), `last_sync_v1`, `firebase_cfg_v1` (override Firebase
config nâng cao, riêng từng máy — KHÔNG đồng bộ, xem mục 7).

> Nếu thấy key `..._login_ok_v1` trong localStorage của 1 máy nào đó: đó là tàn dư của cơ chế đăng
> nhập cũ (user/pass viết cứng, trước khi đổi sang Firebase Auth thật ở mục 3) — code hiện tại
> **không còn khai báo hay đọc/ghi key này ở đâu nữa**, có thể bỏ qua hoặc xoá thủ công, không ảnh
> hưởng gì.

### STATE (phiên Checklist đang nhập)
```js
STATE = {
  header: { dateStart: "YYYY-MM-DD", dateEnd: "YYYY-MM-DD", period: "YYYY-MM", note: "" },
  items: { [itemId]: { status: "pending"|"pass"|"fail"|"na", by: "", note: "", photos: [] } },
  updatedAt: <ms epoch>
}
```
`period` (Đợt Tháng/Năm) là trường **mới thêm** — nhập ở đầu tab Checklist (`<input type="month"
id="hPeriod">`), dùng làm nguồn xác định "đợt" cho tên thư mục ảnh Evidence
(`activePeriodFromHeader()` ưu tiên trường này, chỉ suy từ `dateStart` nếu chưa nhập).

### 1 phiên trong History (`rec`)
```js
{ id: "S...", savedAt: "<ISO>", header: {...}, items: {...}, summary: {...}, updatedAt: <ms epoch> }
```
`updatedAt` dùng để merge "mới nhất thắng" khi đồng bộ (xem mục 8) — luôn set lại khi sửa phiên.

### `computeSummaryFor(header, itemsState)` — **quan trọng, mới đổi công thức**
```js
pct = Math.round((pass + na) / total * 100)   // "Không đạt" (fail) KHÔNG được tính là đã xong
```
Trước đây `done = pass+fail+na` (fail tính là đã chấm). Đã đổi theo yêu cầu người dùng: **1 hạng
mục "Không đạt" coi như CHƯA hoàn thành** cho tới khi sửa lại thành Đạt (sau khắc phục) hoặc N/A.
Hệ quả: **Submit bị chặn vĩnh viễn nếu còn bất kỳ hạng mục nào đang ở trạng thái Fail**, kể cả khi
mọi hạng mục khác đã chấm xong. Đây là quyết định có chủ đích, đã hỏi lại người dùng và xác nhận
2 lần trước khi đổi — **không tự ý đổi lại nếu chưa hỏi**. `groupCompletion()` (badge %/tự thu gọn
từng Công đoạn) và `computeStageProgress()`/`currentNextStage()` (Công đoạn hiện tại/tiếp theo)
đều dùng chung logic này (cùng coi fail = chưa xong) — sửa 1 chỗ, các chỗ khác tự nhất quán theo.

`pendingBlockReason(summary)` — hàm dùng chung để mô tả lý do chưa Submit được (tách rõ "còn X
chưa chấm" và "còn Y Không đạt cần xử lý lại"), dùng cho cả hint dưới Checklist và toast khi Submit.

### Thứ tự hạng mục Checklist
`allItems()` trả về `ITEMS_OVERRIDE.items || DEFAULT_ITEMS` (mảng phẳng). `groupByStage(items)`
nhóm theo `stage` rồi sort theo `STAGE_CODES`. **Thứ tự hiển thị trong từng nhóm = thứ tự phần tử
trong mảng** — tab Chỉnh sửa có nút ⬆️/⬇️ (`moveItemInStage(id, dir)`) hoán đổi vị trí trong cùng
Công đoạn rồi **dựng lại toàn bộ mảng** theo đúng thứ tự nhóm hiện tại, nên tab Checklist tự động
hiển thị đúng thứ tự đã sắp ở tab Chỉnh sửa (cùng dùng `groupByStage(allItems())`).

## 6. Vấn đề đã gặp và cách đã xử lý (đọc để tránh lặp lại)

- **Lỗi `auth/api-key-not-valid` khi đăng nhập**: hoá ra không phải do key/quyền — là do **cache
  trình duyệt cũ** trên máy người dùng (đổi trình duyệt khác thì đăng nhập được ngay). Nếu gặp lại
  báo lỗi đăng nhập, **luôn nghi ngờ cache trước** (bảo hard-refresh `Ctrl+Shift+R` hoặc xoá site
  data), đừng vội sửa code/Firebase config.
- **Nút "Xem"/"Nạp lại"/"Sửa" trong Lịch sử vô hình**: do **trùng tên class CSS `view`** với class
  `.view{display:none}` dùng cho các panel tab (`#view-check` v.v.) — bug có từ trước, không phải
  do code mới. Đã đổi tên class nút thành `viewbtn`. Nếu thêm nút mới vào bảng Lịch sử/Truy xuất,
  **không đặt class `view`** cho bất kỳ phần tử nào ngoài các div `<div class="view" id="view-...">`
  của hệ thống tab.
- **Comment HTML đóng sai kiểu `*/` thay vì `-->`**: từng làm trình duyệt "nuốt" cả 1 đoạn lớn
  phía sau vào trong comment (mất cả `<div class="app">`...). Luôn kiểm tra kỹ khi thêm
  `<!-- ... -->` thủ công, nhất là các đoạn dài nhiều dòng.
- **Google API key restriction**: khi cần thêm 1 API mới vào danh sách cho phép của key (Google
  Cloud Console), dropdown "Select API restrictions" dùng ô lọc — gõ tên API để tìm, đừng cuộn tay
  qua danh sách dài (dễ bỏ sót do virtualized list).
- **Auto-mode classifier chặn 1 số thao tác qua trình duyệt**: cụ thể là gõ trực tiếp Firestore
  Rules dạng `allow read, write: if true` (bị chặn vì "làm yếu bảo mật") và **nhập mật khẩu vào ô
  password** (Firebase Console "Add user") — 2 việc này luôn phải nhờ người dùng tự gõ, không thử
  gõ hộ qua automation.

## 7. Đồng bộ dữ liệu (Firebase Cloud — cơ chế "gần như realtime")

- Toàn bộ dữ liệu (trừ ảnh bằng chứng) tự động đẩy lên Firebase, **không cần bấm nút**:
  - `scheduleCloudPush(reasonLabel)` — đẩy **hoãn 2.5 giây** (debounce, gộp các thay đổi liên
    tiếp), gọi từ mọi hàm lưu dữ liệu gốc: `saveDraft`, `saveItemsOverride`, `saveSettings`,
    `saveEvidenceTreeOverride`, `setHighlights`, `setHistory`.
  - `backgroundCloudSyncIfConfigured(reasonLabel)` — đẩy **ngay lập tức**, dùng cho thao tác rõ
    ràng (Lưu, Submit, thêm/xoá Highlight, Cập nhật phiên Lịch sử...). Tự huỷ lịch đẩy-hoãn đang
    chờ (nếu có) để tránh đẩy trùng.
  - Khi tab bị ẩn (`visibilitychange` → `hidden`), nếu đang có lịch đẩy-hoãn thì đẩy ngay (best
    effort chống mất dữ liệu khi đóng trình duyệt đột ngột).
  - `__applyingRemoteMerge` (biến cờ) — bật khi đang áp dụng dữ liệu vừa TẢI VỀ
    (`pullFromFirebaseCloud` → `mergeRemoteData`), để `scheduleCloudPush` bỏ qua, tránh đẩy ngược
    lại đúng dữ liệu vừa nhận.
- **Ảnh bằng chứng KHÔNG đồng bộ lên Firebase** (giới hạn 1MB/document của Firestore) —
  `sanitizeForCloudSync()` loại bỏ ảnh nhúng base64 trước khi đẩy. Ảnh chỉ lưu trên
  ổ đĩa/thư mục đã chọn ở tab Cài đặt (`evidenceDirHandle`, File System Access API — tính năng
  **độc lập**, không liên quan Firebase, người dùng có thể tự chọn 1 thư mục OneDrive/Google Drive
  dùng chung nếu muốn ảnh cũng đồng bộ được).
- `mergeRemoteData(d)` — merge "mới nhất thắng" theo `updatedAt` cho: history, highlight,
  items override, settings, evidenceTree. Có cơ chế tombstone (`deletedIds`) để không "hồi sinh"
  bản ghi đã xoá khi merge. **Nếu thêm loại dữ liệu mới cần đồng bộ, phải thêm cả vào
  `snapshotData()` (đẩy lên) VÀ `mergeRemoteData()` (nhận về) — thiếu 1 trong 2 là dữ liệu không
  qua được giữa các máy.**
- Không còn tính năng đồng bộ qua thư mục dùng chung (OneDrive/Google Drive) — đã xoá hoàn toàn
  theo yêu cầu, chỉ còn Firebase (tự động) + JSON thủ công (nút trong tab Đồng bộ, để backup
  ngoại tuyến/chuyển máy thủ công).

## 8. Việc đã làm theo thứ tự thời gian (tóm tắt `git log`, mới nhất ở cuối)

1. Thêm Firebase Cloud sync (Firestore) làm phương án thay thế đồng bộ qua thư mục dùng chung.
2. Đổi tên file `Startup Checklist App.html` → `index.html` (URL GitHub Pages gọn hơn).
3. Gắn cứng `firebaseConfig` mặc định trong code + màn đăng nhập app (ban đầu là user/pass viết
   cứng "QAILD").
4. Đổi màn đăng nhập sang Firebase Authentication thật (Email/Password) — theo yêu cầu người dùng
   muốn tự quản lý user qua Firebase Console.
5. Tự động tải dữ liệu về khi đăng nhập xong (không cần bấm "Tải về" thủ công nữa).
6. **Xoá hẳn tính năng đồng bộ qua thư mục dùng chung** (OneDrive/Google Drive) — chỉ giữ Firebase
   + JSON thủ công.
7. Thêm trường "Đợt (Tháng/Năm)" ở đầu Checklist; Submit tự reset phiên mới (bỏ hộp thoại xác
   nhận); cho phép **Sửa trực tiếp 1 phiên trong Lịch sử** (không tạo bản trùng); xoá tab Plan.
8. Sửa lỗi nút Xem/Sửa/Nạp lại vô hình (class CSS trùng tên — mục 6).
9. Mặc định bật "Ẩn các mục đã hoàn thành" ở tab Highlight.
10. Mọi thay đổi dữ liệu tự đẩy lên Firebase (không chỉ khi bấm Lưu/Submit) — cơ chế debounce
    (mục 7).
11. Cho phép đổi vị trí câu hỏi trong tab Chỉnh sửa (áp dụng luôn cho Checklist); sửa Due date
    Highlight ngay trên từng dòng; **đổi công thức % — "Không đạt" không tính là đã hoàn thành**
    (mục 5).

## 9. Chạy thử cục bộ

Không cần build. Có sẵn `.claude/launch.json` (cấu hình cho Claude Code's preview) chạy
`python -m http.server 8723`. Cách khác: bất kỳ static server nào cũng được, ví dụ
`npx serve .` hoặc mở thẳng file (một số tính năng như Firebase Auth/Firestore vẫn hoạt động qua
`file://` nhưng File System Access API cho thư mục ảnh cần origin http(s), nên ưu tiên chạy qua
server cục bộ khi test).

## 10. Việc chưa làm / có thể cân nhắc sau (không phải yêu cầu, chỉ ghi chú)

- Chưa có cách "quên mật khẩu" thật cho tài khoản Firebase Auth nếu domain email không nhận được
  mail (domain `ild-coffee.com` hiện là giả) — muốn đổi mật khẩu phải xoá + tạo lại user trong
  Firebase Console (mất lịch sử đăng nhập của user đó, không mất dữ liệu app).
- README.md (cùng thư mục) có hướng dẫn setup Firebase từ đầu, chi tiết hơn HANDOFF này — đọc
  thêm nếu cần thiết lập lại project Firebase từ số 0.
