
# ใบงานการทดลองที่ 12.3
## การสร้างแดชบอร์ดแสดงผลเซนเซอร์เรียลไทม์ด้วย Radial Gauge และ Live Trend Line Chart

### คำชี้แจง

การทดลองนี้เป็นการพัฒนาแอปพลิเคชัน Flutter สำหรับแสดงผลข้อมูลแรงดันไฟฟ้าที่ได้รับจาก ESP32 ผ่าน MQTT โดยรับข้อมูลในรูปแบบ JSON แล้วนำมาแสดงผลบนหน้าจอแดชบอร์ด ประกอบด้วยมาตรวัดทรงกลม (Radial Gauge) ที่มีแอนิเมชันเคลื่อนไหวอย่างนุ่มนวล กราฟเส้นแสดงแนวโน้มของข้อมูลย้อนหลัง และการ์ดแสดงค่าสถิติต่าง ๆ เช่น ค่าแรงดันสูงสุด ค่าแรงดันต่ำสุด ค่า ADC และระยะเวลาการทำงานของบอร์ด

---

## 1. วัตถุประสงค์การทดลอง

1. สามารถรับและถอดรหัสข้อมูล JSON จากหัวข้อ `iot/esp32/telemetry` เพื่อนำข้อมูลเข้าสู่โปรแกรมภาษา Dart ได้
2. สามารถสร้างมาตรวัดทรงกลม (Radial Gauge) ด้วย `CustomPainter` และใช้ `TweenAnimationBuilder` เพื่อทำให้เข็มมาตรวัดเคลื่อนไหวอย่างนุ่มนวลได้
3. สามารถเก็บประวัติข้อมูลแรงดันไฟฟ้าและแสดงผลเป็นกราฟเส้นแนวโน้มย้อนหลังได้
4. สามารถคำนวณและแสดงค่าสถิติ ได้แก่ ค่าแรงดันสูงสุด (Max) ค่าแรงดันต่ำสุด (Min) และระยะเวลาการทำงาน (Uptime) ได้

---

## 2. โครงสร้างหัวข้อ MQTT และการไหลของข้อมูล (Topic Architecture & Data Flow)

| ชื่อหัวข้อ (Topic) | ทิศทางการส่ง | รูปแบบข้อมูล (Payload Format) | วัตถุประสงค์ |
|---|---|---|---|
| `iot/esp32/telemetry` | ESP32 → Flutter | `{"voltage": 2.50, "raw": 3102, "uptime": 10}` | ส่งข้อมูลแรงดันไฟฟ้า ค่า ADC และระยะเวลาการทำงานของระบบแบบเรียลไทม์ |

ตัวอย่างข้อมูล JSON ที่ได้รับจาก ESP32

```json
{
  "voltage": 2.50,
  "raw": 3102,
  "uptime": 10
}
```

ความหมายของข้อมูลแต่ละรายการ

- `voltage` คือ ค่าแรงดันไฟฟ้าที่ตรวจวัดได้ หน่วยเป็นโวลต์ (V)
- `raw` คือ ค่าดิบที่อ่านได้จาก ADC ของ ESP32
- `uptime` คือ ระยะเวลาที่บอร์ดทำงาน หน่วยเป็นวินาที

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-3.1 การเตรียมโปรเจกต์และการสร้างคอมโพเนนต์มาตรวัด

#### 1. คัดลอกโปรเจกต์จาก Lab ก่อนหน้า

คัดลอกโฟลเดอร์โปรเจกต์จากการทดลองที่ 12.1 หรือ 12.2 มาใช้เป็นโปรเจกต์ใหม่ เพื่อให้สามารถใช้งานแพ็กเกจ `mqtt_client` ที่ติดตั้งไว้แล้วได้

**คำสั่ง PowerShell**

```powershell
# กรณีที่ 1: คัดลอกโปรเจกต์จาก Lab12-1
Copy-Item -Recurse -Path "..\Lab12-1\iot_mqtt_basics" -Destination "..\Lab12-3\iot_telemetry_gauge"

# กรณีที่ 2: คัดลอกโปรเจกต์จาก Lab12-2
Copy-Item -Recurse -Path "..\Lab12-2\iot_actuator_switch" -Destination "..\Lab12-3\iot_telemetry_gauge"

# เข้าไปยังโฟลเดอร์โปรเจกต์ใหม่
cd ..\Lab12-3\iot_telemetry_gauge

# ล้างแคชการคอมไพล์เดิม
flutter clean
```

หมายเหตุ: สามารถคัดลอกโฟลเดอร์ผ่าน File Explorer ได้เช่นกัน แต่หลังจากคัดลอกแล้วควรรันคำสั่ง `flutter clean` ทุกครั้ง เพื่อป้องกันปัญหาจากไฟล์ Build เดิม

#### 2. สร้างคอมโพเนนต์มาตรวัด (`lib/radial_gauge_widget.dart`)

สร้างไฟล์ `radial_gauge_widget.dart` ภายในโฟลเดอร์ `lib/` เพื่อใช้วาดมาตรวัดแรงดันไฟฟ้า โดยกำหนดช่วงการแสดงผลตั้งแต่ 0.0 ถึง 3.3 โวลต์ และใช้แอนิเมชันช่วยให้การเปลี่ยนแปลงของเข็มดูต่อเนื่อง

**ไฟล์: `lib/radial_gauge_widget.dart`**

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class RadialGaugeWidget extends StatelessWidget {
  final double value; // ค่าแรงดันไฟฟ้า 0.0 ถึง 3.3V
  final double maxLimit;

  const RadialGaugeWidget({
    super.key,
    required this.value,
    this.maxLimit = 3.3,
  });

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder<double>(
      tween: Tween<double>(begin: 0.0, end: value),
      duration: const Duration(milliseconds: 300),
      curve: Curves.easeOutCubic,
      builder: (context, animatedValue, child) {
        return SizedBox(
          width: 260,
          height: 260,
          child: Stack(
            alignment: Alignment.center,
            children: [
              CustomPaint(
                size: const Size(260, 260),
                painter: _GaugePainter(
                  value: animatedValue,
                  maxLimit: maxLimit,
                ),
              ),
              Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(
                    animatedValue.toStringAsFixed(2),
                    style: const TextStyle(
                      fontSize: 48,
                      fontWeight: FontWeight.bold,
                      color: Colors.white,
                      fontFamily: 'monospace',
                    ),
                  ),
                  const Text(
                    'VOLTS (V)',
                    style: TextStyle(
                      fontSize: 14,
                      letterSpacing: 2,
                      fontWeight: FontWeight.bold,
                      color: Color(0xFF00E5FF),
                    ),
                  ),
                ],
              ),
            ],
          ),
        );
      },
    );
  }
}

class _GaugePainter extends CustomPainter {
  final double value;
  final double maxLimit;

  _GaugePainter({
    required this.value,
    required this.maxLimit,
  });

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = size.width / 2 - 20;

    const startAngle = 135 * (math.pi / 180);
    const sweepAngle = 270 * (math.pi / 180);

    // 1. วาดเส้นพื้นหลังของมาตรวัด
    final trackPaint = Paint()
      ..color = const Color(0xFF1E293B)
      ..style = PaintingStyle.stroke
      ..strokeWidth = 18
      ..strokeCap = StrokeCap.round;

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      startAngle,
      sweepAngle,
      false,
      trackPaint,
    );

    // 2. คำนวณสัดส่วนของแรงดันไฟฟ้าที่วัดได้
    final ratio = (value / maxLimit).clamp(0.0, 1.0);
    final currentSweep = sweepAngle * ratio;

    // 3. วาดเส้นแสดงระดับแรงดันด้วยสี Gradient
    final activePaint = Paint()
      ..shader = const SweepGradient(
        colors: [
          Color(0xFF00E5FF),
          Color(0xFF00E676),
          Color(0xFFFFD600),
          Color(0xFFFF5252),
        ],
        startAngle: 0.0,
        endAngle: math.pi * 2,
      ).createShader(
        Rect.fromCircle(center: center, radius: radius),
      )
      ..style = PaintingStyle.stroke
      ..strokeWidth = 18
      ..strokeCap = StrokeCap.round;

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      startAngle,
      currentSweep,
      false,
      activePaint,
    );
  }

  @override
  bool shouldRepaint(covariant _GaugePainter oldDelegate) {
    return oldDelegate.value != value;
  }
}
```

---

### กิจกรรมที่ 12-3.2 การสร้างหน้าจอ Telemetry Dashboard

#### 1. สร้างไฟล์หน้าจอแดชบอร์ด (`lib/telemetry_dashboard_screen.dart`)

สร้างไฟล์ `telemetry_dashboard_screen.dart` ภายในโฟลเดอร์ `lib/` เพื่อใช้เชื่อมต่อ MQTT รับข้อมูล JSON จาก ESP32 และนำข้อมูลไปแสดงบนมาตรวัด การ์ดสถิติ และกราฟเส้นย้อนหลัง

**ไฟล์: `lib/telemetry_dashboard_screen.dart`**

```dart
import 'dart:convert';
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';
import 'radial_gauge_widget.dart';

class TelemetryDashboardScreen extends StatefulWidget {
  final String brokerIp;

  const TelemetryDashboardScreen({
    super.key,
    required this.brokerIp,
  });

  @override
  State<TelemetryDashboardScreen> createState() =>
      _TelemetryDashboardScreenState();
}

class _TelemetryDashboardScreenState
    extends State<TelemetryDashboardScreen> {
  MqttServerClient? _client;

  double _currentVoltage = 0.0;
  int _currentRaw = 0;
  int _currentUptime = 0;

  double _maxVoltage = 0.0;
  double _minVoltage = 3.3;

  final List<double> _history = [];
  bool _isConnected = false;

  @override
  void initState() {
    super.initState();
    _connect();
  }

  @override
  void dispose() {
    _client?.disconnect();
    super.dispose();
  }

  // เชื่อมต่อ MQTT Broker
  Future<void> _connect() async {
    final clientId =
        'Flutter_Telemetry_${DateTime.now().millisecondsSinceEpoch}';

    _client = MqttServerClient(widget.brokerIp, clientId);
    _client!.port = 1883;

    _client!.onConnected = () {
      setState(() => _isConnected = true);

      // Subscribe เพื่อรับข้อมูลจาก ESP32
      _client!.subscribe(
        'iot/esp32/telemetry',
        MqttQos.atMostOnce,
      );

      _client!.updates?.listen(_handleIncomingMessage);
    };

    _client!.onDisconnected = () {
      setState(() => _isConnected = false);
    };

    try {
      await _client!.connect();
    } catch (e) {
      debugPrint('MQTT connection error: $e');
      _client?.disconnect();
    }
  }

  // รับและแปลงข้อมูล JSON จาก MQTT
  void _handleIncomingMessage(
    List<MqttReceivedMessage<MqttMessage>> c,
  ) {
    final recMess = c[0].payload as MqttPublishMessage;

    final payload = MqttPublishPayload.bytesToStringAsString(
      recMess.payload.message,
    );

    try {
      final data = jsonDecode(payload) as Map<String, dynamic>;

      final v = (data['voltage'] as num?)?.toDouble() ?? 0.0;
      final raw = (data['raw'] as num?)?.toInt() ?? 0;
      final up = (data['uptime'] as num?)?.toInt() ?? 0;

      // ตรวจสอบช่วงแรงดันก่อนนำไปแสดงผล
      if (v < 0.0 || v > 3.3) {
        debugPrint('Voltage out of range: $v');
        return;
      }

      if (!mounted) return;

      setState(() {
        _currentVoltage = v;
        _currentRaw = raw;
        _currentUptime = up;

        // คำนวณค่าแรงดันสูงสุดและต่ำสุด
        _maxVoltage = math.max(_maxVoltage, v);
        _minVoltage = math.min(_minVoltage, v);

        // เก็บประวัติแรงดันสำหรับวาดกราฟ
        _history.add(v);

        // จำกัดข้อมูลย้อนหลังไม่เกิน 30 ตัวอย่าง
        if (_history.length > 30) {
          _history.removeAt(0);
        }
      });
    } catch (e) {
      debugPrint('Parsing error: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFF0F172A),
      appBar: AppBar(
        title: const Text('Real-time Telemetry Dashboard'),
        backgroundColor: const Color(0xFF1E293B),
        actions: [
          Padding(
            padding: const EdgeInsets.only(right: 16.0),
            child: Icon(
              Icons.circle,
              size: 14,
              color: _isConnected
                  ? const Color(0xFF00E676)
                  : Colors.redAccent,
            ),
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(20),
        child: Column(
          children: [
            // 1. มาตรวัดแรงดันไฟฟ้าทรงกลม
            Center(
              child: RadialGaugeWidget(
                value: _currentVoltage,
                maxLimit: 3.3,
              ),
            ),

            const SizedBox(height: 24),

            // 2. การ์ดแสดงค่าสถิติ
            Row(
              children: [
                _buildMetricCard(
                  'ค่าดิบ ADC',
                  '$_currentRaw / 4095',
                  const Color(0xFF00E5FF),
                ),
                const SizedBox(width: 12),
                _buildMetricCard(
                  'แรงดันสูงสุด',
                  '${_maxVoltage.toStringAsFixed(2)} V',
                  const Color(0xFFFF5252),
                ),
                const SizedBox(width: 12),
                _buildMetricCard(
                  'แรงดันต่ำสุด',
                  '${_minVoltage == 3.3 ? 0.0 : _minVoltage.toStringAsFixed(2)} V',
                  const Color(0xFF00E676),
                ),
              ],
            ),

            const SizedBox(height: 20),

            // 3. กราฟแสดงแนวโน้มแรงดันย้อนหลัง
            Container(
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color: const Color(0xFF1E293B).withOpacity(0.8),
                borderRadius: BorderRadius.circular(20),
                border: Border.all(color: Colors.white12),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'แนวโน้มแรงดันย้อนหลัง 30 ตัวอย่าง (Live Trend)',
                    style: TextStyle(
                      fontWeight: FontWeight.bold,
                      color: Colors.white70,
                    ),
                  ),
                  const SizedBox(height: 16),
                  SizedBox(
                    height: 100,
                    width: double.infinity,
                    child: CustomPaint(
                      painter: _SparklinePainter(
                        history: _history,
                        maxLimit: 3.3,
                      ),
                    ),
                  ),
                ],
              ),
            ),

            const SizedBox(height: 16),

            // แสดงระยะเวลาการทำงานของ ESP32
            Text(
              'ESP32 Uptime: ${_currentUptime}s',
              style: const TextStyle(
                color: Colors.white38,
                fontSize: 13,
              ),
            ),
          ],
        ),
      ),
    );
  }

  // สร้างการ์ดแสดงข้อมูลแต่ละประเภท
  Widget _buildMetricCard(
    String title,
    String val,
    Color color,
  ) {
    return Expanded(
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: const Color(0xFF1E293B),
          borderRadius: BorderRadius.circular(16),
          border: Border.all(
            color: color.withOpacity(0.3),
          ),
        ),
        child: Column(
          children: [
            Text(
              title,
              style: const TextStyle(
                fontSize: 11,
                color: Colors.white54,
              ),
            ),
            const SizedBox(height: 6),
            Text(
              val,
              style: TextStyle(
                fontSize: 16,
                fontWeight: FontWeight.bold,
                color: color,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// คลาสสำหรับวาดกราฟเส้นแนวโน้มแรงดัน
class _SparklinePainter extends CustomPainter {
  final List<double> history;
  final double maxLimit;

  _SparklinePainter({
    required this.history,
    required this.maxLimit,
  });

  @override
  void paint(Canvas canvas, Size size) {
    if (history.length < 2) return;

    final paint = Paint()
      ..color = const Color(0xFF00E5FF)
      ..strokeWidth = 2.5
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    final path = Path();
    final dx = size.width / (history.length - 1);

    for (int i = 0; i < history.length; i++) {
      final x = i * dx;

      final y = size.height -
          ((history[i] / maxLimit).clamp(0.0, 1.0) * size.height);

      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }

    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(covariant _SparklinePainter oldDelegate) {
    return true;
  }
}
```

#### 2. ปรับปรุงไฟล์ `lib/main.dart`

เปิดไฟล์ `lib/main.dart` แล้วแทนที่โค้ดเดิม เพื่อกำหนดให้ `TelemetryDashboardScreen` เป็นหน้าจอเริ่มต้นของแอปพลิเคชัน

**ไฟล์: `lib/main.dart`**

```dart
import 'package:flutter/material.dart';
import 'telemetry_dashboard_screen.dart';

void main() {
  runApp(const TelemetryApp());
}

class TelemetryApp extends StatelessWidget {
  const TelemetryApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Lab 12.3 Telemetry Dashboard',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),

      // กำหนด IP ของ MQTT Broker
      home: const TelemetryDashboardScreen(
        brokerIp: '127.0.0.1',
      ),

      debugShowCheckedModeBanner: false,
    );
  }
}
```

#### 3. สั่งรัน Flutter Application

เปิด Terminal ภายในโฟลเดอร์โปรเจกต์ แล้วใช้คำสั่งต่อไปนี้

```powershell
# รันบน Windows Desktop
flutter run -d windows

# หรือรันบน Android Device / Emulator
flutter run -d <device-id>
```

---

### กิจกรรมที่ 12-3.3 การทดสอบร่วมกับสคริปต์จำลองเซนเซอร์ (Telemetry Mock Simulation)

หากยังไม่มีบอร์ด ESP32 หรือวงจรแบ่งแรงดัน Potentiometer สามารถจำลองการส่งข้อมูลเซนเซอร์เข้าสู่ MQTT Broker ได้ โดยเลือกใช้ Python หรือ Mosquitto CLI

#### 1. การทดสอบด้วยสคริปต์ Python Telemetry Simulator

สร้างไฟล์ชื่อ `pyTeleSim.py` ภายในโปรเจกต์หรือโฟลเดอร์สำหรับทดสอบ เพื่อจำลองการส่งค่าแรงดันไฟฟ้าในรูปแบบคลื่น Sine Wave ตั้งแต่ 0.0 ถึง 3.3V อย่างต่อเนื่อง

**ไฟล์: `pyTeleSim.py`**

```python
import paho.mqtt.client as mqtt
import time
import json
import math

client = mqtt.Client(
    mqtt.CallbackAPIVersion.VERSION2,
    "PyTelemetrySim"
)

client.connect("127.0.0.1", 1883)
client.loop_start()

t = 0

try:
    while True:
        # สร้างคลื่น Sine จำลองแรงดันไฟฟ้า 0.0 ถึง 3.3V
        voltage = round(1.65 + 1.5 * math.sin(t * 0.2), 2)

        # คำนวณค่า ADC จากแรงดันไฟฟ้า
        raw = int((voltage / 3.3) * 4095)

        # สร้างข้อมูล JSON
        payload = json.dumps({
            "voltage": voltage,
            "raw": raw,
            "uptime": t
        })

        # ส่งข้อมูลไปยัง MQTT Topic
        client.publish("iot/esp32/telemetry", payload)

        print(f"Published: {payload}")

        t += 1
        time.sleep(0.5)

except KeyboardInterrupt:
    print("หยุดการจำลองข้อมูล")

finally:
    client.loop_stop()
    client.disconnect()
```

ติดตั้งไลบรารี Paho MQTT หากยังไม่มี

```powershell
pip install paho-mqtt
```

จากนั้นรันสคริปต์ด้วยคำสั่ง

```powershell
python pyTeleSim.py
```

เมื่อเปิดแอปพลิเคชัน Flutter ควบคู่กับสคริปต์ Python จะเห็นว่าเข็ม Radial Gauge เปลี่ยนแปลงตามค่าแรงดันไฟฟ้าที่ส่งเข้ามา การ์ดสถิติแสดงค่า ADC แรงดันสูงสุด และแรงดันต่ำสุด ส่วนกราฟจะแสดงแนวโน้มของข้อมูลย้อนหลังไม่เกิน 30 ตัวอย่าง
<img width="665" height="338" alt="Screenshot 2569-11-01 at 02 28 42" src="https://github.com/user-attachments/assets/9fc839c7-a8c1-4eb6-80fc-a235ef107ac3" />

#### 2. การทดสอบแบบเจาะจงค่าผ่าน Mosquitto CLI (PowerShell)

สามารถใช้คำสั่ง `mosquitto_pub` เพื่อส่งค่าแรงดันไฟฟ้าที่กำหนดเอง และตรวจสอบการเปลี่ยนแปลงบนแดชบอร์ดได้

**ทดสอบส่งค่าปกติ 1.65 V**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/telemetry" -m '{"voltage": 1.65, "raw": 2048, "uptime": 10}'
```

ผลที่ควรสังเกต:
- เข็ม Radial Gauge แสดงค่าใกล้กึ่งกลางของมาตรวัด
- แรงดันไฟฟ้าแสดงเป็น 1.65 V
- การ์ดค่า ADC แสดงค่า 2048 / 4095

**ทดสอบส่งค่าแรงดันสูง 3.15 V**

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/telemetry" -m '{"voltage": 3.15, "raw": 3910, "uptime": 20}'
```

ผลที่ควรสังเกต:
- เข็มมาตรวัดเคลื่อนที่ไปใกล้ระดับสูงสุด
- การ์ดแรงดันสูงสุดเปลี่ยนเป็น 3.15 V
- ค่า ADC แสดงเป็น 3910 / 4095

**จำลองการส่งข้อมูลต่อเนื่อง 30 ค่า ด้วย PowerShell Loop**

```powershell
1..30 | ForEach-Object {
    $v = [Math]::Round(1.65 + 1.5 * [Math]::Sin($_ * 0.4), 2)
    $raw = [int](($v / 3.3) * 4095)

    mosquitto_pub -h 127.0.0.1 -p 1883 `
        -t "iot/esp32/telemetry" `
        -m "{`"voltage`": $v, `"raw`": $raw, `"uptime`": $_}"

    Start-Sleep -Milliseconds 500
}
```

เมื่อรันคำสั่ง จะมีการส่งข้อมูลแรงดันไฟฟ้า 30 ค่า โดยเว้นช่วงประมาณ 500 มิลลิวินาทีต่อครั้ง สามารถสังเกตการเปลี่ยนแปลงของมาตรวัดและกราฟเส้นบนหน้าจอได้

---

## 4. สรุปผลและคำถามท้ายการทดลอง

### สรุปผลการทดลอง

จากการทดลองได้เรียนรู้การสร้างแดชบอร์ดด้วย Flutter เพื่อรับข้อมูลแรงดันไฟฟ้าจาก ESP32 ผ่าน MQTT โดยใช้ข้อมูล JSON ในการส่งค่าแรงดันไฟฟ้า ค่า ADC และระยะเวลาการทำงานของระบบ จากนั้นนำข้อมูลมาแสดงผลผ่านมาตรวัดทรงกลมที่ใช้ CustomPainter และ TweenAnimationBuilder ช่วยให้การเคลื่อนไหวของมาตรวัดดูต่อเนื่อง นอกจากนี้ยังสามารถเก็บข้อมูลแรงดันย้อนหลังเพื่อสร้างกราฟเส้น และคำนวณค่าสถิติ เช่น ค่าแรงดันสูงสุดและต่ำสุดได้ ทำให้สามารถติดตามการเปลี่ยนแปลงของข้อมูลเซนเซอร์แบบเรียลไทม์ได้

### คำถามท้ายการทดลอง (Review Questions)

**1. เหตุใดการใช้ `TweenAnimationBuilder` จึงทำให้เข็มมาตรวัดดูนุ่มนวลและไม่กระตุก แม้ข้อมูลเซนเซอร์จะเข้ามาทุก ๆ 500 มิลลิวินาที?**

เนื่องจาก `TweenAnimationBuilder` จะช่วยเปลี่ยนค่าจากตำแหน่งเดิมไปยังค่าใหม่ทีละขั้นตามระยะเวลาที่กำหนดไว้ ในการทดลองนี้กำหนดเวลาแอนิเมชันไว้ที่ 300 มิลลิวินาที และใช้ `Curves.easeOutCubic` เพื่อให้การเคลื่อนไหวดูนุ่มนวลขึ้น เมื่อได้รับข้อมูลแรงดันไฟฟ้าค่าใหม่ เข็มมาตรวัดจึงไม่เปลี่ยนตำแหน่งทันที แต่จะค่อย ๆ เคลื่อนที่ไปยังตำแหน่งใหม่ ทำให้การแสดงผลต่อเนื่องและดูเป็นธรรมชาติมากขึ้น แม้เซนเซอร์จะส่งข้อมูลเข้ามาเป็นระยะ ๆ ทุก 500 มิลลิวินาทีก็ตาม

**2. การจำกัดความยาวของอาร์เรย์ `_history` ให้เก็บไม่เกิน 30 ตัวอย่าง ส่งผลดีต่อประสิทธิภาพหน่วยความจำของสมาร์ตโฟนอย่างไร?**

การกำหนดให้ `_history` เก็บข้อมูลย้อนหลังไม่เกิน 30 ตัวอย่าง ช่วยควบคุมปริมาณข้อมูลที่แอปพลิเคชันต้องเก็บไว้ในหน่วยความจำ หากไม่มีการจำกัดจำนวนข้อมูล เมื่อเซนเซอร์ส่งข้อมูลเข้ามาอย่างต่อเนื่อง รายการข้อมูลก็จะเพิ่มขึ้นเรื่อย ๆ และอาจทำให้แอปใช้หน่วยความจำมากเกินความจำเป็น การเก็บไว้เพียง 30 ตัวอย่างจึงช่วยลดภาระในการจัดเก็บและการประมวลผลกราฟ ทำให้แอปทำงานได้คล่องตัวและเหมาะกับการแสดงข้อมูลแบบเรียลไทม์มากขึ้น

**3. หากเซนเซอร์ส่งค่าผิดพลาด เช่น ค่านอกช่วง 0 - 3.3V หรือค่าเป็น `null` โค้ดส่วนใดใน `telemetry_dashboard_screen.dart` ที่ช่วยป้องกันไม่ให้แอปแครช?**

ส่วนที่ช่วยจัดการข้อมูลผิดพลาดคือขั้นตอนการถอดรหัส JSON ภายในฟังก์ชัน `_handleIncomingMessage()` โดยมีการใช้ `try-catch` เพื่อดักจับข้อผิดพลาดที่อาจเกิดขึ้นระหว่างการแปลงข้อมูล และใช้ `??` กำหนดค่าเริ่มต้นเมื่อข้อมูลบางรายการเป็น `null` เช่น หากไม่มีค่าแรงดันไฟฟ้า โปรแกรมจะกำหนดค่าเริ่มต้นเป็น 0.0 แทนการนำค่า null ไปใช้งานโดยตรง นอกจากนี้ควรตรวจสอบช่วงของแรงดันไฟฟ้าก่อนนำไปแสดงผล หากได้รับค่าที่ต่ำกว่า 0 หรือสูงกว่า 3.3 โวลต์ ควรปฏิเสธข้อมูลดังกล่าว เพื่อป้องกันไม่ให้ข้อมูลที่ผิดปกติส่งผลต่อการแสดงผลของแดชบอร์ด

---
