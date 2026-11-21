# VPS Script State & Handoff

> **Quy tắc cho AI (Codex / Gemini / Antigravity):** Đọc file này trước tiên để nắm cấu trúc, ràng buộc an toàn và lịch sử thay đổi. Tra cứu `graphify-out/graph.json` hoặc `graphify-out/GRAPH_REPORT.md` (hoặc chạy `graphify query`) để tìm hàm/file cụ thể. **Không đọc toàn bộ codebase** trừ khi cần sửa trực tiếp file liên quan.

## 1. Tổng quan dự án
- **Repo:** `hostvn-script` (`git@github.com:ngvannguyen/hostvn-script.git` — phân phối qua GitHub Pages: `https://ngvannguyen.github.io/hostvn-script`).
- **Phiên bản hiện tại:** `1.2.21` (khai báo tại `version`, `hostvn`, `update`).
- **Hệ điều hành mục tiêu:** Ubuntu **22.04 / 24.04 / 26.04 LTS** (26.04 experimental) + hỗ trợ container **Proxmox LXC** (unprivileged, `nesting=1`).
- **Web server / dịch vụ liên quan:** LEMP Stack — **Nginx** (build từ source 1.30.x: HTTP/3 QUIC, Brotli, `ngx_cache_purge` 2.5.6, `headers-more` 0.40, `vts` 0.2.5), **MariaDB 11.8 LTS**, **PHP-FPM** (`8.4` mặc định + multi-PHP `8.3 / 8.2 / 7.4`), **Redis**, **Memcached**, **UFW + Fail2ban** (tích hợp Cloudflare WAF sync), **Cloudflare Tunnel (`cloudflared`)**, **Rclone** (Google Drive / OneDrive / S3-compatible), **acme.sh** (Let's Encrypt + Wildcard DNS Cloudflare), **Python Telegram Bot** (`hostvn-telegram-bot.service`).
- **Thư mục đồ thị code (Graphify - local, đã `.gitignore`):** `graphify-out/graph.json`, `graphify-out/GRAPH_REPORT.md`, `graphify-out/graph.html`.

## 2. Cấu trúc thư mục & nhóm script chính
> Cập nhật các nhóm chính để agent định vị nhanh thay vì quét toàn bộ thư mục.

- `install` → `ubuntu` → `hostvn`: Chuỗi cài đặt mới từ đầu (kiểm tra OS/RAM/panel xung đột → cài gói nền & swap → build Nginx, cài MariaDB/PHP/phpMyAdmin, giải nén `menu` vào `/var/hostvn/`).
- `update` & `version`: Script nâng cấp tổng thể và file nguồn phiên bản tập trung (`script_version`, phiên bản PHP/MariaDB/extension).
- `.github/workflows/static.yml`: CI tự động đóng gói `menu/` thành `menu.tar.gz` (không commit tay `menu.tar.gz`) và publish toàn bộ artifact lên GitHub Pages khi push `main`.
- `menu/hostvn` & `menu/route/*`: Entrypoint lệnh CLI `hostvn` (`/usr/bin/hostvn`) và bộ điều hướng 15 nhóm menu.
- `menu/controller/*`: Logic nghiệp vụ CLI chia theo miền chức năng (`domain/`, `ssl/`, `cache/`, `php/`, `database/`, `backup/`, `cronjob/`, `firewall/`, `vps/`, `wordpress/`, `admin/`, `account/`, `permission/`, `telegram/`, `tools/`, `tunnel/`).
- `menu/helpers/` & `menu/validate/` & `menu/lang/`: Biến dùng chung (`variable_common`), hàm tiện ích & upload/index S3 (`function`), validate input (`rule`), ngôn ngữ `vi`/`en`.
- `menu/template/`: Template cấu hình Nginx vhost, rewrite theo mã nguồn (`wordpress`, `laravel`, `cakephp`, `cscart`, `sendy`...), FastCGI cache, SSL, alias/redirect.
- `menu/cronjob/`: Tác vụ cron chạy ngầm (`gg_drive_one`, `gg_drive_all`, `backup_onedrive_one`, `backup_onedrive_all`, `backup_local`, gia hạn/tối ưu).
- `menu/telegram_bot/`: Bot quản trị VPS bằng Python (`bot.py`, `handlers.py`, `menus.py`, `hostvn.py`, `permissions.py`, `config.py`, `progress.py`, `texts.py`) + lệnh điều khiển `hostvn-bot` (`bot_ctl.sh`).
- `menu/fail2ban/` & `menu/tunnel/`: Helper đồng bộ ban IP lên Cloudflare WAF (`cf_waf.sh`) và quản lý DNS/route Cloudflare Tunnel (`cf_tunnel.sh`).
- `docs/`: Tài liệu vận hành & nhật ký trạng thái (`VPS_STATE.md`), đi kèm lịch sử rebuild chi tiết ở `PROJECT.md`.

## 3. Luồng chạy chính & biến cấu hình quan trọng
- **Entrypoint / Script chính:**
  - Cài mới: `bash install` (ưu tiên file local nếu chạy từ bản clone `HOSTVN_LOCAL_DIR`, fallback tải từ `SCRIPT_LINK`).
  - Quản trị CLI: `/usr/bin/hostvn` → gọi `/var/hostvn/menu/hostvn`.
  - Quản trị qua Telegram: `hostvn-telegram-bot.service` (`venv/bin/python /var/hostvn/menu/telegram_bot/bot.py`), quản lý nhanh bằng `hostvn-bot {restart|start|stop|status|log}`.
- **Biến môi trường / file cấu hình cốt lõi trên VPS:**
  - `/var/hostvn/.hostvn.conf` (`FILE_INFO`): Chứa thông tin cốt lõi của máy (`mysql_pwd`, `admin_port`, `php1_version`, `php2_version`, `remote`, `backup_num`, `network_mode=direct|tunnel`, `lang`, `script_version`...).
  - `/var/hostvn/ipaddress`: Chứa `IPADDRESS` của VPS.
  - `/var/hostvn/users/.<domain>.conf`: Cấu hình từng website (`username`, `domain`, `db_name`, `db_user`, `php_mode`...).
  - `/var/hostvn/ggdrive/<domain>` & `/var/hostvn/onedrive/<domain>`: Cấu hình cron backup riêng từng domain (`remote`, `time`, `backup_num`).
  - `/var/hostvn/.backup_index.<remote>` & `<remote>:hostvn_backup_index_<IP>.txt`: Sổ mục lục backup phục vụ các S3 gateway không hỗ trợ liệt kê theo prefix (`ListObjectsV2`).
  - `/var/hostvn/.telegram_bot.conf`: Cấu hình token, `ADMIN_IDS`, `ALLOWED_CHAT_IDS` (override khi test bằng biến `HOSTVN_TGBOT_CONF`).
  - `/var/hostvn/menu/helpers/variable_common`: Định nghĩa `UPDATE_LINK`, `CURRENT_DATE`, `PATH`/`TERM` chuẩn cho cả CLI lẫn cronjob.
- **Đường dẫn dữ liệu & log:**
  - Web root: `/home/<user>/<domain>/public_html` (user riêng cho từng domain).
  - Thư mục tạm khi nén backup: `/home/backup/<YYYY-MM-DD>/<domain>`.
  - Nginx conf: `/etc/nginx/conf.d/<domain>.conf`, `/etc/nginx/ssl/`, `/etc/nginx/alias/`, `/etc/nginx/redirect/`.
  - Log: `/var/log/rclone.log` (backup cloud), `/home/<user>/<domain>/logs/` (error/access log từng web), `journalctl -u hostvn-telegram-bot` (log bot).
- **Điểm cần lưu ý khi trace luồng:**
  - Khi nâng cấp script (`update_scripts` / `update`), thư mục `/var/hostvn/menu/telegram_bot/venv` **phải được giữ lại** rồi chạy `hostvn-bot restart` (Python chỉ nạp module 1 lần lúc khởi động).
  - Mọi file script bắt buộc dùng line-ending **LF** (đã ép trong `.gitattributes`).
  - Khi nâng phiên bản script hoặc kiểm tra update từ GitHub Pages, các URL lấy `version`/`update`/`menu.tar.gz` đều gắn query `?t=$(date +%s)` để tránh dính cache CDN.

## 4. Nguyên tắc bảo mật Production, Không cho phép lỗi & Quyết định đã chốt

### 4.1. Nguyên tắc bảo mật khi chạy trên Production (Zero-Trust & Least Privilege)
- **Chống Command / SQL Injection tuyệt đối:** Cấm dùng `eval`, cấm truyền biến không bọc ngoặc kép (`"$var"`), và cấm nối trực tiếp đầu vào người dùng vào lệnh shell hoặc câu SQL. Mọi tham số (`domain`, `username`, `db_name`, callback từ Telegram) bắt buộc lọc qua whitelist (`menu/validate/rule`) hoặc `shlex.quote()` (trong `hostvn.py`).
- **Quyền file mật & cô lập tài khoản:** Toàn bộ file chứa thông tin nhạy cảm (`.hostvn.conf`, `.telegram_bot.conf`, `rclone.conf`, `.backup_index.*`, token Cloudflare) bắt buộc đặt quyền `600` thuộc `root:root`. Mỗi website chạy dưới Linux user và PHP-FPM pool riêng biệt, không được phép đọc chéo dữ liệu web khác hoặc truy cập `/var/hostvn/`.
- **Bảo vệ Secret & Log:** Tuyệt đối không commit mật khẩu, token, key hay đường dẫn máy cá nhân vào Git. Không để lộ token/mật khẩu ra `stdout`, traceback hay log hệ thống (ví dụ: ép `httpx` ở mức `WARNING`, chặn log `InvalidToken`). Script kiểm thử không bao giờ được chạm vào file cấu hình thật trên máy đang chạy (phải tách qua biến môi trường như `HOSTVN_TGBOT_CONF`).
- **Tối thiểu hóa bề mặt mạng & ép buộc HTTPS:** Mặc định chỉ mở đúng các port thiết yếu (SSH, `80`, `443` TCP/UDP, port Admin ngẫu nhiên); MariaDB mặc định khóa ở `127.0.0.1`. Mọi gói binary, extension (ionCube, PECL...), chữ ký ClamAV tải từ ngoài bắt buộc qua **HTTPS** (cấm HTTP trần tránh MITM). Ở chế độ Cloudflare Tunnel phải bật `real_ip_header CF-Connecting-IP` và `ignoreip 127.0.0.1/8 ::1` để Fail2ban chặn đúng IP thật thay vì khóa nhầm localhost.
- **Xác thực kép trên Telegram Bot:** Mọi lệnh/nút bấm bắt buộc kiểm tra đồng thời **ID phòng chat (`effective_chat.id`) VÀ ID người thao tác (`effective_user.id`)** qua `is_actor_allowed()` / `_gate()`, ngăn thành viên lạ trong nhóm chiếm quyền `root`.

### 4.2. Nguyên tắc không cho phép lỗi & Kiểm soát thay đổi Production (Zero-Fault / Change Control)
> **Trạng thái:** Script **đang chạy trên nhiều VPS Production thật** — một thay đổi thiếu kiểm soát có thể làm sập toàn bộ website hoặc mất dữ liệu không thể khôi phục. Mọi thay đổi mới bắt buộc tuân thủ:

- **Kiểm soát phạm vi thay đổi (Minimal Diff & Review bắt buộc):**
  - Không tự ý refactor lan man các script đang chạy ổn định (`update`, `hostvn`, `menu/template/*`). Sửa đúng điểm gây lỗi hoặc đúng tính năng được yêu cầu.
  - Trước khi sửa bất kỳ file nào có khả năng chạm tới dịch vụ đang chạy, phải phân tích rõ **tác động lên VPS đang chạy (khi bấm Update)** và trình bày rõ cho người dùng duyệt trước khi áp dụng/commit.
- **Zero-Downtime (Không làm gián đoạn dịch vụ đang chạy):**
  - Mọi tính năng/bản vá mới có thao tác sửa cấu hình hệ thống (Nginx, PHP-FPM, MariaDB, SSH, Firewall) phải đi đúng quy trình: **Sao lưu file cấu hình hiện tại → Ghi cấu hình mới → Kiểm tra cú pháp (`nginx -t`, `php-fpm -t`, `sshd -t`, `mariadbd --validate-config`, `bash -n`, `py_compile`) → Chỉ `reload` khi PASS; nếu FAIL phải tự động rollback ngay về bản sao lưu và giữ nguyên tiến trình cũ**.
  - Cập nhật script (`update_scripts` / `update`) hoặc nâng cấp dịch vụ phải giữ nguyên toàn bộ thiết lập tùy chỉnh đang chạy (`nginx.conf`, `bind-address` MariaDB, port SSH/Admin, `venv` của bot); không restart dịch vụ nếu file cấu hình của dịch vụ đó không thay đổi.
  - *Lưu ý thực trạng (`main` vs nhánh `dev`):* Trên nhánh `main` (production `v1.2.21`), file `update` vẫn giữ bản cũ. Toàn bộ cơ chế **Safe Update (CI `bash -n`/`py_compile`, staging `menu/`, backup + auto-rollback Nginx/PHP-FPM/MariaDB/Fail2ban, zero-downtime `reload`)** được phát triển tách riêng trên nhánh `dev` để review và test staging độc lập, **chưa merge vào `main`**.
- **Zero-Data-Loss (Tuyệt đối không mất dữ liệu):**
  - Cấm dùng `rm -rf` khi không có guard chống rỗng (`"${dir:?}"`) và kiểm tra đường dẫn hợp lệ.
  - Không bao giờ "xóa bản cũ trước rồi mới ghi bản mới". Với backup: chỉ được xóa thư mục tạm ở local sau khi đã kiểm tra **cả 3 chốt**: `mariadb-dump` trả `PIPESTATUS[0] == 0` (xóa ngay file `.sql.gz` rỗng 20-byte nếu lỗi để tránh mất bảng khi restore), `tar` trả mã thoát `< 2` kèm file `.tar.gz` khác rỗng (`-s`), và đối chiếu khớp đúng **byte size** từng file trên cloud (`rclone lsjson`).
- **No Silent Failure (Không nuốt lỗi, không báo thành công giả):**
  - Cấm bỏ qua mã thoát ở các bước trọng yếu rồi vẫn in thông báo thành công (`OK`). Gặp lỗi phải dừng ngay tại bước đó (`fail-fast`), giữ nguyên dữ liệu/cấu hình an toàn ban đầu và ghi rõ `ERROR`/`WARN` kèm nguyên nhân vào log để truy vết được ngay.
  - Trước khi phát hành: bắt buộc kiểm tra cú pháp toàn bộ file chạm tới (`bash -n`, `py_compile`, `git diff --check`, line-ending `LF`) và đối chiếu đầy đủ các điểm gọi liên quan qua đồ thị Graphify (tránh sửa chỗ này làm gãy chỗ khác hoặc bỏ sót nút chết `callback_data`).
- **Chống xung đột đồng thời & đảm bảo Idempotent:**
  - Các tác vụ nền/cron dùng chung thư mục tạm hoặc đẩy dữ liệu lên cloud bắt buộc xếp hàng tuần tự qua `flock` (`/var/lock/hostvn_cloud_backup.lock`).
  - Mọi hàm cài đặt/cập nhật phải có tính *idempotent* (chạy lại 2–3 lần vẫn ra đúng 1 kết quả sạch, không nhân đôi dòng cấu hình hay làm hỏng trạng thái hiện có).

### 4.3. Quy tắc Git & Quyết định kỹ thuật đã chốt (Không đề xuất lại cái đã loại)
- **Quy tắc Git & Release bắt buộc:** **TUYỆT ĐỐI KHÔNG tự động `git commit`, `git tag`, hay `git push`**. Chỉ sửa ở local và tóm tắt thay đổi để người dùng review `git diff` trước; chỉ commit/tag/push khi được yêu cầu rõ trong lượt đó.
- **Quyết định kỹ thuật đã chốt:**
  - **2026-07-17 — Bỏ hoàn toàn `ngx_pagespeed`, chuyển sang OpenSSL 3 + PCRE2 hệ thống + HTTP/3:** `ngx_pagespeed` đã bị Google ngừng phát triển, không compile được trên GCC/Ubuntu mới.
  - **2026-07-18 — Không commit `menu.tar.gz` vào Git:** Để GitHub Actions (`static.yml`) tự đóng gói khi push lên `main`, tránh lỗi sửa `menu/` nhưng quên nén lại gói phân phối.
  - **2026-07-18 — Không mang các hack chroot của nhánh ARM64 sang nhánh x86:** Không dùng shim thay `systemd` toàn cục, `hostvn-noop` fail2ban hay MariaDB trên `tmpfs` (nhánh x86 dùng `systemd` và `iptables`/`ufw` chuẩn; chỉ giữ shim tối giản nội bộ trong `hostvn.py`).
  - **2026-07-19 — Upload S3/Cloud dùng `rclone copy` cả thư mục + đối chiếu byte size qua `rclone lsjson` (`backup_upload_dir`):** **Đã loại bỏ** hướng `rclone copyto` từng file và hướng xóa-trước-ghi-đè (gây từ chối file thứ 2 hoặc mất trắng bản cũ trên S3 gateway `gofakes3`/TeleCloud). Nếu thư mục ngày đã có dữ liệu của domain đó thì đẩy sang `<ngày>_<giờ>`, xác minh khớp kích thước từng file mới xóa bản local.
  - **2026-11-12 — Cron backup tách dòng riêng cho từng tên miền:** Lưu cấu hình độc lập tại `/var/hostvn/ggdrive/<domain>` (`remote`, `time`, `backup_num`); có tùy chọn đổi lịch và gỡ khỏi cron mà không xóa file backup đã lưu.
  - **2026-11-13 — Thu hẹp mặt phẳng port khi cài mới (`v1.2.20`):** Mặc định chỉ mở SSH (`8282` hoặc port hiện tại), `80/tcp`, `443/tcp`, `443/udp` (HTTP/3) và `${RANDOM_ADMIN_PORT}/tcp`; bỏ cài `bind9` thừa (chỉ giữ `dnsutils`); MariaDB cài mới mặc định `bind-address=127.0.0.1` (chỉ mở `3306` khi bật Remote MySQL), đồng thời xử lý `ssh.socket` trên Ubuntu 24.04+ khi đổi port SSH.
  - **2026-11-15 — Tuần tự hóa toàn bộ cron backup bằng `flock` (`v1.2.21`):** Dùng chung khóa `/var/lock/hostvn_cloud_backup.lock` cho mọi script trong `menu/cronjob/` — **đã loại bỏ** việc chạy song song nhiều `rclone copy` lúc `00:00:00` vì gây race condition tạo trùng tên thư mục ngày (khác Folder ID) trên Google Drive.

## 5. Nhật ký thay đổi gần đây (Changelog ngắn)
> Mỗi lần sửa xong một tính năng/script và test ổn, thêm 1 mục ở trên cùng rồi cập nhật lại graph (`graphify update .` hoặc `graphify extract . --code-only` + `graphify cluster-only .`).

- **2026-11-21 — Nhánh `dev` (Safe Update, CI Gate & Auto-Rollback — KHÔNG merge vào `main`):**
  - Thay đổi: Tách nhánh `dev` từ `main`; thêm job `validate` (`bash -n` + `py_compile`) vào `.github/workflows/static.yml` (deploy Pages vẫn chỉ chạy cho `main`/`master`); nâng cấp `menu/controller/vps/update_scripts` (`curl -fsS`, kiểm tra `new_v` rỗng, `bash -n` trước khi chạy); nâng cấp `update` với staging + kiểm tra cú pháp toàn bộ `menu/` trước khi swap (giữ `/var/hostvn/menu.bak`), backup + kiểm tra cú pháp + auto-rollback cho Nginx (`nginx -t` + `reload`), PHP-FPM (`php-fpm -t` + `reload`), MariaDB (`mariadbd --validate-config`), Fail2ban (`fail2ban-client -t`), không ghi đè `cloudflare.conf` khi bật Tunnel, tải WP-CLI nguyên tử.
  - Đã kiểm tra: Review `git diff --check` trên nhánh `dev`; nhánh `main` giữ nguyên 100% tại `v1.2.21` (`7cf8b45`).
  - Còn tồn đọng / bước tiếp theo: Push nhánh `dev` để GitHub Actions chạy job `validate`, sau đó test thực tế trên VPS staging trước khi cân nhắc phát hành.
- **2026-11-21 — Tích hợp Graphify & khởi tạo `docs/VPS_STATE.md`:**
  - Thay đổi: Quét đồ thị code toàn repo vào `graphify-out/` (`1634 nodes`, `2903 edges`), đưa `graphify-out/` vào `.gitignore` (tránh đẩy cache & đường dẫn local lên GitHub Pages), thêm tài liệu trạng thái `docs/VPS_STATE.md`.
  - Đã kiểm tra: Kiểm tra `git status` xác nhận `graphify-out/` đã được ignore sạch sẽ.
  - Còn tồn đọng / bước tiếp theo: Chờ gộp `.gitignore` và `docs/VPS_STATE.md` vào đợt commit tiếp theo khi người dùng duyệt.
- **2026-11-15 — `v1.2.21` (`menu/cronjob/*`, `menu/helpers/function`, `menu/helpers/variable_common`):**
  - Thay đổi: Thêm khóa hàng đợi `flock` (`/var/lock/hostvn_cloud_backup.lock`) cho toàn bộ cron backup; tích hợp `backup_upload_dir` + `backup_index_add` để đối chiếu byte size trên remote trước khi xóa bản tạm; kiểm tra `PIPESTATUS[0]` của `mariadb-dump` và mã thoát `tar`; sửa lỗi dọn backup cũ sai đường dẫn trong `backup_onedrive_all`.
  - Đã kiểm tra: Review diff & push tag `v1.2.21`, hướng dẫn gộp thư mục trùng trên Google Drive bằng `rclone dedupe --dedupe-mode merge`.
  - Còn tồn đọng / bước tiếp theo: Theo dõi kỳ chạy cron `@weekly` / `@midnight` tiếp theo trên VPS thực tế.
- **2026-11-13 — `v1.2.20` (`hostvn`, `ubuntu`, `update`, `change_port_ssh`, `remote_mysql`):**
  - Thay đổi: Giới hạn mở port mặc định (SSH, 80, 443 TCP/UDP, port admin), gỡ `bind9`, đặt `bind-address=127.0.0.1` mặc định cho MariaDB cài mới (giữ nguyên trên máy cũ khi update), xử lý `ssh.socket` trên Ubuntu 24.04+.
  - Đã kiểm tra: Đã review và phát hành `v1.2.20`.
  - Còn tồn đọng / bước tiếp theo: Không.
- **2026-11-12 — `v1.2.17` → `v1.2.19` (`menu/controller/cronjob/backup_google`, `backup_onedrive`, `destroy_cron`, `delete_domain`):**
  - Thay đổi: Tách lịch cron, remote và số ngày lưu trữ riêng từng domain; ẩn domain đã lên lịch khi thêm mới; bổ sung xem danh sách, đổi lịch backup và xóa khỏi cron không xóa dữ liệu backup.
  - Đã kiểm tra: Chạy thực tế trên VPS nhiều domain với lịch `@midnight`, `@weekly`, `@monthly`.
  - Còn tồn đọng / bước tiếp theo: Đã bổ sung khóa `flock` ở `v1.2.21`.

## 6. Prompt mẫu khi mở chat mới
> *"Đọc `docs/VPS_STATE.md` và dùng `graphify-out/graph.json` (hoặc `graphify query`) để tìm đúng file/hàm liên quan trước khi sửa. Không đọc lại toàn bộ các file script không liên quan. Hôm nay cần: [mô tả task]."*
