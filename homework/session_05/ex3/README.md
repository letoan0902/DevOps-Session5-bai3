# Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

Môn DevOps - Session 05. Toàn bộ lệnh trong báo cáo được chạy thật bằng Git 2.49.0.windows.1 (Git Bash, Windows 11) trên repo demo `rebase-conflict-demo`. Output được chép nguyên văn từ file `terminal_log.txt` đi kèm (hash và thời gian là của lần chạy thật).

## 1. Mục tiêu

- Hiểu khác biệt giữa giải quyết xung đột khi Merge và khi Rebase.
- Áp dụng quy trình giải quyết xung đột từng chặng (từng commit) trong Rebase.
- Đưa lịch sử nhánh tính năng `feature-api` lên trên đầu nhánh `main` thành một đường thẳng, không có merge commit.

Ràng buộc của đề: phải giải quyết xung đột thủ công ở từng chặng, không dùng merge thông thường để tích hợp, không làm hỏng code của nhánh chính `main`.

Cách thực hiện thỏa ràng buộc:

- Mọi xung đột được sửa bằng cách ghi lại nội dung hoàn chỉnh của `config.json`, rồi `git add` và `git rebase --continue`.
- Không dùng `-X ours`, `-X theirs`, `git checkout --ours/--theirs`, `git rebase --skip`.
- Không dùng `git merge` để tích hợp `main` vào `feature-api`. Lệnh duy nhất có chữ merge ở cuối bài là `git merge --ff-only` (chỉ di chuyển con trỏ, không sinh merge commit) để chứng minh `main` có thể tiến lên thẳng hàng.
- Trong toàn bộ quá trình rebase, nhánh `main` không bị thay đổi (kiểm chứng ở mục 8).

## 2. Kịch bản và trạng thái trước khi rebase

Repo demo có một file `config.json`. Các commit được tạo theo đúng gợi ý của đề:

| Thứ tự tạo | Nhánh | Commit | Hash | Nội dung thay đổi |
|---|---|---|---|---|
| 1 | main | `init config` | `6ab2977` | Tạo `config.json` với `port` 8080, `debug` false |
| 2 | feature-api | `feat: change port` | `92244b2` | `port` 8080 -> 9000 |
| 3 | feature-api | `feat: enable debug` | `f2583be` | `debug` false -> true |
| 4 | main | `update port on main` | `759ffd1` | `port` 8080 -> 8081 |
| 5 | main | `add env config` | `582a71f` | Thêm `"env": "production"` (kèm dấu phẩy sau `"debug": false`) |

Lưu ý quan trọng ở commit 5: để thêm dòng `"env"` vào JSON hợp lệ thì dòng `"debug": false` phải thêm dấu phẩy. Vì vậy `main` đã sửa cả dòng `debug`, dòng mà commit `feat: enable debug` của `feature-api` cũng sửa. Đây là nguyên nhân của xung đột lần 2.

Trạng thái thực tế sau khi dựng kịch bản:

```
$ git log --graph --oneline --all
* f2583be feat: enable debug
* 92244b2 feat: change port
| * 582a71f add env config
| * 759ffd1 update port on main
|/  
* 6ab2977 init config

$ git merge-base main feature-api
6ab297709cb9a3b603843671c985381e374adedd
```

Tổ tiên chung của hai nhánh là `6ab2977 init config`. Nội dung `config.json` trên từng nhánh lúc này:

```
$ git show main:config.json
{
  "port": 8081,
  "debug": false,
  "env": "production"
}

$ git show feature-api:config.json
{
  "port": 9000,
  "debug": true
}
```

Sơ đồ ASCII trước rebase:

```
                     feature-api
                         |
              f2583be -- 92244b2          (feat: enable debug, feat: change port)
             /
 6ab2977 ---+
 (init)      \
              759ffd1 -- 582a71f          (update port on main, add env config)
                            |
                          main
```

Hai nhánh phân kỳ: mỗi nhánh có 2 commit riêng, và cả 4 commit đều sửa các dòng gần nhau trong cùng một file.

## 3. Cách Rebase hoạt động (để hiểu vì sao xung đột xảy ra nhiều lần)

`git rebase main` (đứng ở `feature-api`) thực hiện:

1. Lưu danh sách các commit của `feature-api` chưa có trong `main`: `92244b2`, `f2583be`.
2. Đưa `HEAD` về đầu `main` (`582a71f`), ở trạng thái detached HEAD.
3. Lần lượt áp dụng lại (cherry-pick) từng commit: lấy phần thay đổi (diff) của commit đó rồi áp lên đỉnh hiện tại. Mỗi lần áp dụng là một phép trộn ba bên riêng, nên xung đột có thể xuất hiện ở từng commit.
4. Mỗi commit áp dụng xong tạo ra một commit mới (hash mới) với cùng thông điệp. Cuối cùng nhánh `feature-api` được trỏ sang commit mới nhất.

Hệ quả: nếu cả hai commit đều đụng vào vùng `main` đã sửa thì phải giải quyết xung đột hai lần, mỗi lần cho một commit.

Ghi chú về biến môi trường: các lệnh tiếp tục rebase trong báo cáo ghi là `git rebase --continue`, nhưng thực tế đã chạy `GIT_EDITOR=true git rebase --continue`. Biến `GIT_EDITOR=true` khiến Git dùng lệnh `true` (không làm gì) thay cho trình soạn thảo, nên thông điệp commit được giữ nguyên và không bị mở editor tương tác.

## 4. Xung đột lần 1: commit `feat: change port` (1/2)

### 4.1 Phát sinh xung đột

```
$ git switch feature-api
Switched to branch 'feature-api'

$ git rebase main
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
Rebasing (1/2)error: could not apply 92244b2... feat: change port
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
hint: Disable this message with "git config set advice.mergeConflict false"
Could not apply 92244b2... feat: change port
```

Dòng `Rebasing (1/2)` cho biết đang áp dụng commit thứ nhất trong tổng số 2 commit. (Dòng này là chỉ báo tiến độ của Git, trong terminal thật nó bị dòng `error:` ghi đè nên trong log chép lại hai dòng dính liền.)

Nguyên nhân: commit `92244b2` đổi dòng `"port": 8080` thành `9000`, còn `main` đã đổi chính dòng đó thành `8081`, đồng thời sửa dòng `debug` và thêm dòng `env` ở sát bên. Git không thể tự quyết định giữ giá trị nào nên đánh dấu xung đột.

### 4.2 Trạng thái repo trong lúc xung đột

```
$ git status
interactive rebase in progress; onto 582a71f
Last command done (1 command done):
   pick 92244b2 feat: change port
Next command to do (1 remaining command):
   pick f2583be feat: enable debug
  (use "git rebase --edit-todo" to view and edit)
You are currently rebasing branch 'feature-api' on '582a71f'.
  (fix conflicts and then run "git rebase --continue")
  (use "git rebase --skip" to skip this patch)
  (use "git rebase --abort" to check out the original branch)

Unmerged paths:
  (use "git restore --staged <file>..." to unstage)
  (use "git add <file>..." to mark resolution)
	both modified:   config.json

no changes added to commit (use "git add" and/or "git commit -a")
```

Các điểm đọc được từ `git status`:

- `interactive rebase in progress; onto 582a71f`: đang giữa tiến trình rebase, nền là `582a71f` (đầu `main`).
- `Last command done`: `pick 92244b2 feat: change port` là commit đang xử lý (đã dừng vì xung đột).
- `Next command to do`: còn 1 commit `pick f2583be feat: enable debug` đang chờ.
- `both modified: config.json`: file bị sửa ở cả hai phía, cần giải quyết thủ công.

Nội dung file lúc xung đột (đã lưu ở `conflict_1.txt`):

```
$ cat config.json
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> 92244b2 (feat: change port)
}
```

```
$ git diff
diff --cc config.json
index 410d8a0,7388323..0000000
--- a/config.json
+++ b/config.json
@@@ -1,5 -1,4 +1,10 @@@
  {
++<<<<<<< HEAD
 +  "port": 8081,
 +  "debug": false,
 +  "env": "production"
++=======
+   "port": 9000,
+   "debug": false
++>>>>>>> 92244b2 (feat: change port)
  }
```

Trong `git diff` dạng combined (`diff --cc`) có hai cột tiền tố: cột trái so với phía `HEAD` (`main`), cột phải so với phía commit đang áp dụng. Dòng bắt đầu bằng ` +` là của `HEAD`, dòng bắt đầu bằng `+ ` là của commit `feat: change port`.

### 4.3 Ý nghĩa HEAD / ours / theirs trong rebase (ngược với merge)

- Phần từ `<<<<<<< HEAD` đến `=======` ("ours"): nền `main` cộng các commit feature đã áp dụng lại (port 8081, có `env`). Trong merge đứng ở feature thì phần này là nhánh feature của mình.
- Phần từ `=======` đến `>>>>>>>` ("theirs"): commit `92244b2` của `feature-api` đang được áp dụng lại (port 9000). Trong merge thì phần này là `main`.

Lý do đảo ngược: khi rebase, Git đưa `HEAD` về `main` rồi mới áp dụng từng commit của mình lên đó, nên commit của mình trở thành phía "theirs". Đây là điểm dễ nhầm nhất, và là lý do bài này không dùng `--ours`/`--theirs`: viết lại nội dung file bằng tay thì không phụ thuộc quy ước này.

### 4.4 Quyết định giải quyết chặng 1

| Dòng | Phía main (HEAD) | Phía commit feat: change port | Quyết định | Lý do |
|---|---|---|---|---|
| `port` | 8081 | 9000 | Lấy 9000 | Ý định của commit `feat: change port` chính là đổi port sang 9000 |
| `debug` | `false,` (có dấu phẩy) | `false` (không dấu phẩy) | Lấy `false,` có dấu phẩy | Dấu phẩy là thay đổi bắt buộc của `main` để thêm `env`; bỏ đi sẽ làm hỏng JSON |
| `env` | `"env": "production"` | không có | Giữ nguyên | Là code mới của `main`, không được làm mất |

Nguyên tắc: giữ toàn bộ thay đổi của `main`, chỉ chồng ý định riêng của commit đang áp dụng lên trên. `debug` vẫn `false` vì commit này chưa bật debug.

### 4.5 Nội dung sau khi sửa và lệnh tiếp tục

```
$ printf "{\n  \"port\": 9000,\n  \"debug\": false,\n  \"env\": \"production\"\n}\n" > config.json && cat config.json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}

$ grep -nE "^(<<<<<<<|=======|>>>>>>>)" config.json || echo "Khong con ky hieu xung dot"
Khong con ky hieu xung dot

$ python -m json.tool config.json
{
    "port": 9000,
    "debug": false,
    "env": "production"
}

$ git add config.json

```

Sau `git add config.json`, `git status` (xem `terminal_log.txt`) chuyển từ `Unmerged paths` sang `Changes to be committed` và Git báo `all conflicts fixed: run "git rebase --continue"`. Việc kiểm tra bằng `python -m json.tool` xác nhận file là JSON hợp lệ (không còn ký hiệu xung đột, đủ dấu phẩy).

Lệnh tiếp tục:

```
$ git rebase --continue
[detached HEAD 96ebb77] feat: change port
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Commit `92244b2` đã được viết lại thành `96ebb77` (cùng thông điệp, nền mới là `582a71f`). Ngay sau đó, rebase chuyển sang commit thứ hai.

## 5. Xung đột lần 2: commit `feat: enable debug` (2/2)

### 5.1 Phát sinh xung đột

Ngay khi `git rebase --continue` áp dụng commit thứ hai, xung đột lại xuất hiện (đây là phần output tiếp theo của chính lệnh ở mục 4.5):

```
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
Rebasing (2/2)error: could not apply f2583be... feat: enable debug
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
hint: Disable this message with "git config set advice.mergeConflict false"
Could not apply f2583be... feat: enable debug
```

Nguyên nhân: commit `f2583be` đổi `"debug": false` thành `"debug": true` (không có dấu phẩy). Trong khi đó, nền hiện tại đã có dòng `"debug": false,` (có dấu phẩy, kèm dòng `env` ngay bên dưới) do `main` sửa. Cùng một dòng được hai bên sửa theo hai cách nên Git không tự trộn được. Đây là xung đột thật, đúng như đề dự đoán, và nó chỉ xuất hiện vì `main` phải thêm dấu phẩy khi thêm `env`.

Lưu ý: dòng `port` không còn xung đột ở chặng này vì commit `f2583be` không sửa `port` (và `port` 9000 đã được chốt ở chặng 1).

### 5.2 Trạng thái repo trong lúc xung đột

```
$ git status
interactive rebase in progress; onto 582a71f
Last commands done (2 commands done):
   pick 92244b2 feat: change port
   pick f2583be feat: enable debug
No commands remaining.
You are currently rebasing branch 'feature-api' on '582a71f'.
  (fix conflicts and then run "git rebase --continue")
  (use "git rebase --skip" to skip this patch)
  (use "git rebase --abort" to check out the original branch)

Unmerged paths:
  (use "git restore --staged <file>..." to unstage)
  (use "git add <file>..." to mark resolution)
	both modified:   config.json

no changes added to commit (use "git add" and/or "git commit -a")
```

Lần này `Last commands done` liệt kê cả hai lệnh `pick` và `No commands remaining`: đây là commit cuối cùng của tiến trình rebase.

Nội dung file lúc xung đột (đã lưu ở `conflict_2.txt`):

```
$ cat config.json
{
  "port": 9000,
<<<<<<< HEAD
  "debug": false,
  "env": "production"
=======
  "debug": true
>>>>>>> f2583be (feat: enable debug)
}
```

```
$ git diff
diff --cc config.json
index 0ca5390,d164232..0000000
--- a/config.json
+++ b/config.json
@@@ -1,5 -1,4 +1,9 @@@
  {
    "port": 9000,
++<<<<<<< HEAD
 +  "debug": false,
 +  "env": "production"
++=======
+   "debug": true
++>>>>>>> f2583be (feat: enable debug)
  }
```

Quan sát: dòng `"port": 9000,` nằm ngoài vùng xung đột, nghĩa là kết quả chặng 1 đã được giữ. Phần `HEAD` (ours) lúc này là `96ebb77` (commit chặng 1 vừa tạo) chồng trên `main`, và phần `theirs` là commit `f2583be`.

### 5.3 Quyết định giải quyết chặng 2

| Dòng | Phía HEAD | Phía commit feat: enable debug | Quyết định | Lý do |
|---|---|---|---|---|
| `port` | 9000 (ngoài vùng xung đột) | không đổi | Giữ 9000 | Kết quả chặng 1 |
| `debug` | `false,` | `true` | Lấy `true,` (có dấu phẩy) | Ý định của commit là bật debug; dấu phẩy vẫn bắt buộc vì phía dưới còn `env` |
| `env` | `"env": "production"` | không có | Giữ nguyên | Code của `main` |

### 5.4 Nội dung sau khi sửa và lệnh tiếp tục

```
$ printf "{\n  \"port\": 9000,\n  \"debug\": true,\n  \"env\": \"production\"\n}\n" > config.json && cat config.json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}

$ grep -nE "^(<<<<<<<|=======|>>>>>>>)" config.json || echo "Khong con ky hieu xung dot"
Khong con ky hieu xung dot

$ python -m json.tool config.json
{
    "port": 9000,
    "debug": true,
    "env": "production"
}

$ git add config.json

$ git rebase --continue
[detached HEAD a2272bd] feat: enable debug
 1 file changed, 1 insertion(+), 1 deletion(-)
Successfully rebased and updated refs/heads/feature-api.
```

Dòng `Successfully rebased and updated refs/heads/feature-api.` xác nhận hoàn tất: nhánh `feature-api` đã được trỏ sang commit mới `a2272bd`. Tổng cộng có 2 lần xung đột, mỗi commit một lần, đúng như đề mô tả.

## 6. Kết quả cuối cùng

### 6.1 Kiểm tra theo yêu cầu của đề

```
$ git status
On branch feature-api
nothing to commit, working tree clean

$ git log --graph --oneline
* a2272bd feat: enable debug
* 96ebb77 feat: change port
* 582a71f add env config
* 759ffd1 update port on main
* 6ab2977 init config
```

Đây là nội dung "ảnh chụp" `git log --graph --oneline` sau cùng (chép dạng văn bản từ terminal). Lịch sử là một đường thẳng, chỉ có một cột dấu `*`, không có nhánh rẽ.

```
$ git log --oneline --merges

```

`git log --oneline --merges` không in dòng nào: không có merge commit nào trong lịch sử.

Nội dung file cuối (lưu ở `config.json`):

```
$ cat config.json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

Bao gồm đủ: `port` 9000 (từ `feature-api`), `debug` true (từ `feature-api`), `env` production (từ `main`).

So sánh `feature-api` với `main`:

```
$ git diff main feature-api
diff --git a/config.json b/config.json
index 410d8a0..ed51b27 100644
--- a/config.json
+++ b/config.json
@@ -1,5 +1,5 @@
 {
-  "port": 8081,
-  "debug": false,
+  "port": 9000,
+  "debug": true,
   "env": "production"
 }
```

`feature-api` so với `main` chỉ khác đúng hai ý định của mình (port và debug). Dòng `env` không xuất hiện trong diff nghĩa là code của `main` được giữ nguyên.

```
$ git show --stat HEAD~1 HEAD
commit 96ebb77dea252c1f8a08c8a84e820429df83e9e7
Author: Lê Toàn <trinhhanh261293@gmail.com>
Date:   Fri Oct 2 12:47:15 2026 +0700

    feat: change port

 config.json | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

commit a2272bd1d119b197ac3e5b8b9649b38ec092bf8c
Author: Lê Toàn <trinhhanh261293@gmail.com>
Date:   Fri Oct 2 12:47:15 2026 +0700

    feat: enable debug

 config.json | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

```
$ git log --format="%h %p | %s" main..feature-api
a2272bd 96ebb77 | feat: enable debug
96ebb77 582a71f | feat: change port
```

Mỗi commit feature chỉ có một commit cha (cột giữa), không commit nào có hai cha, tức không có merge commit.

### 6.2 Hash cũ và hash mới của các commit feature

```
$ git reflog -n 8
a2272bd HEAD@{0}: rebase (finish): returning to refs/heads/feature-api
a2272bd HEAD@{1}: rebase (continue): feat: enable debug
96ebb77 HEAD@{2}: rebase (continue): feat: change port
582a71f HEAD@{3}: rebase (start): checkout main
f2583be HEAD@{4}: checkout: moving from main to feature-api
582a71f HEAD@{5}: commit: add env config
759ffd1 HEAD@{6}: commit: update port on main
6ab2977 HEAD@{7}: checkout: moving from feature-api to main
```

| Commit | Hash trước rebase | Hash sau rebase | Cha (parent) trước | Cha sau |
|---|---|---|---|---|
| `feat: change port` | `92244b2` | `96ebb77` | `6ab2977` | `582a71f` |
| `feat: enable debug` | `f2583be` | `a2272bd` | `92244b2` | `96ebb77` |

Rebase không "di chuyển" commit mà tạo commit mới: nội dung (cây file) khác, cha khác nên hash khác. Các commit cũ `92244b2`, `f2583be` vẫn còn trong reflog (`HEAD@{4}` là `f2583be`) một thời gian, có thể khôi phục nếu cần.

Rebase giữ nguyên author date của commit gốc, còn committer date là lúc áp dụng lại (xem `terminal_log.txt`, Buoc 8).

### 6.3 Sơ đồ ASCII sau rebase

```
 6ab2977 -- 759ffd1 -- 582a71f -- 96ebb77 -- a2272bd
 (init)     (port)     (env)      (feat:     (feat:
                        |          change     enable
                        |          port)      debug)
                      main                       |
                                            feature-api
```

Hai commit của `feature-api` nối tiếp ngay sau commit mới nhất của `main` (`582a71f`), đúng như kết quả mong đợi của đề.

## 7. Bảng so sánh xử lý xung đột: Merge và Rebase

Cột Merge dưới đây mô tả cách hoạt động lý thuyết của Git để so sánh; trong bài này không chạy `git merge` (đề cấm), nên các con số ở cột Merge là suy ra từ cơ chế, không phải đo từ thực nghiệm.

| Tiêu chí | Merge (`git merge main` ở feature) | Rebase (`git rebase main` ở feature) |
|---|---|---|
| Số lần xung đột với kịch bản này | Thường 1 lần: Git trộn một lần giữa đầu hai nhánh và tổ tiên chung, mọi khác biệt gộp vào cùng một lần giải quyết | 2 lần (đã đo thực tế): mỗi commit của feature được áp dụng lại và có thể xung đột riêng |
| Ý nghĩa `ours` / `HEAD` | Nhánh đang đứng (feature) | Nền `main` cộng các commit đã áp dụng lại |
| Ý nghĩa `theirs` | Nhánh được kéo vào (`main`) | Commit của feature đang được áp dụng lại |
| Dạng lịch sử | Có merge commit với hai cha, đồ thị có rẽ nhánh và gộp | Tuyến tính, một đường thẳng, không merge commit |
| Hash của commit feature | Giữ nguyên | Đổi hết (92244b2 -> 96ebb77, f2583be -> a2272bd) |
| Lưu vết giải quyết xung đột | Nằm gọn trong merge commit | Nằm rải vào từng commit đã viết lại |
| Độ khó lúc giải quyết | Một lần, thấy tổng thể, nhưng khó biết ý định từng commit | Nhiều lần, mỗi lần nhỏ và rõ ý định từng commit, nhưng lặp lại nếu có nhiều commit |
| An toàn với nhánh dùng chung | An toàn, không viết lại lịch sử | Nguy hiểm nếu nhánh đã được người khác dùng, vì viết lại lịch sử (xem mục 9) |
| Dễ hoàn tác | `git reset`/`git revert` merge commit | `git rebase --abort` (khi đang dở) hoặc `git reflog` + `git reset --hard` (khi xong) |

## 8. Chứng minh không làm hỏng nhánh `main`

Trong toàn bộ quá trình rebase chỉ nhánh `feature-api` thay đổi; `main` vẫn trỏ vào `582a71f`:

```
$ git log --graph --oneline --decorate --all
* a2272bd (HEAD -> feature-api) feat: enable debug
* 96ebb77 feat: change port
* 582a71f (main) add env config
* 759ffd1 update port on main
* 6ab2977 init config

$ git log --oneline main
582a71f add env config
759ffd1 update port on main
6ab2977 init config

$ git show main:config.json
{
  "port": 8081,
  "debug": false,
  "env": "production"
}
```

`main` giữ nguyên 3 commit và nội dung `config.json` (port 8081, debug false, env production) như trước khi rebase.

Do `feature-api` giờ nằm ngay trên đầu `main`, việc tích hợp vào `main` là một fast-forward thuần túy (chỉ dịch con trỏ, không sinh commit mới, không xung đột):

```
$ git switch main
Switched to branch 'main'

$ git merge --ff-only feature-api
Updating 582a71f..a2272bd
Fast-forward
 config.json | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)

$ git log --graph --oneline --decorate
* a2272bd (HEAD -> main, feature-api) feat: enable debug
* 96ebb77 feat: change port
* 582a71f add env config
* 759ffd1 update port on main
* 6ab2977 init config
```

`--ff-only` chỉ thành công khi có thể tiến thẳng; nếu lịch sử phân kỳ nó sẽ từ chối thay vì tạo merge commit. Đây là cách an toàn để đảm bảo "không có merge commit phụ". Sau fast-forward, `git log --oneline --merges` vẫn rỗng và `git status` báo `working tree clean`.

## 9. Lệnh thoát hiểm và công cụ hỗ trợ

| Lệnh | Khi nào dùng | Tác dụng |
|---|---|---|
| `git rebase --abort` | Đang dở rebase, thấy rối hoặc sửa sai | Hủy toàn bộ, trả nhánh về đúng trạng thái trước `git rebase` (ở bài này là `f2583be`). Không mất gì |
| `git rebase --skip` | Commit đang áp dụng không còn cần thiết (ví dụ nội dung đã có sẵn ở `main`) | Bỏ qua commit hiện tại, chuyển sang commit kế tiếp. Commit bị bỏ sẽ mất khỏi nhánh mới. Bài này không dùng vì cả hai commit đều mang ý định cần giữ |
| `git rebase --continue` | Đã sửa xong và `git add` | Tạo commit từ nội dung đã sửa rồi áp dụng commit kế tiếp |
| `git reflog` + `git reset --hard <hash cũ>` | Rebase đã xong nhưng muốn quay lại | Đưa nhánh về hash cũ (ví dụ `f2583be`) nhờ reflog |
| `git config rerere.enabled true` | Phải rebase/merge lặp lại nhiều lần cùng một xung đột | Bật `rerere` (reuse recorded resolution): Git ghi nhớ cách giải quyết và tự áp dụng lại lần sau. Trạng thái trong repo demo: `rerere.enabled` chưa được đặt (mặc định tắt) |

Trước khi `git add` luôn kiểm tra không còn ký hiệu xung đột (bằng `grep`) và JSON hợp lệ (bằng `python -m json.tool`), như đã làm ở mục 4.5 và 5.4.

## 10. Nguyên tắc: không rebase nhánh đã chia sẻ

Rebase viết lại lịch sử (tạo commit mới với hash mới). Nếu nhánh đã được push lên remote và người khác đã kéo về, việc rebase làm lịch sử của bạn và của họ lệch nhau. Khi họ pull, Git sẽ thấy hai chuỗi commit "giống nhau về nội dung nhưng khác hash", dẫn đến merge rối, commit bị nhân đôi, hoặc phải ép đẩy `git push --force` và ghi đè công việc của người khác.

Quy tắc thực hành:

- Chỉ rebase nhánh cá nhân, chưa ai khác dựa vào (như `feature-api` trong bài này, nơi chưa push).
- Không rebase `main` hay bất kỳ nhánh chung nào.
- Nếu buộc phải push lại nhánh sau khi rebase, dùng `git push --force-with-lease` (an toàn hơn `--force`) và báo cho những người liên quan.
- Khi nhánh đã chia sẻ, ưu tiên `git merge` hoặc `git revert` thay vì rebase.

## 11. Đối chiếu yêu cầu đề bài

| Yêu cầu của đề | Kết quả | Minh chứng |
|---|---|---|
| Dựng đúng 5 commit theo gợi ý (`init config`, `feat: change port`, `feat: enable debug`, `update port on main`, `add env config`) | Đạt | Mục 2, các hash `6ab2977`, `92244b2`, `f2583be`, `759ffd1`, `582a71f` |
| `git rebase main` gặp xung đột lần 1, sửa thủ công, `git add`, `git rebase --continue` | Đạt | Mục 4 |
| Lặp lại cho xung đột lần 2 | Đạt (thực tế có xung đột lần 2) | Mục 5 |
| Giải quyết thủ công từng chặng, không merge thông thường | Đạt | Viết lại file thủ công; không dùng `git merge`, `-X`, `--ours/--theirs`, `--skip` |
| Không phá hỏng code của `main` | Đạt | Mục 8: `main` vẫn là `582a71f`, `config.json` không đổi |
| `git status` (trạng thái rebase) | Đạt | Mục 4.2, 5.2, 6.1; trạng thái sau `git add` xem `terminal_log.txt` |
| `git log --graph --oneline` sau cùng thẳng tắp, không merge commit | Đạt | Mục 6.1 và `git log --oneline --merges` rỗng |
| Commit của `feature-api` nối ngay sau commit mới nhất của `main` | Đạt | `96ebb77` có cha là `582a71f` (mục 6.1) |
| README chi tiết các lần conflict và cách giải quyết từng chặng | Đạt | Mục 4 và 5 |
| Ảnh chụp `git log --graph --oneline` sau cùng | Đạt | Output của lệnh được chép nguyên văn dạng văn bản (mục 6.1) và lưu đầy đủ trong `terminal_log.txt` |

## 12. Kết luận

- Kịch bản của đề tái hiện đúng: `git rebase main` gặp 2 lần xung đột, mỗi lần ở một commit của `feature-api`, vì cả hai commit đều đụng vào vùng `config.json` mà `main` đã sửa (đặc biệt dòng `debug` bị đổi dấu phẩy khi `main` thêm `env`).
- Trong rebase, `HEAD`/"ours" là nền `main` và phần "theirs" là commit của mình đang được áp dụng lại, ngược với merge. Giải quyết bằng cách viết lại nội dung file hoàn chỉnh giúp tránh nhầm phía.
- Nguyên tắc giải quyết mỗi chặng: giữ nguyên thay đổi của `main`, chỉ chồng ý định riêng của commit đang xử lý lên trên, kiểm tra JSON hợp lệ trước khi `git add`.
- Kết quả: lịch sử thẳng tắp gồm 5 commit, không có merge commit; hai commit feature được viết lại thành `96ebb77` và `a2272bd`, nối ngay sau `582a71f` của `main`. `config.json` cuối cùng có `port` 9000, `debug` true, `env` production.
- Cái giá của rebase là đổi hash commit và phải giải quyết xung đột nhiều lần; vì vậy chỉ nên rebase nhánh cá nhân và dùng `--abort`, reflog làm lưới an toàn.

## Danh sách file trong thư mục này

| File | Nội dung |
|---|---|
| `README.md` | Báo cáo này |
| `terminal_log.txt` | Toàn bộ lệnh và output thật |
| `config.json` | Bản cuối cùng sau rebase |
| `conflict_1.txt` | Nội dung `config.json` lúc xung đột chặng 1 |
| `conflict_2.txt` | Nội dung `config.json` lúc xung đột chặng 2 |
