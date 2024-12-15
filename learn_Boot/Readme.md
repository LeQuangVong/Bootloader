#### Hoạt động của BootLoader
##### Khởi động hệ thống 
- Sau khi xong các bước startup và linker
- Chương trình sẽ vào hàm main()
- Cấu hình CLock hệ thống.
- Cấu hình GPIO
- Khởi tạo thư viện
##### Kiểm tra vòng lặp và điều kiện
```
while (1)
{
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        bootloader();
    }
}
```
##### Hàm bootloader
![1](/learn_Boot/1.png)
![2](/learn_Boot/2.png)

Vậy chương trình bootloader sẽ được nạp ở địa chỉ 0x80000000 trong Flash memory
Còn chương trình ứng dụng sẽ được nạp ở địa chỉ 0x80004000 trong bộ nhớ Flash.
```
uint32_t MY_APP = 0x08004000;
```

- Tắt SysTick Timer: vì để tránh xung đột hoặc lỗi khi ứng dụng chính khởi động.
```
SysTick->CTRL = 0x0;
SysTick->LOAD = 0;
```

- Dừng HAL: đảm bảo khi ứng dụng chính bắt đầu với trạng thái hệ thống sạch.
```
HAL_DeInit();
```

- Cấu hình lại Main Stack Pointer (MSP).
    - Lấy giá trị tại địa chỉ đầu tiên của bảng VECTOR của ứng dụng 
    - Sử dụng nó để thiết lập Main Stack Pointer (MSP).
```
__set_MSP(*(uint32_t*)MY_APP);

__STATIC_FORCEINLINE void __set_MSP(uint32_t topOfMainStack)
{
  __ASM volatile ("MSR msp, %0" : : "r" (topOfMainStack) : );
}
```

- Cấu hình bảng VECTOR ngắt:
    - Ghi vào bit VTOR của thanh ghi SCB để thay đổi bảng VECTOR ngắt của hệ thống từ bootloader sang ứng dụng.
    - Việc cấu hình giúp cho CPU biết nơi tìm các vector ngắt khi ứng dụng hoạt động.
```
__DMB();
SCB->VTOR = MY_APP;
__DMB();
```
- Lấy địa chỉ reset_handler của ứng dụng
    - Giá trị MSP cộng thêm 4 byte sẽ là địa chỉ của reset_handler (hàm khởi động chính của ứng dụng).
    - Gán địa chỉ này cho con trỏ hàm reset_handler.

```
uint32_t JumpAddress = *((volatile uint32_t *) (MY_APP + 4));
void (*reset_handler)(void) = (void(*)(void))JumpAddress;
```

-  Gọi hàm reset_handler để chuyển quyền điều khiển từ bootloader sang ứng dụng, từ đây ứng dụng sẽ hoạt động như bình thường bắt đầu từ hàm main của nó.
```
reset_handler();
```
- Hàm blinkled trong chương trình chính của ứng dụng.
```
while (1)
{
/* USER CODE END WHILE */

/* USER CODE BEGIN 3 */
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    HAL_Delay(1000);
}
```
