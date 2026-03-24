# BÀI TẬP HỆ ĐIỀU HÀNH NHÚNG TUẦN 6 ỨNG DỤNG TỔNG HỢP
## 1. GIAO TIẾP VỚI DRIVER TỪ ỨNG DỤNG 
- Nạp driver : Yêu cầu kernel load driver và driver này dùng để điều khiển LED qua
  ```bash
  modprobe leds-gpio
- Kiểm tra xem đã tạo file tương ứng Led chưa
  ```bash
  ls /sys/class/leds/beaglebone:green:/
- Tắt chế độ điều khiển tự động của LED và chuyển sang điều khiển thủ công
  ```bash
  echo none > /sys/class/leds/beaglebone:green:usr3/trigger
- Bật LED
  ```bash
  echo 1 > /sys/class/leds/beaglebone:green:usr3/brightness
- Tắt LED
  ```bash
  echo 0 > /sys/class/leds/beaglebone:green:usr3/brightness
- Xem Led đang bật hay tắt bằng lệnh
  ```bash
  cat /sys/class/leds/beaglebone:green:usr3/brightness
- Kết quả thu được
  <img width="2560" height="1920" alt="image" src="https://github.com/user-attachments/assets/1077b980-e925-449f-8b7e-6e4cb91dfb29" />

# BÀI 2  Viết chương trình giao tiếp
## 1. CẤU TRÚC THƯ MỤC
```bash
package/
└── led-driver/
    ├── Config.in
    ├── led-driver.mk
    └── src/
        ├── led_driver.c
        └── Makefile
```
## 2. CÁC BƯỚC LÀM
### BƯỚC 1: TẠO THƯ MỤC PACKAGE
```bash
cd ~/buildroot/package
mkdir led-driver
cd led-driver
mkdir src
```
### BƯỚC 2: TẠO FILE CONFIG.IN
```bash
nano Config.in
```
- Dán code:
```bash
config BR2_PACKAGE_LED_DRIVER
    bool "led-driver"
    help
      Simple GPIO LED driver for BBB
```
### BƯỚC 3: TẠO FILE LED_DRIVER.MK
```bash
nano led_driver.mk
```
- Dán code:
```bash
LED_DRIVER_VERSION = 1.0
LED_DRIVER_SITE = $(TOPDIR)/package/led-driver/src
LED_DRIVER_SITE_METHOD = local

define LED_DRIVER_BUILD_CMDS
	$(MAKE) -C $(@D)
endef

define LED_DRIVER_INSTALL_TARGET_CMDS
	mkdir -p $(TARGET_DIR)/lib/modules
	cp $(@D)/led_driver.ko $(TARGET_DIR)/lib/modules/
endef

$(eval $(generic-package))
```
### BƯỚC 4: TẠO FILE SOURCE CODE TRONG src/

4.1 Tạo driver
```bash
nano src/led_driver.c
```
- Dán code:
```bash
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "led_test"

static int major;
static char msg[10];

static ssize_t dev_read(struct file *file, char __user *buf, size_t len, loff_t *off) {
    copy_to_user(buf, msg, sizeof(msg));
    return sizeof(msg);
}

static ssize_t dev_write(struct file *file, const char __user *buf, size_t len, loff_t *off) {
    copy_from_user(msg, buf, len);

    if (strncmp(msg, "ON", 2) == 0) {
        printk("LED ON\n");
    } else if (strncmp(msg, "OFF", 3) == 0) {
        printk("LED OFF\n");
    }

    return len;
}

static struct file_operations fops = {
    .read = dev_read,
    .write = dev_write,
};

static int __init led_init(void) {
    major = register_chrdev(0, DEVICE_NAME, &fops);
    printk("Driver loaded\n");
    return 0;
}

static void __exit led_exit(void) {
    unregister_chrdev(major, DEVICE_NAME);
    printk("Driver removed\n");
}

module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");
```

4.2 Tạo Makefile cho driver.
```bash
nano src/Makefile
```
- Dán code:
```bash
obj-m += led_driver.o

all:
	make -C $(KERNEL_DIR) M=$(PWD) modules

clean:
	make -C $(KERNEL_DIR) M=$(PWD) clean
```
## BƯỚC 5: KHAI BÁO PACKAGE VỚI BUILDROOT
- Mở file:
```bash
nano ~/buildroot/package/Config.in
```
- Thêm dòng vào cuối file:
```bash
source "package/led-driver/Config.in"
```
## BƯỚC 6: ENBALE PACKAGE
```bash
cd ~/buildroot
make menuconfig
```
- Vào:
```bash
Target packages → led-driver
```
## BƯỚC 7: BUILD
```bash
make
```
## BƯỚC 8: TEST KIỂM THỬ TRÊN BBB
- Trong quá trình làm thì nhóm có sửa code và quy ước thành:
```bash
0: LED OFF
1: LED ON
```
- Thu được kết quả dưới đây:




https://github.com/user-attachments/assets/f9976622-5af5-4f5a-8993-099a4a1e0547

# BÀI 3 TỰ KHỞI ĐỘNG 
- Cấu trúc thư mục bổ sung:
package/led_driver/
└── S99blynk
- Trong thư mục package của bạn (package/led_driver/), hãy tạo thêm một file tên là S99blynk
```bash
#!/bin/sh

case "$1" in
  start)
    echo "Starting Blynk_user2..."
    # Chạy ngầm chương trình bằng dấu & để không làm treo quá trình boot
    /usr/bin/Blynk_user2 &
    ;;
  stop)
    echo "Stopping Blynk_user2..."
    killall Blynk_user2
    ;;
  restart|reload)
    $0 stop
    $0 start
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
esac

exit 0
```
- Cập nhật file led_driver.mk

- Sửa nội dung define BLYNK_USER2_INSTALL_TARGET_CMDS như sau:

define BLYNK_USER2_INSTALL_TARGET_CMDS
    # Cài đặt file thực thi
    $(INSTALL) -D -m 0755 $(@D)/Blynk_user2 \
        $(TARGET_DIR)/usr/bin/Blynk_user2
    
    # Cài đặt script khởi động (S99 để nó chạy sau cùng)
    $(INSTALL) -D -m 0755 $(BR2_EXTERNAL_BLYNK_USER2_PATH)/S99blynk \
        $(TARGET_DIR)/etc/init.d/S99blynk
endef
- Build lại

- Build lại package: make blynk_user2-rebuild

- Đóng gói Image: make

- Flash thẻ nhớ và cắm nguồn cho BeagleBone Black.

- Tắt tạm thời:
```bash
/etc/init.d/S99blynk stop
```
- Bật lại chương trình:
```bash
/etc/init.d/S99blynk start
```


