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
driver_led/
│
├── led_driver.c
├── Makefile
└── README.md
```
## 3. CÁC BƯỚC LÀM
### Bước 1: Tạo thư mục làm bài
```bash
cd ~
mkdir led_driver
cd led_driver
```

### Bước 2: Tạo file driver
```bash
nano led_driver.c"
```
- Dán code sau:
```bash
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "myled"

static int major;
static char buffer[10];

static ssize_t my_read(struct file *file, char *user_buf, size_t len, loff_t *off)
{
    copy_to_user(user_buf, buffer, strlen(buffer));
    return strlen(buffer);
}

static ssize_t my_write(struct file *file, const char *user_buf, size_t len, loff_t *off)
{
    copy_from_user(buffer, user_buf, len);

    if(buffer[0] == '1')
        printk("LED ON\n");

    if(buffer[0] == '0')
        printk("LED OFF\n");

    return len;
}

static struct file_operations fops =
{
    .read = my_read,
    .write = my_write
};

static int __init my_init(void)
{
    major = register_chrdev(0, DEVICE_NAME, &fops);
    printk("Driver loaded Major=%d\n", major);
    return 0;
}

static void __exit my_exit(void)
{
    unregister_chrdev(major, DEVICE_NAME);
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
```
- Lưu:
```bash
CTRL + O
ENTER
CTRL + X
```
### Bước 3: Tạo Makefile
```bash
nano Makefile
```
```bash
obj-m += led_driver.o

KDIR = /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### Bước 4: Complie Driver
```bash
make
```
- nếu thành công sẽ có:
```bash
led_driver.ko
```
- kiểm tra:
```bash
ls
```

### Bước 5: Load driver
```bash
sudo insmod led_driver.ko
```
- kiểm tra:
```bash
dmesg
```
- sẽ thấy:
```bash
Driver loaded Major = xxx
```
- Ở đây sau khi build xong thì nhận đc  Driver loaded Major = 237

### Bước 6: Tạo file device
```bash
sudo mknod /dev/myled c 237 0
```

### Bước 7: Gửi lệnh từ user space
- Test trên giả lập QEMU
- Bật LED
```bash
echo 1 > /dev/myled
```
- Tắt LED
```bash
echo 0 > /dev/myled
```
- Kết quả thu được ảnh dưới đây:
