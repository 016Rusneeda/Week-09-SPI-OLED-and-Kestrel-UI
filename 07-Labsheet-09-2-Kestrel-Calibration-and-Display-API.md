# ใบงานการทดลองที่ 9.2 (Lab 9.2)
### การพัฒนาเอนจินปรับเทียบเซนเซอร์และ API ควบคุมการแสดงผลบน Kestrel Web Server พร้อมการพิสูจน์หลักฐานเครือข่าย (HTTP Payload Forensics)

> **คำชี้แจง:** ในใบงานนี้ นักศึกษาจะได้ต่อยอดเว็บเซิร์ฟเวอร์ Kestrel (.NET 8 Minimal API) จากสัปดาห์ที่ 8 โดยเพิ่มบริการ **Sensor Calibration Engine** เพื่อแปลงค่าแอนะล็อกดิบจาก Potentiometer ให้อยู่ในสเกลมาตรฐาน และสร้าง **Display Control API** เพื่อให้ผู้ใช้งานสามารถส่งข้อความจากหน้าเว็บย้อนกลับไปสั่งการหน้าจอ OLED ได้ พร้อมทั้งฝึกกระบวนการ **HTTP Packet & Payload Forensics** เพื่อตรวจสอบความถูกต้องของข้อมูลที่วิ่งบนเครือข่าย

---

## 1. วัตถุประสงค์การทดลอง (Objectives)
1. เข้าใจหลักการและสามารถเขียนเอนจินคำนวณการปรับเทียบเซนเซอร์เชิงเส้นแบบสองจุด (**Two-Point Linear Calibration**) ในภาษา C# ได้
2. สามารถสร้าง Minimal API Endpoint สำหรับการรับพารามิเตอร์ Calibrate (`POST /api/potentiometer/calibrate`) ได้
3. สามารถสร้าง Endpoint สำหรับส่งข้อความสั่งการหน้าจอ OLED (`POST /api/oled/message`) ได้
4. สามารถตรวจสอบและตรวจพิสูจน์หลักฐานแพ็กเก็ตเครือข่าย (**HTTP Request/Response Forensics**) ผ่าน `curl` หรือ Browser DevTools ได้
5. เข้าใจและป้องกันข้อผิดพลาดทางคณิตศาสตร์ (เช่น การหารด้วยศูนย์ Divide-by-Zero และ Out-of-Bounds Clamping) ในบริการ IoT

---

## 2. โครงสร้าง REST Minimal API ที่ต้องสร้าง

```
[ Kestrel Web Server (Port 5000) ]
  ├── GET  /api/telemetry                : สตรีมค่า Raw ADC, Calibrated Value, และข้อความสถานะปัจจุบัน
  ├── POST /api/potentiometer/calibrate  : ปรับเทียบค่าศูนย์และค่าเต็มสเกล (Zero & Span Calibration)
  └── POST /api/oled/message             : สั่งส่งข้อความ Broadcast ไปยังหน้าจอ OLED ทางกายภาพ
```

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 2.1: การสร้างโมเดลและบริการปรับเทียบ (Calibration Service)

สร้างคลาส `CalibrationService.cs` ในโปรเจกต์ Kestrel:

```csharp
public class CalibrationSettings
{
    public int RawMin { get; set; } = 150;     // ค่าดิบต่ำสุด (Zero Point)
    public int RawMax { get; set; } = 3950;    // ค่าดิบสูงสุด (Span Point)
    public double ScaleMin { get; set; } = 0.0;
    public double ScaleMax { get; set; } = 100.0;
    public string Unit { get; set; } = "%";
}

public class CalibrationService
{
    private CalibrationSettings _settings = new();
    private string _currentOledMessage = "SYSTEM READY";

    public CalibrationSettings Settings => _settings;
    public string CurrentOledMessage => _currentOledMessage;

    public void UpdateSettings(CalibrationSettings newSettings)
    {
        // ป้องกันข้อผิดพลาดการหารด้วยศูนย์
        if (newSettings.RawMax <= newSettings.RawMin)
        {
            throw new ArgumentException("RawMax ต้องมีค่ามากกว่า RawMin เสมอ!");
        }
        _settings = newSettings;
    }

    public void SetOledMessage(string msg)
    {
        _currentOledMessage = msg.Length > 20 ? msg[..20] : msg;
    }

    public double Compute(int rawAdc)
    {
        // Clamp ค่าให้อยู่ในช่วงที่กำหนด ป้องกันสเกลทะลัก
        int clamped = Math.Clamp(rawAdc, _settings.RawMin, _settings.RawMax);
        return ((double)(clamped - _settings.RawMin) / (_settings.RawMax - _settings.RawMin)) 
               * (_settings.ScaleMax - _settings.ScaleMin) + _settings.ScaleMin;
    }
}
```

---

### กิจกรรมที่ 2.2: การผูก Minimal API Routes (Program.cs)

ลงทะเบียน Route บน Kestrel เพื่อรองรับการเรียกใช้งาน:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<CalibrationService>();
var app = builder.Build();

// Route 1: อ่านข้อมูล Telemetry
app.MapGet("/api/telemetry", (CalibrationService cal) =>
{
    int simulatedRaw = 2048; // หรือดึงจาก Serial Stream
    double calibrated = cal.Compute(simulatedRaw);
    return Results.Ok(new
    {
        raw = simulatedRaw,
        calibrated = Math.Round(calibrated, 1),
        unit = cal.Settings.Unit,
        displayMsg = cal.CurrentOledMessage,
        timestamp = DateTime.UtcNow
    });
});

// Route 2: ปรับเทียบเซนเซอร์
app.MapPost("/api/potentiometer/calibrate", (CalibrationSettings newSettings, CalibrationService cal) =>
{
    try
    {
        cal.UpdateSettings(newSettings);
        cal.SetOledMessage("CALIBRATED OK");
        return Results.Ok(new { status = "success", settings = cal.Settings });
    }
    catch (ArgumentException ex)
    {
        return Results.BadRequest(new { status = "error", message = ex.Message });
    }
});

// Route 3: สั่งข้อความขึ้นหน้าจอ OLED
app.MapPost("/api/oled/message", (DisplayMessageRequest req, CalibrationService cal) =>
{
    if (string.IsNullOrWhiteSpace(req.Message))
    {
        return Results.BadRequest(new { status = "error", message = "ข้อความต้องไม่ว่างเปล่า" });
    }
    cal.SetOledMessage(req.Message);
    return Results.Ok(new { status = "success", current = cal.CurrentOledMessage });
});

app.Run();

public record DisplayMessageRequest(string Message);
```

---

## 4. ขั้นตอนการตรวจสอบเชิงนิติวิทยาศาสตร์ (HTTP Payload Forensics)

### กิจกรรมนิติวิทยาศาสตร์ 2.1: ตรวจชันสูตรการปรับเทียบและ Headers ด้วย curl
เปิด PowerShell หรือ Terminal เพื่อทดสอบและตรวจดู Raw HTTP Response Headers:

```powershell
# 1. ตรวจสอบ Telemetry ปัจจุบัน
curl.exe -i -X GET http://localhost:5000/api/telemetry

# 2. ทำการ Calibrate ใหม่ (ตั้งช่วง 200 ถึง 3800 สเกล 0 ถึง 1000 RPM)
curl.exe -i -X POST http://localhost:5000/api/potentiometer/calibrate `
  -H "Content-Type: application/json" `
  -d '{"rawMin": 200, "rawMax": 3800, "scaleMin": 0, "scaleMax": 1000, "unit": "RPM"}'
```

ให้นักศึกษาบันทึกค่า:
- **HTTP Status Code:** ได้ `200 OK` หรือไม่?
- **Content-Type:** ตรวจสอบว่าเป็น `application/json; charset=utf-8` หรือไม่?

### กิจกรรมนิติวิทยาศาสตร์ 2.2: Fault Injection & Vulnerability Probe (การจงใจทดสอบส่งค่าวิกฤต)
ทดสอบส่งค่าที่อาจทำให้เกิด Bug เช่น:
- ส่งค่า `rawMin: 4000` และ `rawMax: 1000` (Max น้อยกว่า Min)
- ส่งข้อความว่างเปล่า `""` ไปที่ `/api/oled/message`

บันทึกผลว่าเซิร์ฟเวอร์ตอบสนองด้วยรหัส **`400 Bad Request`** พร้อมข้อความแจ้งเตือนที่ปลอดภัย โดยที่เซิร์ฟเวอร์ Kestrel **ไม่เกิดการ Crash** ใช่หรือไม่!

---

## 5. คำถามท้ายการทดลองเพื่อการประเมินผล
1. เหตุใดการคำนวณสเกลเซนเซอร์จึงควรทำที่ฝั่ง Kestrel Server แทนที่จะคำนวณบนไมโครคอนโทรลเลอร์ ESP32 ตั้งแต่แรก?
2. จากการทำ HTTP Forensics หากไม่มีการตรวจสอบเงื่อนไข `RawMax <= RawMin` ในโค้ด จะเกิด Exception ชนิดใดขึ้นในภาษา C#?
