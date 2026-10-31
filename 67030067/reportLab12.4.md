
# ใบงานการทดลองที่ 12.4
## การทดสอบผสานรวมระบบครบวงจร (ESP32 Hardware <-> Mosquitto Broker <-> Flutter Mobile App)

> [!NOTE]
> **คำชี้แจง**
>
> ในการทดลองนี้ นักศึกษาจะนำส่วนประกอบที่พัฒนามาจากสัปดาห์ที่ 11 และสัปดาห์ที่ 12 มาผสานรวมเป็นระบบ IoT แบบครบวงจร (Complete End-to-End IoT Ecosystem) โดยให้บอร์ด ESP32 สื่อสารกับแอปพลิเคชัน Flutter ผ่าน Mosquitto MQTT Broker
>
> นักศึกษาจะทดสอบการหมุน Potentiometer บนบอร์ดจริงเพื่อส่งค่ามาแสดงบนหน้าจอมือถือแบบเรียลไทม์ และทดสอบการกดสวิตช์บนแอปพลิเคชันเพื่อสั่งเปิด-ปิดหลอดไฟ LED ที่เชื่อมต่อกับบอร์ด ESP32

---

## 1. วัตถุประสงค์การทดลอง

1. สามารถผสานรวมหน้าจอควบคุม Actuator และหน้าจอแสดงผล Telemetry เข้าเป็นแอปพลิเคชันเดียวที่มีหลายแท็บ (Unified Multi-tab IoT App) ได้
2. สามารถเชื่อมต่อบอร์ด ESP32 เข้ากับ Mosquitto Broker และสื่อสารกับแอปพลิเคชัน Flutter ได้
3. สามารถวัดและวิเคราะห์ความหน่วงเวลาในการตอบสนองแบบไป-กลับ (Round-Trip Time: RTT) ของคำสั่งควบคุมได้
4. สามารถทดสอบการทำงานของระบบเมื่อเครือข่ายหรืออุปกรณ์เกิดความผิดพลาด (Fault-Tolerance & LWT Handling) ได้

---

## 2. แผนผังการทดสอบระบบและโครงสร้าง MQTT Topic

<p align="center">
  <img src="Images/Lab12.4-1_academic.svg" alt="End-to-End IoT Pipeline">
</p>

### 2.1 โครงสร้าง MQTT Topic

| ชื่อหัวข้อ (Topic) | ทิศทางการส่ง | ชนิดข้อมูล (Payload Format) | วัตถุประสงค์ |
| :--- | :--- | :--- | :--- |
| `iot/esp32/status` | ESP32 -> Broker (LWT) | Plaintext: `"online"` หรือ `"offline"` | แจ้งสถานะการทำงานของบอร์ด โดยใช้ Retained Message |
| `iot/esp32/control` | Flutter -> ESP32 | JSON: `{"led": "ON"}` หรือ `{"led": "OFF"}` | ส่งคำสั่งเปิด-ปิดหลอดไฟ LED |
| `iot/esp32/led_ack` | ESP32 -> Flutter | JSON: `{"led_state": true, "ack": true}` | ส่งข้อความยืนยันสถานะหลอดไฟกลับมายังแอป |
| `iot/esp32/telemetry` | ESP32 -> Flutter | JSON: `{"voltage": 2.50, "raw": 3102, "uptime": 10}` | ส่งค่าแรงดันจาก Potentiometer และเวลา Uptime |

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-4.1 การเตรียมโปรเจกต์และสร้างแอปพลิเคชันรวมศูนย์

#### 1. คัดลอกโปรเจกต์และรวบรวมไฟล์

นำหน้าจอ `actuator_control_screen.dart` จาก Lab 12-2 และหน้าจอ `telemetry_dashboard_screen.dart` รวมถึง `radial_gauge_widget.dart` จาก Lab 12-3 มารวมไว้ในโปรเจกต์เดียวกัน

**คำสั่ง PowerShell**

```powershell
# 1. คัดลอกโปรเจกต์จาก Lab12-3 มาเป็น Lab12-4
Copy-Item -Recurse -Path "..\Lab12-3\iot_telemetry_gauge" -Destination "..\Lab12-4\iot_unified_dashboard"

# 2. คัดลอกหน้าจอ Actuator Control จาก Lab12-2
Copy-Item -Path "..\Lab12-2\iot_actuator_switch\lib\actuator_control_screen.dart" -Destination "..\Lab12-4\iot_unified_dashboard\lib\"

# 3. เข้าไปยังโฟลเดอร์โปรเจกต์ใหม่
cd ..\Lab12-4\iot_unified_dashboard

# 4. ล้างไฟล์ที่เกิดจากการคอมไพล์ก่อนหน้า
flutter clean

# 5. ดาวน์โหลด Dependencies
flutter pub get
```

> **หมายเหตุ:** คำสั่งคัดลอกข้างต้นอ้างอิงโครงสร้างโฟลเดอร์ที่กำหนด หากชื่อโฟลเดอร์หรือที่อยู่ไฟล์ของนักศึกษาแตกต่างกัน ให้ปรับ Path ให้ตรงกับตำแหน่งจริง

ตรวจสอบให้แน่ใจว่าภายในโฟลเดอร์ `lib/` มีไฟล์อย่างน้อยดังนี้

```text
lib/
├── main.dart
├── actuator_control_screen.dart
├── telemetry_dashboard_screen.dart
└── radial_gauge_widget.dart
```

#### 2. ปรับปรุงโค้ดรวมศูนย์แอปพลิเคชัน (`lib/main.dart`)

เปิดไฟล์ `lib/main.dart` แล้วแทนที่ด้วยโค้ดต่อไปนี้ โดยใช้ `IndexedStack` ร่วมกับ `NavigationBar` เพื่อรักษาสถานะของหน้าจอทั้งสองแท็บเมื่อมีการสลับหน้า

```dart
import 'package:flutter/material.dart';
import 'actuator_control_screen.dart';
import 'telemetry_dashboard_screen.dart';

void main() {
  runApp(const UnifiedIotApp());
}

class UnifiedIotApp extends StatelessWidget {
  const UnifiedIotApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ESP32 Smart IoT Commander',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),
      home: const MainNavigationShell(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class MainNavigationShell extends StatefulWidget {
  const MainNavigationShell({super.key});

  @override
  State<MainNavigationShell> createState() =>
      _MainNavigationShellState();
}

class _MainNavigationShellState extends State<MainNavigationShell> {
  int _currentIndex = 0;

  // กำหนด IP ของเครื่องที่ติดตั้ง Mosquitto Broker
  // Windows Desktop: ใช้ 127.0.0.1 เมื่อ Broker อยู่เครื่องเดียวกัน
  // Android Emulator: โดยทั่วไปใช้ 10.0.2.2 เมื่อ Broker อยู่บนเครื่อง Host
  // โทรศัพท์จริง: ใช้ IPv4 ของคอมพิวเตอร์ในเครือข่ายเดียวกัน
  final String brokerIp = '127.0.0.1';

  late final List<Widget> _pages;

  @override
  void initState() {
    super.initState();

    _pages = [
      TelemetryDashboardScreen(brokerIp: brokerIp),
      ActuatorControlScreen(brokerIp: brokerIp),
    ];
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      // IndexedStack ช่วยรักษาสถานะของแต่ละหน้าจอ
      body: IndexedStack(
        index: _currentIndex,
        children: _pages,
      ),

      bottomNavigationBar: NavigationBar(
        selectedIndex: _currentIndex,
        onDestinationSelected: (index) {
          setState(() {
            _currentIndex = index;
          });
        },
        backgroundColor: const Color(0xFF1E293B),
        indicatorColor: const Color(0xFF00E5FF).withOpacity(0.2),
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.speed),
            selectedIcon: Icon(
              Icons.speed,
              color: Color(0xFF00E5FF),
            ),
            label: 'Telemetry Gauge',
          ),
          NavigationDestination(
            icon: Icon(Icons.toggle_on_outlined),
            selectedIcon: Icon(
              Icons.toggle_on,
              color: Color(0xFF00E676),
            ),
            label: 'Actuator Switch',
          ),
        ],
      ),
    );
  }
}
```

**ข้อควรตรวจสอบ**

- ชื่อคลาส `TelemetryDashboardScreen` และ `ActuatorControlScreen` ต้องตรงกับชื่อคลาสในไฟล์ที่นำมาจาก Lab ก่อนหน้า
- ทั้งสองหน้าจอต้องรองรับการรับค่า `brokerIp` ผ่าน Constructor
- ต้องติดตั้ง MQTT Package ตามที่ใช้ในโปรเจกต์เดิม

#### 3. สั่งรัน Flutter App

```powershell
# รันบน Windows Desktop
flutter run -d windows

# ตรวจสอบอุปกรณ์ที่เชื่อมต่อ
flutter devices

# รันบนอุปกรณ์ Android โดยระบุ Device ID
flutter run -d <device-id>
```

> [!TIP]
> **การกำหนด Broker IP เมื่อทดสอบกับ ESP32 หรือโทรศัพท์จริง**
>
> 1. ตรวจสอบ IPv4 ของคอมพิวเตอร์ด้วยคำสั่ง `ipconfig`
> 2. ตัวอย่าง IP ของคอมพิวเตอร์คือ `192.168.1.105`
> 3. แก้ไข `brokerIp` ในไฟล์ `main.dart` ให้เป็น IP ดังกล่าว เมื่อใช้งานผ่านโทรศัพท์จริง
> 4. ตรวจสอบไฟล์ `mosquitto.conf` ให้มีการกำหนด Listener ที่อนุญาตให้เชื่อมต่อจากอุปกรณ์ในเครือข่าย เช่น

```conf
listener 1883 0.0.0.0
allow_anonymous true
```

> 5. เปิด Mosquitto Broker ด้วยคำสั่ง

```powershell
mosquitto -c mosquitto.conf -v
```

> **ข้อควรระวัง:** การตั้งค่า `allow_anonymous true` เหมาะสำหรับการทดลองในเครือข่ายปิดเท่านั้น ไม่ควรนำไปใช้กับระบบจริงที่เปิดให้เข้าถึงจากอินเทอร์เน็ต

---

### กิจกรรมที่ 12-4.2 การเตรียมและเริ่มทำงานของ ESP32 หรือระบบจำลองฮาร์ดแวร์

ในการทดลองนี้สามารถเลือกดำเนินการได้ 2 รูปแบบ

- **ทางเลือก A (Hardware Mode):** ใช้บอร์ด ESP32 จริง ต่อ Potentiometer และ LED บน Breadboard
- **ทางเลือก B (Mock Simulation Mode):** ใช้สคริปต์ PowerShell จำลองการทำงานของ ESP32 ในกรณีที่ไม่มีบอร์ดจริง

> [!IMPORTANT]
> **ตรวจสอบ Mosquitto Broker ก่อนเริ่มทดลอง**
>
> ต้องเปิด Mosquitto MQTT Broker ให้ทำงานก่อนเริ่มรัน Flutter App หรือสคริปต์จำลอง หาก Broker ยังไม่ทำงาน อุปกรณ์จะไม่สามารถเชื่อมต่อได้ และอาจพบข้อความแจ้งเตือนว่า
>
> `Error: No connection could be made because the target machine actively refused it.`

เปิด PowerShell อีกหน้าต่างหนึ่ง แล้วสั่งรัน

```powershell
mosquitto -c mosquitto.conf -v
```

---

#### ทางเลือก A: การใช้งานฮาร์ดแวร์จริง (Physical ESP32 Board)

##### 1. การต่อวงจรบน Breadboard

**Potentiometer (ตัวต้านทานปรับค่าได้)**

- ขาริมซ้าย: ต่อเข้าขา `GND` ของ ESP32
- ขาริมขวา: ต่อเข้าขา `3.3V` ของ ESP32
- ขากลาง: ต่อเข้าขา `GPIO 34` (ADC1_CH6 สำหรับ ESP32 รุ่นดั้งเดิม)

**LED (หลอดไฟแสดงสถานะ)**

- ขา Anode (+): ต่อเข้าขา `GPIO 2` ผ่านตัวต้านทาน `330Ω` หรือ `470Ω`
- ขา Cathode (-): ต่อเข้าขา `GND`

> **หมายเหตุ:** ผังขา GPIO อาจแตกต่างกันตามรุ่นของบอร์ด ESP32 ควรตรวจสอบ Pinout ของบอร์ดที่ใช้งานจริงก่อนต่อวงจร โดย GPIO 34 เป็นขา Input-only ของ ESP32 รุ่นดั้งเดิม จึงเหมาะสำหรับอ่านค่า ADC แต่ไม่สามารถใช้ควบคุม LED ได้

##### 2. การกำหนดค่าและแฟลชเฟิร์มแวร์

1. เปิดโปรเจกต์เฟิร์มแวร์ในโฟลเดอร์ `Example_codes/Lab12-4/esp32_firmware` หรือใช้โปรเจกต์ MQTT Client จาก Lab 11-2
2. เปิดไฟล์ `main/main.c` และกำหนดค่าต่อไปนี้

**การตั้งค่า Wi-Fi และ MQTT**

- กำหนด `WIFI_SSID` ให้ตรงกับชื่อเครือข่าย Wi-Fi
- กำหนด `WIFI_PASS` ให้ตรงกับรหัสผ่าน Wi-Fi
- กำหนด `MQTT_BROKER_URI` ให้เป็น IP ของคอมพิวเตอร์ที่รัน Broker เช่น `mqtt://192.168.1.105:1883`

**การทำงานของเฟิร์มแวร์ที่ต้องมี**

- อ่านค่า Potentiometer จาก `GPIO 34` (`ADC_CHANNEL_6`)
- ส่งค่า Telemetry ไปยัง `iot/esp32/telemetry` ทุก 500 มิลลิวินาที
- ส่งข้อมูลในรูปแบบ JSON เช่น `{"voltage": 2.50, "raw": 3102, "uptime": 10}`
- Subscribe หัวข้อ `iot/esp32/control` เพื่อรับคำสั่งเปิด-ปิด LED
- ควบคุมระดับสัญญาณที่ `GPIO 2` ตามคำสั่ง ON หรือ OFF
- Publish ข้อความยืนยันไปยัง `iot/esp32/led_ack` เช่น `{"led_state": true, "ack": true}`
- กำหนด Last Will and Testament (LWT) ให้ส่งข้อความ `"offline"` ไปยัง `iot/esp32/status` โดยตั้งค่า Retain เป็น `true`
- เมื่อเชื่อมต่อ MQTT สำเร็จ ให้ Publish สถานะ `"online"` ไปยัง `iot/esp32/status`

3. สั่งคอมไพล์และแฟลชเฟิร์มแวร์ลงบอร์ด ESP32

```powershell
idf.py set-target esp32
idf.py flash monitor
```

> **หมายเหตุ:** คำสั่งข้างต้นใช้กับ ESP32 รุ่นดั้งเดิม หากใช้บอร์ดรุ่นอื่น เช่น ESP32-C3 หรือ ESP32-C6 ต้องเปลี่ยน Target ให้ตรงกับรุ่นบอร์ด และตรวจสอบหมายเลข GPIO/ADC ใหม่ด้วย

---

#### ทางเลือก B: การจำลองพฤติกรรม ESP32 ด้วย PowerShell (Mock ESP32 Simulator)

สำหรับนักศึกษาที่ไม่มีบอร์ด ESP32 จริง สามารถใช้สคริปต์ `mock_esp32.ps1` เพื่อจำลองการทำงานของระบบได้

##### พฤติกรรมที่สคริปต์จำลอง

1. **ตรวจสอบ Broker:** ตรวจสอบการเชื่อมต่อกับ MQTT Broker ที่พอร์ต 1883 ก่อนเริ่มทำงาน
2. **Online Status:** ส่งข้อความ `"online"` ไปยัง `iot/esp32/status` พร้อม Retain
3. **Telemetry Streaming:** ส่งข้อมูลจำลองเป็นคลื่น Sine Wave เพื่อเลียนแบบการหมุน Potentiometer ระหว่าง 0.15V ถึง 3.15V ทุก 500 มิลลิวินาที
4. **Actuator Control:** รับคำสั่งจาก `iot/esp32/control` และส่ง ACK กลับไปยัง `iot/esp32/led_ack`
5. **Offline Status:** เมื่อหยุดสคริปต์ด้วย `Ctrl + C` จะส่งข้อความ `"offline"` ไปยัง `iot/esp32/status` เพื่อจำลองการหยุดทำงานของบอร์ด

> [!WARNING]
> การส่ง `"offline"` ในบล็อก `finally` เป็นการส่งข้อความด้วยสคริปต์โดยตรง ไม่ใช่การทำงานของ LWT จริง
>
> LWT จริงจะทำงานเมื่ออุปกรณ์ MQTT หลุดการเชื่อมต่อโดยไม่ส่งแพ็กเก็ต Disconnect ตามปกติ ส่วนการหยุดสคริปต์จำลองจะส่งสถานะ Offline ด้วยคำสั่ง Publish เอง

##### ซอร์สโค้ด `mock_esp32.ps1`

สร้างไฟล์ `mock_esp32.ps1` ไว้ในโฟลเดอร์ `Example_codes/Lab12-4/`

```powershell
# mock_esp32.ps1
# สคริปต์จำลองการทำงานของ ESP32 ผ่าน Mosquitto MQTT Broker

param(
    [string]$Broker = "127.0.0.1",
    [int]$Port = 1883
)

Write-Host "==========================================================" -ForegroundColor Cyan
Write-Host "       [MOCK ESP32] ESP32 IoT Simulator                   " -ForegroundColor Cyan
Write-Host "==========================================================" -ForegroundColor Cyan

# ตรวจสอบการเชื่อมต่อกับ MQTT Broker
function Test-BrokerConnection {
    param(
        [string]$Server,
        [int]$TargetPort
    )

    $tcpClient = New-Object System.Net.Sockets.TcpClient

    try {
        $connect = $tcpClient.BeginConnect(
            $Server,
            $TargetPort,
            $null,
            $null
        )

        $success = $connect.AsyncWaitHandle.WaitOne(1000)

        if ($success -and $tcpClient.Connected) {
            $tcpClient.EndConnect($connect)
            return $true
        }

        return $false
    }
    catch {
        return $false
    }
    finally {
        $tcpClient.Close()
    }
}

if (-not (Test-BrokerConnection -Server $Broker -TargetPort $Port)) {
    Write-Host "`n[ERROR] ไม่สามารถเชื่อมต่อ MQTT Broker ได้!" -ForegroundColor Red
    Write-Host "Broker: ${Broker}:${Port}" -ForegroundColor Yellow
    Write-Host "`nกรุณาเปิด Mosquitto Broker ก่อน โดยใช้คำสั่ง:" -ForegroundColor White
    Write-Host "mosquitto -c mosquitto.conf -v" -ForegroundColor Green
    exit 1
}

# ตรวจสอบคำสั่ง Mosquitto
if (-not (Get-Command mosquitto_pub -ErrorAction SilentlyContinue)) {
    Write-Host "[ERROR] ไม่พบคำสั่ง mosquitto_pub ใน PATH" -ForegroundColor Red
    exit 1
}

if (-not (Get-Command mosquitto_sub -ErrorAction SilentlyContinue)) {
    Write-Host "[ERROR] ไม่พบคำสั่ง mosquitto_sub ใน PATH" -ForegroundColor Red
    exit 1
}

# ส่งสถานะ Online
Write-Host "`n[1/3] ส่งสถานะ ONLINE..." -ForegroundColor Green

mosquitto_pub -h $Broker -p $Port `
    -t "iot/esp32/status" `
    -m "online" -r

if ($LASTEXITCODE -ne 0) {
    Write-Host "[ERROR] ส่งสถานะ ONLINE ไม่สำเร็จ" -ForegroundColor Red
    exit 1
}

# เริ่มส่งข้อมูล Telemetry ใน Background Job
$job = Start-Job -ScriptBlock {
    param($b, $p)

    $t = 0

    while ($true) {
        # สร้างค่าแรงดันจำลองแบบ Sine Wave
        $v = [Math]::Round(
            1.65 + 1.5 * [Math]::Sin($t * 0.2),
            2
        )

        # แปลงแรงดันเป็นค่า ADC
        $raw = [int](($v / 3.3) * 4095)

        # สร้าง JSON Payload
        $payload = "{`"voltage`": $v, `"raw`": $raw, `"uptime`": $t}"

        # ส่งข้อมูลไปยัง MQTT Broker
        mosquitto_pub -h $b -p $p `
            -t "iot/esp32/telemetry" `
            -m $payload

        $t++

        Start-Sleep -Milliseconds 500
    }
} -ArgumentList $Broker, $Port

Write-Host "[2/3] เริ่มส่ง Telemetry ทุก 500 ms..." -ForegroundColor Yellow
Write-Host "[3/3] กำลังรอรับคำสั่งจาก Flutter..." -ForegroundColor Magenta

Write-Host "Topic: iot/esp32/control" -ForegroundColor Gray
Write-Host "กด Ctrl+C เพื่อหยุดการจำลอง" -ForegroundColor Gray
Write-Host "----------------------------------------------------------"

try {
    # รับคำสั่งจาก Flutter
    & mosquitto_sub -h $Broker -p $Port `
        -t "iot/esp32/control" |
        ForEach-Object {

            $cmd = $_

            Write-Host "ได้รับคำสั่ง: $cmd" -ForegroundColor White

            # ตรวจสอบคำสั่ง LED จาก JSON
            $ledState = $false

            try {
                $data = $cmd | ConvertFrom-Json

                if ($data.led -eq "ON") {
                    $ledState = $true
                }
                elseif ($data.led -eq "OFF") {
                    $ledState = $false
                }
            }
            catch {
                Write-Host "รูปแบบคำสั่งไม่ถูกต้อง" -ForegroundColor Red
                return
            }

            if ($ledState) {
                Write-Host "LED = ON" -ForegroundColor Green
            }
            else {
                Write-Host "LED = OFF" -ForegroundColor Red
            }

            # สร้าง ACK JSON
            $ackData = @{
                led_state = $ledState
                ack = $true
            } | ConvertTo-Json -Compress

            # ส่ง ACK กลับไปยัง Flutter
            mosquitto_pub -h $Broker -p $Port `
                -t "iot/esp32/led_ack" `
                -m $ackData
        }
}
finally {
    # หยุด Background Job
    if ($job) {
        Stop-Job $job -ErrorAction SilentlyContinue
        Remove-Job $job -ErrorAction SilentlyContinue
    }

    # ส่งสถานะ Offline เมื่อหยุดสคริปต์
    Write-Host "`n[STOP] ส่งสถานะ OFFLINE..." -ForegroundColor Yellow

    mosquitto_pub -h $Broker -p $Port `
        -t "iot/esp32/status" `
        -m "offline" -r

    Write-Host "หยุดการจำลองเรียบร้อยแล้ว" -ForegroundColor Red
}
```

##### วิธีการรันสคริปต์จำลอง ESP32

เปิด PowerShell แล้วเข้าไปยังโฟลเดอร์ที่เก็บไฟล์ `mock_esp32.ps1`

```powershell
# เข้าโฟลเดอร์ของใบงาน
cd Example_codes/Lab12-4

# เริ่มการจำลอง ESP32
powershell -ExecutionPolicy Bypass -File .\mock_esp32.ps1

# หาก Broker อยู่ที่ IP อื่น ให้ระบุ IP
powershell -ExecutionPolicy Bypass -File .\mock_esp32.ps1 -Broker 192.168.1.105
```

---

### กิจกรรมที่ 12-4.3 การทดสอบระบบ 4 กรณี (Verification Test Matrix)

ให้นักศึกษาทดสอบระบบตามสถานการณ์ที่กำหนด และบันทึกผลการทดลองลงในตาราง

| รายการทดสอบ | การกระทำ (Hardware / Mock Action) | ผลที่คาดหวังบน Flutter App | ผลการทดลอง (ผ่าน/ไม่ผ่าน) |
| :--- | :--- | :--- | :--- |
| **Test Case 1: Telemetry Monitoring** | **ฮาร์ดแวร์:** หมุน Potentiometer จากซ้ายไปขวา **จำลอง:** ให้สคริปต์ส่งค่า Sine Wave | เข็ม Radial Gauge เปลี่ยนแปลงตามค่าแรงดันระหว่าง 0.0V–3.3V ค่า ADC เปลี่ยนแปลง และกราฟ Live Sparkline เคลื่อนไหวต่อเนื่อง | |
| **Test Case 2: Actuator Remote Control** | แตะสวิตช์บนแอปเพื่อเปิดหรือปิด LED | **ฮาร์ดแวร์:** LED ที่ GPIO 2 เปลี่ยนสถานะ **จำลอง:** PowerShell แสดงคำสั่งที่ได้รับ และส่ง ACK กลับไปยังแอป | |
| **Test Case 3: State Sync & Physical ACK** | กดสวิตช์บนแอปและตรวจสอบข้อความ ACK ที่ได้รับ | แอปได้รับ ACK และปรับสถานะสวิตช์ให้ตรงกับข้อมูลที่ได้รับจากอุปกรณ์ | |
| **Test Case 4: Fault Tolerance (LWT)** | **ฮาร์ดแวร์:** ตัดการเชื่อมต่อ ESP32 โดยไม่ส่ง Disconnect **จำลอง:** หยุดสคริปต์ด้วย `Ctrl + C` | แอปแสดงสถานะอุปกรณ์เป็น OFFLINE และควรปิดกั้นการส่งคำสั่งเมื่ออุปกรณ์ไม่พร้อมใช้งาน | |

> [!TIP]
> **คำสั่งจำลองการทดสอบด้วยตนเอง (Manual PowerShell Commands)**
>
> หากต้องการทดสอบแต่ละ Topic โดยไม่ใช้สคริปต์ `mock_esp32.ps1` สามารถใช้คำสั่งต่อไปนี้ได้

**1. จำลองสถานะ Online**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "online" -r
```

**2. จำลองการส่งข้อมูล Telemetry**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/telemetry" -m '{"voltage": 2.10, "raw": 2608, "uptime": 45}'
```

**3. ตรวจสอบข้อความคำสั่งจาก Flutter**

```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "iot/esp32/control" -v
```

**4. จำลองการส่ง ACK กลับไปยัง Flutter**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/led_ack" -m '{"led_state": true, "ack": true}'
```

**5. จำลองสถานะ Offline**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "offline" -r
```

---

### กิจกรรมที่ 12-4.4 การวัดความหน่วงเวลาในการควบคุม (Control Latency Benchmark)

1. แตะสวิตช์เปิดหรือปิด LED บนแอปพลิเคชัน Flutter
2. จับเวลาตั้งแต่ส่งคำสั่งจากแอปไปยัง ESP32 จนกระทั่งแอปได้รับข้อความยืนยันจากหัวข้อ `iot/esp32/led_ack`
3. บันทึกเวลาที่วัดได้และเปรียบเทียบความแตกต่างของสภาพแวดล้อมเครือข่าย

**ตารางบันทึกผลการทดลอง**

| สภาพแวดล้อมเครือข่าย | ประเภทการเชื่อมต่อ | เวลาตอบสนองโดยเฉลี่ย (RTT: ms) | ความเสถียร / ข้อสังเกต |
| :--- | :--- | :--- | :--- |
| **Local Wi-Fi LAN** | สมาร์ตโฟนและ ESP32 เชื่อมต่อ Wi-Fi Router เดียวกัน | ประมาณ 20–60 ms | การตอบสนองรวดเร็วและมีความหน่วงต่ำ |
| **Mobile Hotspot / 4G** | เชื่อมต่อผ่านเครือข่ายมือถือและ Broker ผ่าน Public IP หรือ VPN | ประมาณ 80–250 ms | ความเร็วขึ้นอยู่กับสัญญาณและเส้นทางเครือข่าย |

> **หมายเหตุ:** ค่าความหน่วงในตารางเป็นค่าประมาณสำหรับใช้อ้างอิง ไม่ใช่ผลการวัดจริง นักศึกษาต้องบันทึกค่าที่วัดได้จากการทดลองของตนเอง

---

## 4. สรุปผลและคำถามท้ายบทเรียน

### คำถามข้อที่ 1

**เหตุใดการใช้ `IndexedStack` ใน `MainNavigationShell` จึงมีความสำคัญสำหรับแอปพลิเคชัน IoT แบบ Multi-tab เมื่อเทียบกับการใช้ `body: _pages[_currentIndex]`?**

**คำตอบ**

```text
IndexedStack ช่วยรักษาสถานะของแต่ละแท็บไว้เมื่อมีการสลับหน้าจอ ทำให้ข้อมูลเซนเซอร์และการเชื่อมต่อ MQTT ไม่ต้องเริ่มต้นใหม่ทุกครั้ง ช่วยให้แอปพลิเคชันทำงานได้ต่อเนื่อง
```

### คำถามข้อที่ 2

**เมื่อเปรียบเทียบการพัฒนาด้วย Flutter กับการเปิดหน้าเว็บ HTTP บนเบราว์เซอร์ในสัปดาห์ที่ 8–10 เทคโนโลยีใดให้ประสบการณ์การใช้งาน (UX) ที่เหมาะสมกับการรับข้อมูลเซนเซอร์ความถี่สูงมากกว่า เพราะเหตุใด?**

**คำตอบ**

```text
Flutter เหมาะกับการพัฒนาแอปพลิเคชัน IoT ที่ต้องรับข้อมูลบ่อย ๆ เพราะสามารถอัปเดตหน้าจอและแสดงแอนิเมชันได้อย่างลื่นไหล แต่ความเสถียรยังขึ้นอยู่กับการจัดการข้อมูลและประสิทธิภาพของแอปด้วย
```

### คำถามข้อที่ 3

**หากต้องการนำแอปพลิเคชันนี้ไปเผยแพร่บน Google Play Store หรือ Apple App Store ต้องปรับปรุงด้านความปลอดภัยเพิ่มเติมอย่างไรบ้าง?**

**คำตอบ**

```text
ควรใช้ MQTTS ร่วมกับ TLS เพื่อเข้ารหัสข้อมูล ใช้ Token สำหรับยืนยันตัวตนและกำหนดสิทธิ์การเข้าถึง รวมถึงจัดเก็บ Credential ไว้ในพื้นที่ปลอดภัย เช่น Android Keystore หรือ iOS Keychain
```

---

## 5. สรุปผลการทดลอง

จากการทดลองสามารถนำส่วนประกอบต่าง ๆ ของระบบ IoT มาผสานรวมกันได้ โดย ESP32 ทำหน้าที่อ่านค่าจาก Potentiometer และควบคุม LED ส่วน Mosquitto Broker ทำหน้าที่เป็นตัวกลางในการรับส่งข้อมูลระหว่างบอร์ด ESP32 กับแอปพลิเคชัน Flutter

แอปพลิเคชันสามารถแสดงข้อมูล Telemetry และส่งคำสั่งควบคุมอุปกรณ์ผ่าน MQTT ได้ รวมถึงตรวจสอบสถานะการเชื่อมต่อและทดสอบความหน่วงในการตอบสนองของระบบ ทำให้เห็นภาพการทำงานของระบบ IoT ตั้งแต่การรับข้อมูลจากอุปกรณ์จริงไปจนถึงการแสดงผลและควบคุมผ่านสมาร์ตโฟน
