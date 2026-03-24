# Bài tập HDH Nhúng - Ứng dụng tổng hợp
## I. Giao tiếp trực tiếp với Device Driver
Hệ điều hành quản lý 4 đèn LED tích hợp của BBB tại đường dẫn ảo /sys/class/leds/.
- Tắt chế độ nhấp nháy mặc định của hệ thống:
Sử dụng lệnh echo none để ngắt trigger và giành quyền điều khiển LED USR0:
```
echo none > /sys/class/leds/beaglebone\:green\:usr2/trigger
```
- Bật LED (ON) và kiểm tra trạng thái:
```
echo 1 > /sys/class/leds/beaglebone\:green\:usr2/brightness
cat /sys/class/leds/beaglebone\:green\:usr2/brightness
```

<img width="895" height="504" alt="image" src="https://github.com/user-attachments/assets/0a37f8aa-9d74-4324-96a0-7d7f3280b884" />

- Tắt LED (OFF):
 ```
echo 0 > /sys/class/leds/beaglebone\:green\:usr2/brightness
```

<img width="934" height="519" alt="image" src="https://github.com/user-attachments/assets/7ff3a346-3ef9-4654-ab5e-60885a535717" />

## II. Buildroot + Blink LED USR2 
+ Truy cập thư mục Buildroot và Tạo cấu trúc package
```
cd ~/buildroot/
```
+ Tạo cấu trúc package
```
mkdir -p package/blink/src
```
### Tạo file chương trình C (blink.c) – dùng USR2
```
#include <stdio.h>
#include <unistd.h>

#define TRIGGER "/sys/class/leds/beaglebone:green:usr2/trigger"
#define BRIGHTNESS "/sys/class/leds/beaglebone:green:usr2/brightness"

void write_file(const char *path, const char *value) {
    FILE *f = fopen(path, "w");
    if (f != NULL) {
        fprintf(f, "%s", value);
        fclose(f);
    }
}

int main() {
    printf("CHUONG TRINH BLINK LED USR2\n");
    fflush(stdout);

    // Tắt trigger mặc định
    write_file(TRIGGER, "none");
    
    while(1) {
        write_file(BRIGHTNESS, "1");
        printf(">> LED USR2: ON\n");
        fflush(stdout);
        sleep(1);
        
        write_file(BRIGHTNESS, "0");
        printf(">> LED USR2: OFF\n");
        fflush(stdout);
        sleep(1);
    }   
    return 0;
}
```
### Tạo script tự khởi chạy
```
#!/bin/sh
case "$1" in
  start)
    printf "Starting Blink LED App... "
    /usr/bin/blink &
    echo "OK"
    ;;
  stop)
    printf "Stopping Blink LED App... "
    killall blink
    echo "OK"
    ;;
  *)
    echo "Usage: $0 {start|stop}"
    exit 1
    ;;
esac
```
Cấp quyền thực thi:
```
chmod +x package/blink/S99blink
```
### Tạo Config.in
```
config BR2_PACKAGE_BLINK
    bool "Blink LED App (USR2)"
    help
      Ung dung dieu khien LED USR2 va tu khoi dong cho BBB.
```

### Tạo file blink.mk
```
BLINK_VERSION = 1.0
BLINK_SITE = $(BLINK_PKGDIR)/src
BLINK_SITE_METHOD = local

define BLINK_BUILD_CMDS
    $(TARGET_CC) $(TARGET_CFLAGS) $(@D)/blink.c -o $(@D)/blink
endef

define BLINK_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/blink $(TARGET_DIR)/usr/bin/blink
endef

define BLINK_INSTALL_INIT_SYSV
    $(INSTALL) -D -m 0755 $(BLINK_PKGDIR)/S99blink $(TARGET_DIR)/etc/init.d/S99blink
endef

$(eval $(generic-package))
```
## Thêm package vào Buildroot
Mở file:
```
nano package/Config.in
source "package/blink/Config.in"
```
Lưu:
•	Ctrl + O → Enter
•	Ctrl + X


 ### Bật package trong menuconfig
```
make menuconfig
``` 
Đi theo đường:
```
Target packages
     → [*] Blink LED App (USR2)
```
 Save → Exit

### Biên dịch toàn bộ hệ thống
```
make
```
### Nạp image vào thẻ nhớ
Sau khi build xong, file nằm ở:
```
output/images/
```
 Ghi vào SD card (ví dụ /dev/sdb):
```
sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
sync
```
Kiểm tra đúng ổ đĩa trước khi ghi!

### Boot trên BeagleBone Black
-	Cắm SD card
-	Cấp nguồn
-	Mở serial (hoặc SSH)

### Kiểm tra chương trình chạy tự động

<img width="945" height="190" alt="image" src="https://github.com/user-attachments/assets/8e2dcaea-e0dc-41c5-8c59-835651ee8386" />

Sau khi boot xong:
```
ps | grep blink
```
Nếu thấy:
```
/usr/bin/blink
```
✔️ OK
### Kiểm tra LED USR2
LED sẽ:
```
ON 1s → OFF 1s → lặp lại
```
### Điều khiển thủ công (test)
Dừng app:
```
/etc/init.d/S99blink stop
```
Chạy lại:
```
/etc/init.d/S99blink start
```
