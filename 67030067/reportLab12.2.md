
# ใบงานการทดลองที่ 12.2
## การสร้างสวิตช์ควบคุมอุปกรณ์สองทาง (Bi-directional Actuator Switch) พร้อมระบบ Feedback และสถานะ LWT

### คำชี้แจง
พัฒนาแอปพลิเคชัน Flutter สำหรับควบคุมอุปกรณ์เปิด-ปิดผ่าน MQTT โดยให้แอปส่งคำสั่งไปยัง ESP32 และรอรับข้อมูลตอบกลับเพื่อยืนยันสถานะการทำงานจริงของอุปกรณ์ รวมถึงตรวจสอบสถานะ Online/Offline ผ่านกลไก Last Will and Testament (LWT)

---

## 1. วัตถุประสงค์การทดลอง

1. สามารถสร้างหน้าจอควบคุมหลอดไฟที่มีเอฟเฟกต์แสงเรือง (Glow Effect) และการสั่นตอบสนอง (Haptic Feedback) ได้
2. สามารถส่งคำสั่ง JSON ไปยังหัวข้อ `iot/esp32/control` ได้
3. สามารถรับข้อมูลจากหัวข้อ `iot/esp32/status` เพื่อตรวจสอบสถานะ Online/Offline ของบอร์ดได้
4. สามารถทดสอบการซิงโครไนซ์สถานะ (State Synchronization) และจำลองกรณีที่อุปกรณ์หรือเครือข่ายเกิดความผิดปกติได้

---

## 2. โครงสร้างหัวข้อ MQTT (Topic Architecture)

| ชื่อหัวข้อ (Topic) | ทิศทางการส่ง | รูปแบบข้อมูล (Payload Format) | วัตถุประสงค์ |
|---|---|---|---|
| `iot/esp32/control` | Flutter → ESP32 | `{"led": "ON"}` หรือ `{"led": "OFF"}` | ใช้ส่งคำสั่งเปิดหรือปิดหลอดไฟ |
| `iot/esp32/led_ack` | ESP32 → Flutter | `{"led_state": true, "ack": true}` | ใช้แจ้งผลการทำงานและยืนยันสถานะหลอดไฟ |
| `iot/esp32/status` | ESP32 → Broker (LWT) | `online` หรือ `offline` | ใช้รายงานความพร้อมในการทำงานของบอร์ด |

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-2.1 การเตรียมโปรเจกต์และสร้างหน้าจอควบคุมสวิตช์

#### 1. คัดลอกโปรเจกต์จาก Lab 12-1

นำโฟลเดอร์โปรเจกต์จากการทดลองที่ 12.1 มาคัดลอกเป็นโปรเจกต์ใหม่ เพื่อใช้งานแพ็กเกจ `mqtt_client` ที่ติดตั้งไว้แล้ว

**คำสั่ง PowerShell**

```powershell
# คัดลอกโปรเจกต์จาก Lab12-1 ไปยัง Lab12-2
Copy-Item -Recurse -Path "..\Lab12-1\iot_mqtt_basics" -Destination "..\Lab12-2\iot_actuator_switch"

# เข้าไปยังโฟลเดอร์โปรเจกต์
cd ..\Lab12-2\iot_actuator_switch

# ล้างไฟล์ Build เดิม
flutter clean
```

#### 2. สร้างไฟล์ actuator_control_screen.dart

สร้างไฟล์ `actuator_control_screen.dart` ภายในโฟลเดอร์ `lib/` แล้วใส่โค้ดดังนี้

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

class ActuatorControlScreen extends StatefulWidget {
  final String brokerIp;

  const ActuatorControlScreen({super.key, required this.brokerIp});

  @override
  State<ActuatorControlScreen> createState() => _ActuatorControlScreenState();
}

class _ActuatorControlScreenState extends State<ActuatorControlScreen> {
  MqttServerClient? _client;
  bool _isLedOn = false;
  bool _isDeviceOnline = false;
  bool _isWaitingAck = false;
  String _networkStatus = 'กำลังเชื่อมต่อ Broker...';

  @override
  void initState() {
    super.initState();
    _connectMqtt();
  }

  @override
  void dispose() {
    _client?.disconnect();
    super.dispose();
  }

  Future<void> _connectMqtt() async {
    final clientId =
        'Flutter_Switch_${DateTime.now().millisecondsSinceEpoch}';

    _client = MqttServerClient(widget.brokerIp, clientId);
    _client!.port = 1883;
    _client!.keepAlivePeriod = 20;

    _client!.onConnected = () {
      setState(() => _networkStatus = 'เชื่อมต่อ Broker เรียบร้อย');
      _subscribeTopics();
    };

    _client!.onDisconnected = () {
      setState(() {
        _networkStatus = 'ขาดการเชื่อมต่อ Broker!';
        _isDeviceOnline = false;
      });
    };

    try {
      await _client!.connect();
    } catch (e) {
      setState(() => _networkStatus = 'เชื่อมต่อล้มเหลว: $e');
    }
  }

  void _subscribeTopics() {
    // รับสถานะ Online / Offline
    _client?.subscribe('iot/esp32/status', MqttQos.atMostOnce);

    // รับข้อมูลยืนยันสถานะหลอดไฟ
    _client?.subscribe('iot/esp32/led_ack', MqttQos.atMostOnce);

    _client?.updates?.listen((List<MqttReceivedMessage<MqttMessage>> c) {
      final recMess = c[0].payload as MqttPublishMessage;
      final pt = MqttPublishPayload.bytesToStringAsString(
        recMess.payload.message,
      );
      final topic = c[0].topic;

      setState(() {
        if (topic == 'iot/esp32/status') {
          _isDeviceOnline = (pt.trim().toLowerCase() == 'online');
        } else if (topic == 'iot/esp32/led_ack') {
          _isWaitingAck = false;

          try {
            final data = jsonDecode(pt);
            _isLedOn = data['led_state'] == true;
          } catch (_) {}
        }
      });
    });
  }

  void _toggleLed(bool targetState) {
    if (_client?.connectionStatus?.state !=
        MqttConnectionState.connected) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('กรุณาเชื่อมต่อ Broker ก่อนสั่งการ'),
        ),
      );
      return;
    }

    if (!_isDeviceOnline) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('อุปกรณ์ ESP32 ออฟไลน์อยู่ ไม่สามารถส่งคำสั่งได้'),
        ),
      );
      return;
    }

    // สั่นเบา ๆ เมื่อผู้ใช้กดสวิตช์
    HapticFeedback.lightImpact();

    setState(() {
      _isWaitingAck = true;
    });

    final payloadMap = {'led': targetState ? 'ON' : 'OFF'};
    final jsonStr = jsonEncode(payloadMap);

    final builder = MqttClientPayloadBuilder();
    builder.addString(jsonStr);

    _client?.publishMessage(
      'iot/esp32/control',
      MqttQos.atLeastOnce,
      builder.payload!,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFF0F172A),
      appBar: AppBar(
        title: const Text('Actuator Control Center'),
        backgroundColor: const Color(0xFF1E293B),
      ),
      body: Padding(
        padding: const EdgeInsets.all(20.0),
        child: Column(
          children: [
            // แสดงสถานะ Online / Offline ของ ESP32
            Container(
              padding: const EdgeInsets.symmetric(
                horizontal: 16,
                vertical: 12,
              ),
              decoration: BoxDecoration(
                color: _isDeviceOnline
                    ? const Color(0x1A00E676)
                    : const Color(0x1AFF5252),
                borderRadius: BorderRadius.circular(16),
                border: Border.all(
                  color: _isDeviceOnline
                      ? const Color(0xFF00E676)
                      : const Color(0xFFFF5252),
                ),
              ),
              child: Row(
                children: [
                  Icon(
                    Icons.circle,
                    size: 12,
                    color: _isDeviceOnline
                        ? const Color(0xFF00E676)
                        : const Color(0xFFFF5252),
                  ),
                  const SizedBox(width: 10),
                  Text(
                    _isDeviceOnline
                        ? 'ESP32 ฮาร์ดแวร์: ออนไลน์ (ONLINE)'
                        : 'ESP32 ฮาร์ดแวร์: ออฟไลน์ (OFFLINE)',
                    style: TextStyle(
                      fontWeight: FontWeight.bold,
                      color: _isDeviceOnline
                          ? const Color(0xFF00E676)
                          : const Color(0xFFFF5252),
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 30),

            // การ์ดควบคุมหลอดไฟ
            Container(
              padding: const EdgeInsets.all(24),
              decoration: BoxDecoration(
                color: const Color(0xFF1E293B).withOpacity(0.8),
                borderRadius: BorderRadius.circular(24),
                border: Border.all(
                  color: _isLedOn
                      ? const Color(0xFF00E676).withOpacity(0.6)
                      : Colors.white12,
                  width: 2,
                ),
                boxShadow: _isLedOn
                    ? [
                        BoxShadow(
                          color: const Color(0xFF00E676).withOpacity(0.25),
                          blurRadius: 24,
                          spreadRadius: 4,
                        )
                      ]
                    : [],
              ),
              child: Column(
                children: [
                  AnimatedContainer(
                    duration: const Duration(milliseconds: 300),
                    padding: const EdgeInsets.all(24),
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      color: _isLedOn
                          ? const Color(0x3300E676)
                          : Colors.black26,
                    ),
                    child: Icon(
                      Icons.lightbulb,
                      size: 72,
                      color: _isLedOn
                          ? const Color(0xFF00E676)
                          : Colors.white24,
                    ),
                  ),
                  const SizedBox(height: 20),
                  const Text(
                    'หลอดไฟ LED ทดสอบ (GPIO 2)',
                    style: TextStyle(
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                      color: Colors.white,
                    ),
                  ),
                  const SizedBox(height: 6),
                  Text(
                    _isWaitingAck
                        ? 'กำลังรอการยืนยันจากบอร์ด (Waiting ACK)...'
                        : _isLedOn
                            ? 'สถานะ: เปิดสว่าง (Active)'
                            : 'สถานะ: ปิดการทำงาน (Standby)',
                    style: TextStyle(
                      color: _isWaitingAck
                          ? Colors.amber
                          : (_isLedOn
                              ? const Color(0xFF00E676)
                              : Colors.white38),
                    ),
                  ),
                  const SizedBox(height: 24),

                  // สวิตช์ควบคุมหลอดไฟ
                  Transform.scale(
                    scale: 1.4,
                    child: Switch(
                      value: _isLedOn,
                      activeColor: const Color(0xFF00E676),
                      onChanged: _isWaitingAck ? null : _toggleLed,
                    ),
                  ),
                ],
              ),
            ),
            const Spacer(),
            Text(
              _networkStatus,
              style: const TextStyle(
                fontSize: 12,
                color: Colors.white38,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### 3. ปรับปรุงไฟล์ main.dart

เปิดไฟล์ `lib/main.dart` แล้วแทนที่โค้ดเดิมด้วยโค้ดต่อไปนี้

```dart
import 'package:flutter/material.dart';
import 'actuator_control_screen.dart';

void main() {
  runApp(const ActuatorApp());
}

class ActuatorApp extends StatelessWidget {
  const ActuatorApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Lab 12.2 Actuator Switch',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),
      home: const ActuatorControlScreen(brokerIp: '127.0.0.1'),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

#### 4. สั่งรัน Flutter Application

```powershell
# ทดสอบบน Windows Desktop
flutter run -d windows

# หรือทดสอบบน Android Device / Emulator
flutter run -d <device-id>
```

---

### กิจกรรมที่ 12-2.2 การทดสอบร่วมกับสคริปต์จำลองฮาร์ดแวร์ (Mock Simulation)

หากยังไม่มีบอร์ด ESP32 สำหรับทดสอบจริง สามารถใช้คำสั่ง Mosquitto ใน PowerShell เพื่อจำลองการส่งข้อมูลจากบอร์ดมายังแอปพลิเคชันได้ โดยต้องเปิด Mosquitto Broker ไว้ก่อน

#### 1. จำลองการส่งสถานะ Online

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "online" -r
```

เมื่อส่งคำสั่งแล้ว แถบสถานะด้านบนของแอป Flutter ควรเปลี่ยนเป็นสีเขียวและแสดงสถานะ ESP32 ว่าออนไลน์

#### 2. ตรวจสอบคำสั่งที่ส่งจากแอป Flutter

```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "iot/esp32/control" -v
```

จากนั้นกลับไปที่แอป Flutter แล้วทดลองกดสวิตช์เปิดหรือปิดหลอดไฟ ในหน้าต่าง PowerShell จะปรากฏข้อความ เช่น `{"led":"ON"}` หรือ `{"led":"OFF"}` ตามการกดสวิตช์

#### 3. ส่งข้อความยืนยันกลับไปยังแอปพลิเคชัน

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/led_ack" -m '{"led_state": true, "ack": true}'
```

เมื่อแอปได้รับข้อความยืนยัน จะนำค่า `led_state` ใน JSON มาใช้ปรับสถานะสวิตช์และไอคอนหลอดไฟให้เป็นสถานะเปิด ซึ่งจะแสดงแสงสีเขียวบนหน้าจอ

#### 4. จำลองเหตุการณ์บอร์ดดับหรือเครือข่ายขาดการเชื่อมต่อ

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "offline" -r
```

จากนั้นตรวจสอบหน้าจอ Flutter แถบสถานะควรเปลี่ยนเป็นสีแดงและแสดงว่า ESP32 ออฟไลน์ เมื่ออุปกรณ์อยู่ในสถานะดังกล่าว แอปจะไม่อนุญาตให้ผู้ใช้ส่งคำสั่งควบคุมหลอดไฟ

> **หมายเหตุ:** ขั้นตอนนี้เป็นการจำลองส่งข้อความ `offline` ด้วยคำสั่ง Mosquitto CLI ไม่ใช่การทดสอบการทำงานของ LWT จริงโดยการตัดการเชื่อมต่อ ESP32 แบบผิดปกติ

---

## 4. สรุปผลและคำถามท้ายการทดลอง

### สรุปผลการทดลอง

จากการทดลองได้เรียนรู้การพัฒนาแอปพลิเคชัน Flutter สำหรับควบคุมอุปกรณ์ผ่านโปรโตคอล MQTT โดยสร้างหน้าจอสวิตช์ที่สามารถส่งคำสั่งเปิด-ปิดหลอดไฟไปยัง ESP32 ผ่าน MQTT Broker และรับข้อมูลตอบกลับเพื่อยืนยันผลการทำงานของอุปกรณ์ นอกจากนี้ยังได้ทดลองรับข้อมูลสถานะ Online และ Offline รวมถึงการใช้ Retain เพื่อเก็บสถานะล่าสุดไว้ที่ Broker ทำให้เข้าใจการรับส่งข้อมูลระหว่างแอปพลิเคชันกับอุปกรณ์ IoT และการตรวจสอบสถานะการทำงานของระบบได้ชัดเจนยิ่งขึ้น

### คำถามท้ายการทดลอง (Review Questions)

**1. ประโยชน์ของการรอสัญญาณ ACK จากหัวข้อ `iot/esp32/led_ack` แทนการเปลี่ยน UI ทันที (Optimistic Update) คืออะไร?**

การรอสัญญาณ ACK เป็นวิธีที่ช่วยให้แอปพลิเคชันตรวจสอบผลการทำงานของอุปกรณ์ก่อนนำข้อมูลมาแสดงบนหน้าจอ โดยเมื่อผู้ใช้กดสวิตช์ แอปจะส่งคำสั่งไปยัง ESP32 ผ่านหัวข้อ `iot/esp32/control` จากนั้นจะรอให้บอร์ดประมวลผลคำสั่งและส่งข้อความตอบกลับมายังหัวข้อ `iot/esp32/led_ack` เพื่อแจ้งสถานะของหลอดไฟที่เกิดขึ้นจริง วิธีนี้ช่วยลดความคลาดเคลื่อนระหว่างสถานะที่ผู้ใช้สั่งกับสถานะที่อุปกรณ์ทำงานได้จริง เพราะในบางกรณีคำสั่งอาจส่งไปไม่ถึงบอร์ด หรืออุปกรณ์อาจเกิดข้อผิดพลาดระหว่างการทำงาน หากแอปเปลี่ยนสถานะทันทีหลังจากผู้ใช้กดสวิตช์ ก็อาจทำให้หน้าจอแสดงว่าหลอดไฟเปิดอยู่ ทั้งที่หลอดไฟยังไม่ทำงาน การรอข้อความ ACK จึงช่วยให้ข้อมูลบนหน้าจอสอดคล้องกับผลตอบกลับจากอุปกรณ์มากขึ้น อีกทั้งยังทำให้ผู้ใช้ทราบว่าระบบกำลังรอการยืนยันจากบอร์ดอยู่ ส่งผลให้การควบคุมอุปกรณ์มีความถูกต้องและน่าเชื่อถือมากกว่าการเปลี่ยนสถานะโดยไม่ตรวจสอบผลการทำงานก่อน

**2. ทำไมเราจึงควรตั้งแฟล็ก Retain (`-r`) สำหรับหัวข้อสถานะ `iot/esp32/status`?**

การตั้งค่า Retain มีจุดประสงค์เพื่อให้ MQTT Broker เก็บข้อความล่าสุดที่ได้รับจากหัวข้อที่กำหนดไว้ เมื่อมี Client ใหม่เข้ามาเชื่อมต่อและ Subscribe หัวข้อนั้น Broker จะสามารถส่งข้อความสถานะล่าสุดที่บันทึกไว้ให้ได้ทันที โดยไม่ต้องรอให้อุปกรณ์ส่งข้อมูลรอบใหม่ ตัวอย่างเช่น เมื่อ ESP32 ส่งข้อความ `online` ไปยังหัวข้อ `iot/esp32/status` พร้อมเปิดใช้งาน Retain Broker จะเก็บสถานะดังกล่าวไว้ เมื่อแอป Flutter เปิดขึ้นมาใหม่และสมัครรับข้อมูลในหัวข้อเดียวกัน ก็จะได้รับข้อความ `online` ทำให้แอปสามารถแสดงสถานะของอุปกรณ์ได้ตั้งแต่เริ่มต้นโดยไม่ต้องรอการส่งข้อความครั้งถัดไป นอกจากนี้ยังช่วยให้การติดตามสถานะอุปกรณ์มีความต่อเนื่อง แม้แอปพลิเคชันจะปิดแล้วเปิดใหม่ก็ตาม อย่างไรก็ตาม ข้อมูลที่ถูกเก็บไว้ต้องได้รับการปรับปรุงให้ตรงกับสถานะจริงของอุปกรณ์อยู่เสมอ เช่น หากบอร์ดขาดการเชื่อมต่อ ควรมีการอัปเดตสถานะเป็น `offline` เพื่อให้ผู้ที่เข้ามา Subscribe ภายหลังได้รับข้อมูลที่ถูกต้อง ไม่เข้าใจว่าอุปกรณ์ยังออนไลน์ทั้งที่หยุดทำงานไปแล้ว ดังนั้น Retain จึงช่วยให้ระบบสามารถส่งต่อสถานะล่าสุดได้รวดเร็วและลดระยะเวลาในการรอข้อมูลจากอุปกรณ์
