# DevOps Session 05 - Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

Bài tập thực hành Git: rebase nhánh `feature-api` lên `main` khi cả hai nhánh cùng sửa các dòng trong `config.json`, gặp xung đột ở từng commit và giải quyết thủ công từng chặng. Kết quả là lịch sử thẳng hàng, không có merge commit, nhánh `main` không bị thay đổi.

Báo cáo chi tiết: [homework/session_05/ex3/README.md](homework/session_05/ex3/README.md)

## Danh sách file

| File | Nội dung |
|---|---|
| `homework/session_05/ex3/README.md` | Báo cáo chi tiết từng chặng xung đột và cách giải quyết |
| `homework/session_05/ex3/terminal_log.txt` | Toàn bộ lệnh và output thật khi chạy Git |
| `homework/session_05/ex3/config.json` | Bản `config.json` cuối cùng sau rebase |
| `homework/session_05/ex3/conflict_1.txt` | Nội dung `config.json` lúc xung đột chặng 1 (commit `feat: change port`) |
| `homework/session_05/ex3/conflict_2.txt` | Nội dung `config.json` lúc xung đột chặng 2 (commit `feat: enable debug`) |
