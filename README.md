# BÀI TẬP HỆ ĐIỀU HÀNH NHÚNG TUẦN 6 ỨNG DỤNG TỔNG HỢP
## 1. MỤC TIÊU
- Xây dựng một Linux Device Driver để mô phỏng điều khiển LED
- Người dùng giao tiếp thông qua file:
```bash
/dev/myled
```
- Ghi dữ liệu vào device:
```bash
1 → LED ON
0 → LED OFF
```
## 2. CẤU TRÚC THƯ MỤC
```bash
package/
└── led-driver/
    ├── Config.in
    ├── led-driver.mk
    └── src/
        ├── led_driver.c
        └── Makefile
```
## 3. CÁC BƯỚC LÀM
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
## BƯỚC 8: KIỂM TRA SAU BUILD
```bash
ls output/target/lib/modules
```
- Phải có:
```bash
led_driver.ko
```
## BƯỚC 9: TEST KIỂM THỬ TRÊN BBB
- Trong quá trình làm thì nhóm có sửa code và quy ước thành:
```bash
0: LED OFF
1: LED ON
```
- Thu được kết quả dưới đây:
```bash
LED ON
```
<img width="2560" height="1920" alt="image" src="https://github.com/user-attachments/assets/ac2ea323-0a47-499c-896c-0b45c2c841ee" />
<img width="1920" height="2560" alt="image" src="https://github.com/user-attachments/assets/13eb01c8-fe56-4bcc-9f87-eb68705c9400" />

```bash


LED OFF
```
<img width="2560" height="1920" alt="image" src="https://github.com/user-attachments/assets/197dd33c-6998-4c59-8261-dcbcbb7fdc6d" />
<img width="2560" height="1920" alt="image" src="https://github.com/user-attachments/assets/72a24eb7-5d25-4f1c-ab83-f98ae00042b1" />




