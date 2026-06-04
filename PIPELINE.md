# Pipeline: Thêm project mới vào dataset (Martins + CK format)

Mục tiêu: từ raw data của một project mới, tạo ra một file CSV khớp hoàn toàn
với format của `ALL_merged.csv` (96 cột, file trước đó).

---

## Hiện trạng raw data

| File | Nội dung | Trạng thái |
|---|---|---|
| `<project>.csv` | Metadata + 22 test smells + 13 churn metrics | ✅ đã có |
| `<project>/class.csv` | CK metrics cấp class + loc, wmc, rfc | ✅ đã có |
| `<project>/method.csv` | CK metrics cấp method | Không cần dùng |

---

## Các bước cần thực hiện

### Bước 1: Join class.csv vào project.csv

**Join key:** `Tag` (khớp chính xác) + `TestFilePath` ↔ `file`

**Lưu ý:**
- `class.csv.file` là absolute path local, `project.csv.TestFilePath` là relative path. Cần trích suffix từ `src/` và chuẩn hóa separator (`\` → `/`) trước khi join.
- `class.csv` chứa cả production class, anonymous class, inner class. Chỉ giữ các row có `type == 'class'` và khớp với TestFilePath, tránh join nhiều-một.
- `NOM` trong `ALL_merged.csv` thực chất là `totalMethodsQty` từ CK tool, đổi tên khi join.
- `LOC` ← `loc`, `WMC` ← `wmc`, `RFC` ← `rfc` — đổi tên cho đúng convention.

**Cột lấy từ class.csv:** `file`, `class`, `type`, `loc`→LOC, `wmc`→WMC, `rfc`→RFC, `totalMethodsQty`→NOM, và toàn bộ 46 CK metrics còn lại.

### Bước 2: Tính NAs và AsD

**NAs** = số lượng assertion calls trong test file tại đúng version (tag).
Đếm trực tiếp từ source code đã staged; không cần tsDetect.

**AsD** = NAs / LOC.

**Lưu ý:**
- Source files đã được staged tại local khi chạy CK tool (`E:\NC\200_project\temp\...`). Đọc file tại đường dẫn tương ứng với TestFilePath.
- Nếu source files không còn trên disk, cần checkout lại từ git tại đúng tag.

### Bước 3: Gán nhãn isRefactored bằng RefactoringMiner

**Input:**
- Git repository (đã clone local)
- Cặp commit SHA: SHA_v1 (tag trước) và SHA_v2 (tag sau)

**Lưu ý về SHA:**
- Cột `SHA` trong `project.csv` là SHA của **tag thứ hai** (v2) trong cặp tag.
- SHA của tag v1 cần tra thêm từ git log: `git rev-list -n 1 <tag_v1>`.

**Quy trình:**
1. Với mỗi cặp (SHA_v1, SHA_v2), chạy RefactoringMiner ở chế độ between-commits.
2. RefactoringMiner trả về danh sách refactoring operations, mỗi operation kèm file path bị ảnh hưởng.
3. Lọc các file path thuộc test (kết thúc bằng `Test.java` hoặc nằm trong thư mục `test/`).
4. Test file xuất hiện trong ít nhất 1 refactoring → `isRefactored = True`, ngược lại → `False`.

**Lưu ý:**
- Kiểm tra output JSON của RefactoringMiner: dùng `rightSideLocations` (trạng thái sau refactoring) để map về TestFilePath.
- Một số refactoring type liên quan đến test (AddAssertArgument, ReplaceRuleWithAssertThrows…) nằm trong danh sách per_refactoring của Martins; nếu làm multi-label thì cần lọc theo từng loại refactoring.

### Bước 4: Thêm metadata và kiểm tra

- Thêm cột `project` (tên ngắn, ví dụ `zt-zip`).
- Thêm cột `App` theo format `org/repo` nhất quán với `ALL_merged.csv`.
- Kiểm tra: đúng 96 cột, đúng thứ tự, không có NaN ở các cột feature chính.
- Kiểm tra phân phối `isRefactored`: nếu 100% True hoặc 0% True thì có vấn đề với RefactoringMiner hoặc mapping.

---

## Lưu ý chung

- Nếu có nhiều project, bước 2 và bước 3 có thể song song hóa theo project.
- Các project rất nhỏ (< 10 mẫu) nên ghi chú riêng.
