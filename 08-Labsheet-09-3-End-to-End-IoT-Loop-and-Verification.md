# ใบงานการทดลองที่ 9.3 (Lab 9.3)
### การรวมระบบวงปิดแบบครบวงจร การตรวจสอบความสอดคล้องของข้อมูลและเวลาหน่วง 

> **คำชี้แจง:** ในใบงานนี้ นักศึกษาจะได้นำชิ้นส่วนทั้งหมดมารวมร่างกันเป็นระบบ IoT วงปิดแบบสมบูรณ์: **ตัวต้านทานปรับค่าได้ (Potentiometer) $\rightarrow$ ESP32 ADC1 $\rightarrow$ Kestrel Server (.NET 8) $\rightarrow$ หน้าจอ OLED ทางกายภาพ + เว็บแดชบอร์ด** และทำกิจกรรมทดสอบเวลาหน่วงในการทำงานของระบบโดยรวม เพื่อพิสูจน์ว่าค่าที่ปรากฏบนจอภาพจริงตรงกับค่าบนหน้าเว็บแบบ Real-time แบบ real-time หรือไม่

---
## 1. วัตถุประสงค์การทดลอง (Objectives)
1. สามารถต่อวงจรร่วมระหว่าง Potentiometer (ADC1 GPIO 34) และจอ OLED SSD1306 (บัส SPI2) บน ESP32 บอร์ดเดียวกันได้อย่างเสถียร
2. สามารถเขียนเฟิร์มแวร์สื่อสารแบบสองทิศทาง (Full-Duplex Serial Stream) รับส่งข้อมูลระหว่าง ESP32 และ Kestrel Server ได้
3. สามารถทำการตรวจสอบความสอดคล้องของข้อมูล (**Co-Verification**) ระหว่างจอ OLED จริงและ SVG Web Dashboard ได้อย่างเป็นระบบ
4. สามารถตรวจวัดและวิเคราะห์ความหน่วงเวลาของระบบ (**End-to-End Latency Forensics**) ระหว่างฮาร์ดแวร์และเว็บเบราว์เซอร์ได้

---

## 2. แผนผังระบบ

ระบบประกอบด้วยส่วนของ hardware และ software ดังปรากฏในหัวข้อ 2.1 
### 2.1 บล็อกไดอะแกรม

<p align = "center">

<img src = Images/Closed-Loop%20System%20Flow.svg>

</p>

Potentiometer ควรต่อที่ขา ADC 1 เนื่องจาก ADC 2 นั้นจะถูกใช้สำหรับ ESP32 เพื่อ Wi-Fi และ Bluetooth

### 2.2 การเชื่อมต่อ ESP32 กับ OLED

<p align = "center">

<img src = Images/Lab9-1-connection.svg>

</p>

### 2.2 การออกแบบหน้าจอ

<p align = "center">

<img src = "Images/Display_Zone_assignment.svg" width="450">

</p>

---

## 3. ขั้นตอนการทดลอง

### กิจกรรม 3.1  เฟิร์มแวร์ ESP32 จำลองการทำงานของระบบ

เขียนโค้ดใน FreeRTOS Task ให้ทำหน้าที่ 2 ประการพร้อมกัน:
1. จำลองอ่านค่า ADC จาก GPIO 34 ทุกๆ 50 ms แล้วสตรีมออก Serial: `ADC:2048\n` (ดูผลทาง terminal)
2. เขียนหน้าจอ โดยแบ่งเป็น 3 โซน (Zone 1 Status, Zone2 Bar gauge, Zone 3 แสดงโหมดการทำงาน ว่าเป็น edge  หรือ cloud computing)
3. ดักฟังคำสั่งจาก Serial loopback (จำลองค่าที่ Kestrel ส่งกลับมา เช่น `SET:50:CALIBRATED OK\n` แล้วนำค่าเปอร์เซ็นต์และข้อความไปวาดลงจอ OLED

### กิจกรรม 3.2 Kestrel Server รับ คำนวณ และส่งกลับค่าไปยัง ESP32

สร้าง kestrel project
1. ติดตั้งและรับค่าจาก serial port
2. คำนวณค่า Raw เป็นเปอร์เซนต์ (เตรียมการ calibrate  ด้วย)
3. update ค่า Raw และเปอร์เซนต์ที่ได้จาก ESP32 บน bar gauge บนหน้าเว็บ
4. ส่งค่ากลับไปยัง ESP32 ถ้า ESP32 ได้ข้อมูลจาก Kestrel ให้แสดงที่ Zone 3 ว่า cloud computing, ถ้าไม่ได้ ให้แสดง edge computing

---

### กิจกรรมที่ 3.1: เฟิร์มแวร์ ESP32 สื่อสารสองทิศทาง (Two-Way Serial Bridge)

เขียนโค้ดใน FreeRTOS Task ให้ทำหน้าที่ 2 ประการพร้อมกัน:
1. อ่านค่า ADC จาก GPIO 34 ทุกๆ 50 ms แล้วสตรีมออก Serial: `ADC:2048\n`
2. ดักฟังคำสั่งจาก Serial ที่ Kestrel ส่งกลับมา เช่น `SET:50:CALIBRATED OK\n` แล้วนำค่าเปอร์เซ็นต์และข้อความไปวาดลงจอ OLED

```c
// ตัวอย่างลูปหลักใน main.c
void app_main(void)
{
    oled_spi_init();
    oled_init_display();
    adc_oneshot_unit_handle_t adc1_handle = init_potentiometer_adc();

    int raw_val = 0;
    char rx_buffer[64];

    while (1) {
        // 1. อ่านค่า ADC
        adc_oneshot_read(adc1_handle, ADC_CHANNEL_6, &raw_val);
        
        // 2. สตรีมค่าขึ้น Kestrel ทาง Serial
        printf("ADC:%d\n", raw_val);

        // 3. ตรวจสอบว่ามีข้อมูลตอบกลับจาก Kestrel หรือไม่
        if (read_serial_line(rx_buffer, sizeof(rx_buffer))) {
            // ถอดรหัสคำสั่ง เช่น "50,NORMAL"
            int percent = 0;
            char msg[32] = {0};
            if (sscanf(rx_buffer, "%d,%31s", &percent, msg) == 2) {
                render_multizone_ui(percent, raw_val, msg);
            }
        }

        vTaskDelay(pdMS_TO_TICKS(50)); // รันที่อัตรา 20 Hz
    }
}
```

---
## 4. การตรวจวัดความหน่วงเวลา

### กิจกรรม 4.1 การวัด End-to-End Latency  ในโหมด simulation
1. สั่งรันคำสั่งจับเวลาหรือใส่ Timestamp ในระดับมิลลิวินาที (Unix Timestamp ms) ทั้งใน ESP32 และใน Kestrel Background Service
2. บันทึกเวลาตั้งแต่ **จังหวะที่หมุน Potentiometer ($T_0$)** $\rightarrow$ **Kestrel รับข้อมูล ($T_1$)** $\rightarrow$ **หน้าจอ OLED อัปเดตเสร็จสิ้น ($T_2$)**
3. คำนวณหาค่าความหน่วงเฉลี่ย
   $$\Delta T = T_2 - T_0$$
4. วิเคราะห์ว่าความหน่วงของระบบทั้งหมดต่ำกว่า **100 ms** หรือไม่ ซึ่งเป็นเกณฑ์มาตรฐานของการตอบสนองแบบ Real-time ทางกายภาพ

---

## 5. คำถามท้ายการทดลองเพื่อการประเมินผล
1. หากการแสดงผลบนหน้าจอ OLED มีความล่าช้า (Lag) กว่าหน้าเว็บอย่างเห็นได้ชัด ความล่าช้านั้นน่าจะเกิดจากจุดคอขวด (Bottleneck) ใดในระบบ?
2. เหตุใดการใช้บัส SPI 10 MHz จึงช่วยลดความหน่วงเวลาในการอัปเดตหน้าจอ OLED ได้ดีกว่าการใช้บัส I2C 100 kHz ในระบบ Closed-Loop เช่นนี้?
