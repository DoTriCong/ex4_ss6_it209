# Bài 4: Quản lý tiến trình nền với nohup và tín hiệu Kill

Nội dung thư mục:

- `loop-monitor.sh`: script ghi thời gian vào `/tmp/monitor.log` mỗi 5 giây
- `README.md`: báo cáo này

## 1. Tạo script `loop-monitor.sh`

```bash
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
chmod +x loop-monitor.sh
```

Kiểm tra quyền thực thi:

```text
$ ls -l loop-monitor.sh
-rwxr-xr-x 1 root root 96 Oct  6 20:20 loop-monitor.sh
```

## 2. Khởi chạy nền độc lập bằng nohup

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

- `nohup`: bỏ qua tín hiệu SIGHUP, nên tiến trình không bị tắt khi đóng Terminal / ngắt SSH.
- `> /dev/null 2>&1`: bỏ stdout và stderr, không sinh file `nohup.out`.
- `&`: chạy ở chế độ nền, trả lại dấu nhắc lệnh ngay.

## 3. Tìm PID

```bash
pgrep -f loop-monitor.sh
```

```text
111
```

Kiểm tra bằng `ps`:

```bash
ps aux | grep loop-monitor.sh
```

```text
root       111  0.0  0.0   4332  3392 ?        Ss   20:20   0:00 /bin/bash ./loop-monitor.sh
```

**PID của tiến trình: `111`**

## 4. Kiểm tra log được ghi liên tục

```bash
tail -n 10 /tmp/monitor.log
```

```text
System time: Tue Oct  6 20:20:39 UTC 2026
System time: Tue Oct  6 20:20:44 UTC 2026
System time: Tue Oct  6 20:20:49 UTC 2026
```

Các dòng cách nhau đúng 5 giây, đúng với `sleep 5`.

## 5. Tắt tiến trình

Bước 1: gửi SIGTERM (15) để tiến trình kết thúc nhẹ nhàng:

```bash
kill -15 111
```

Bước 2: kiểm tra lại:

```bash
ps aux | grep loop-monitor.sh
```

Tiến trình đã nhận tín hiệu và dừng, `pgrep -f loop-monitor.sh` không còn trả về PID nào.

> **Ghi chú:** Trong môi trường mình chạy thử (container không có tiến trình init để thu dọn), `ps` còn hiện dòng `[loop-monitor.sh] <defunct>` (trạng thái `Z`, zombie). Đó chỉ là xác tiến trình chờ tiến trình cha thu dọn, nó đã dừng chạy và không còn ghi log. Trên VPS thông thường, tiến trình sẽ biến mất hoàn toàn khỏi `ps aux`.

Chỉ khi SIGTERM không có tác dụng (tiến trình bị treo) mới dùng SIGKILL (9):

```bash
kill -9 111
```

## 6. Kiểm tra tiến trình vẫn sống sau khi ngắt SSH

1. Chạy `nohup ./loop-monitor.sh > /dev/null 2>&1 &`.
2. Đóng Terminal / ngắt SSH, rồi đăng nhập lại.
3. Chạy `pgrep -f loop-monitor.sh` và `tail -f /tmp/monitor.log` thì tiến trình vẫn còn, log vẫn tiếp tục tăng.

## 7. So sánh SIGTERM và SIGKILL

| Tín hiệu | Số  | Đặc điểm                                                                                         |
| -------- | --- | ------------------------------------------------------------------------------------------------ |
| SIGTERM  | 15  | Yêu cầu tiến trình tự kết thúc, cho phép dọn dẹp (đóng file, giải phóng tài nguyên). Dùng trước. |
| SIGKILL  | 9   | Kernel buộc dừng ngay, tiến trình không thể bắt hay bỏ qua. Dùng khi SIGTERM không hiệu quả.     |
