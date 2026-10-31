
# ใบงานการทดลองที่ 12.1
### การติดตั้งสภาพแวดล้อม Flutter และการพัฒนา MQTT Client พื้นฐานบนสมาร์ตโฟน

> **คำชี้แจง**
>
> ในการทดลองนี้เป็นการเริ่มต้นพัฒนาแอปพลิเคชันด้วย Flutter โดยติดตั้งแพ็กเกจ `mqtt_client` และเขียนโปรแกรมภาษา Dart เพื่อเชื่อมต่อกับ Eclipse Mosquitto Broker พร้อมสร้างหน้าจอสำหรับแสดงสถานะการเชื่อมต่อ และทดสอบการส่งและรับข้อความผ่าน MQTT

---

## 1. วัตถุประสงค์การทดลอง

1. สามารถตรวจสอบความพร้อมของ Flutter SDK ด้วยคำสั่ง `flutter doctor` ได้
2. สามารถสร้างโปรเจกต์ Flutter และติดตั้งแพ็กเกจ `mqtt_client` ได้
3. สามารถเขียนโปรแกรมเชื่อมต่อและยกเลิกการเชื่อมต่อกับ Mosquitto Broker ผ่านพอร์ต 1883 ได้
4. สามารถทดสอบการส่งและรับข้อความระหว่าง Flutter กับ Mosquitto CLI ได้

---

## 2. โครงสร้างการเชื่อมต่อระบบ (System Architecture)

ระบบประกอบด้วย Flutter Client ที่ทำงานบนอุปกรณ์ และ Mosquitto Broker ที่ทำหน้าที่เป็นตัวกลางในการรับส่งข้อความ โดยใช้โปรโตคอล MQTT ในการสื่อสาร

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-1.1: ตรวจสอบและสร้างโปรเจกต์ Flutter

**1. ตรวจสอบความพร้อมของ Flutter SDK**

เปิด PowerShell แล้วพิมพ์คำสั่ง

```powershell
flutter doctor
```

ตรวจสอบผลลัพธ์ว่ามีเครื่องมือที่จำเป็นสำหรับการพัฒนาแอปพลิเคชันพร้อมใช้งานหรือไม่

**2. สร้างโปรเจกต์ Flutter**

```powershell
flutter create iot_mqtt_basics
cd iot_mqtt_basics
```

**3. ติดตั้งแพ็กเกจ mqtt_client**

```powershell
flutter pub add mqtt_client
```

หรือตรวจสอบการกำหนดค่าในไฟล์ `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  mqtt_client: ^10.3.0
```

---

### กิจกรรมที่ 12-1.2: การเขียนโปรแกรม MQTT Client (`lib/main.dart`)

เปิดไฟล์ `lib/main.dart` แล้วใส่โค้ดดังต่อไปนี้

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

void main() {
  runApp(const MqttBasicsApp());
}

class MqttBasicsApp extends StatelessWidget {
  const MqttBasicsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Lab 12.1 MQTT Basics',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),
      home: const MqttBasicsScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class MqttBasicsScreen extends StatefulWidget {
  const MqttBasicsScreen({super.key});

  @override
  State<MqttBasicsScreen> createState() => _MqttBasicsScreenState();
}

class _MqttBasicsScreenState extends State<MqttBasicsScreen> {
  final TextEditingController _brokerIpController =
      TextEditingController(text: '127.0.0.1');

  MqttServerClient? _client;
  String _statusText = 'ยังไม่ได้เชื่อมต่อ (Disconnected)';
  Color _statusColor = Colors.redAccent;
  final List<String> _receivedLogs = [];

  @override
  void dispose() {
    _brokerIpController.dispose();
    _client?.disconnect();
    super.dispose();
  }

  Future<void> _connectToBroker() async {
    setState(() {
      _statusText = 'กำลังเชื่อมต่อไปยัง Broker...';
      _statusColor = Colors.amber;
    });

    final broker = _brokerIpController.text.trim();
    final clientId =
        'Flutter_Lab12_1_${DateTime.now().millisecondsSinceEpoch}';

    _client = MqttServerClient(broker, clientId);
    _client!.port = 1883;
    _client!.keepAlivePeriod = 20;
    _client!.autoReconnect = true;

    _client!.onConnected = () {
      if (!mounted) return;
      setState(() {
        _statusText = 'เชื่อมต่อสำเร็จ (Connected)';
        _statusColor = const Color(0xFF00E676);
      });
      _subscribeToTestTopic();
    };

    _client!.onDisconnected = () {
      if (!mounted) return;
      setState(() {
        _statusText = 'การเชื่อมต่อหลุด (Disconnected)';
        _statusColor = Colors.redAccent;
      });
    };

    final connMessage = MqttConnectMessage()
        .withClientIdentifier(clientId)
        .startClean()
        .withWillTopic('iot/flutter/lwt')
        .withWillMessage('Client crashed')
        .withWillQos(MqttQos.atLeastOnce);

    _client!.connectionMessage = connMessage;

    try {
      await _client!.connect();
    } on SocketException catch (e) {
      _appendLog('Socket Error: ไม่สามารถเชื่อมต่อ Broker ได้ ($e)');
      _client?.disconnect();
    } catch (e) {
      _appendLog('Error: $e');
      _client?.disconnect();
    }
  }

  void _disconnectFromBroker() {
    _client?.disconnect();
    _appendLog('ตัดการเชื่อมต่อเรียบร้อย');
  }

  void _subscribeToTestTopic() {
    const topic = 'test/broker/echo';

    _client?.subscribe(topic, MqttQos.atMostOnce);
    _appendLog('Subscribed to: $topic');

    _client?.updates?.listen((messages) {
      final recMess = messages[0].payload as MqttPublishMessage;
      final message = MqttPublishPayload.bytesToStringAsString(
        recMess.payload.message,
      );

      _appendLog('[${messages[0].topic}] -> $message');
    });
  }

  void _publishTestMessage() {
    if (_client?.connectionStatus?.state !=
        MqttConnectionState.connected) {
      _appendLog('ยังไม่ได้เชื่อมต่อ จึงไม่สามารถส่งข้อความได้');
      return;
    }

    const topic = 'test/flutter/ping';

    final builder = MqttClientPayloadBuilder();
    builder.addString(
      'Hello from Flutter at ${DateTime.now().toIso8601String()}',
    );

    _client?.publishMessage(
      topic,
      MqttQos.atMostOnce,
      builder.payload!,
    );

    _appendLog('Published -> $topic');
  }

  void _appendLog(String log) {
    if (!mounted) return;

    setState(() {
      final time = DateTime.now()
          .toLocal()
          .toString()
          .substring(11, 19);

      _receivedLogs.insert(0, '[$time] $log');

      if (_receivedLogs.length > 50) {
        _receivedLogs.removeLast();
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Lab 12.1: MQTT Client Basics'),
        backgroundColor: const Color(0xFF1E293B),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            TextField(
              controller: _brokerIpController,
              decoration: const InputDecoration(
                labelText: 'Broker IPv4 Address',
                border: OutlineInputBorder(),
                prefixIcon: Icon(Icons.dns),
              ),
            ),
            const SizedBox(height: 12),

            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: _statusColor.withOpacity(0.15),
                borderRadius: BorderRadius.circular(10),
                border: Border.all(color: _statusColor),
              ),
              child: Row(
                children: [
                  Icon(Icons.circle, color: _statusColor, size: 14),
                  const SizedBox(width: 8),
                  Text(
                    _statusText,
                    style: TextStyle(
                      color: _statusColor,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 12),

            Row(
              children: [
                Expanded(
                  child: ElevatedButton.icon(
                    onPressed: _connectToBroker,
                    icon: const Icon(Icons.link),
                    label: const Text('Connect'),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: const Color(0xFF00E5FF),
                      foregroundColor: Colors.black,
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: OutlinedButton.icon(
                    onPressed: _disconnectFromBroker,
                    icon: const Icon(Icons.link_off),
                    label: const Text('Disconnect'),
                  ),
                ),
              ],
            ),
            const SizedBox(height: 8),

            SizedBox(
              width: double.infinity,
              child: ElevatedButton.icon(
                onPressed: _publishTestMessage,
                icon: const Icon(Icons.send),
                label: const Text('Publish Test Message (Ping)'),
                style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF00E676),
                  foregroundColor: Colors.black,
                ),
              ),
            ),
            const SizedBox(height: 16),

            const Align(
              alignment: Alignment.centerLeft,
              child: Text(
                'บันทึกการทำงาน (Transaction Logs):',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
            ),
            const SizedBox(height: 8),

            Expanded(
              child: Container(
                padding: const EdgeInsets.all(10),
                decoration: BoxDecoration(
                  color: Colors.black26,
                  borderRadius: BorderRadius.circular(8),
                  border: Border.all(color: Colors.white12),
                ),
                child: ListView.builder(
                  itemCount: _receivedLogs.length,
                  itemBuilder: (context, index) => Text(
                    _receivedLogs[index],
                    style: const TextStyle(
                      fontFamily: 'monospace',
                      fontSize: 12,
                      color: Colors.white70,
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

### กิจกรรมที่ 12-1.3: การทดสอบร่วมกับ Mosquitto CLI

**1. เปิด Mosquitto Broker**

เปิด PowerShell แล้วรันคำสั่ง

```powershell
mosquitto -v -p 1883
```

สำหรับ Mosquitto เวอร์ชัน 2.x หากไม่ได้กำหนดไฟล์ Config จะรับการเชื่อมต่อผ่าน `127.0.0.1` เท่านั้น

**2. เปิดหน้าต่าง PowerShell อีกหน้าต่าง เพื่อรับข้อความจาก Flutter**

```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "test/flutter/ping" -v
```

**3. สั่งรัน Flutter Application**

```powershell
flutter run -d windows
```

หรือหากใช้อุปกรณ์ Android

```powershell
flutter run -d <device-id>
```

หมายเหตุ: ไม่ควรรันแล็บนี้ผ่าน Chrome เนื่องจากโค้ดใช้ `MqttServerClient` ซึ่งทำงานผ่าน TCP Socket และไม่รองรับการทำงานบน Web Browser ตามรูปแบบนี้

**4. ทดลองรับและส่งข้อความ**

- เปิดแอป Flutter แล้วกรอก IP Address ของ Broker
- กดปุ่ม Connect เพื่อเริ่มเชื่อมต่อ
- เมื่อแสดงสถานะ Connected ให้กด Publish Test Message
- ตรวจสอบข้อความที่ส่งออกไปในหน้าต่าง PowerShell ที่เปิด `mosquitto_sub`
- เปิด PowerShell อีกหน้าต่างแล้วทดลองส่งข้อความกลับมายัง Flutter ด้วยคำสั่ง

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "test/broker/echo" -m "Echo response from CLI"
```

- กลับไปตรวจสอบข้อความที่ได้รับในส่วน Transaction Logs ของแอปพลิเคชัน

---

## 4. คำถามท้ายการทดลอง (Review Questions)

**1. เพราะเหตุใดเราจึงต้องสุ่มชื่อ `clientId` เช่นการใส่ Timestamp ต่อท้าย แทนที่จะตั้งชื่อแบบคงที่?**

การกำหนด clientId ให้แตกต่างกันช่วยป้องกันปัญหาการใช้รหัสซ้ำระหว่างอุปกรณ์ที่เชื่อมต่อกับ Broker เนื่องจากหากมี Client ใหม่เข้ามาโดยใช้ ID เดียวกับ Client เดิม อาจทำให้การเชื่อมต่อของตัวเดิมถูกตัดออก การเพิ่ม Timestamp จึงช่วยสร้างรหัสที่ไม่ซ้ำกันในแต่ละครั้ง

**2. หากหน้าจอ Flutter รันบนอุปกรณ์จริง (Physical Android Phone) เหตุใดการกรอก IP `127.0.0.1` จึงไม่สามารถเชื่อมต่อ Broker ได้? และต้องใช้ IP ใด?**

เนื่องจาก 127.0.0.1 เป็น IP ที่ใช้อ้างอิงกลับมายังอุปกรณ์ของตัวเอง เมื่อเปิดแอปบนโทรศัพท์จึงหมายถึงโทรศัพท์เครื่องนั้น ไม่ใช่คอมพิวเตอร์ที่ทำหน้าที่เป็น Broker ดังนั้นต้องเปลี่ยนเป็น IP Address ของคอมพิวเตอร์ที่ติดตั้ง Mosquitto และตรวจสอบให้ทั้งสองอุปกรณ์อยู่ในเครือข่ายเดียวกัน

**3. การตั้งค่า `keepAlivePeriod` มีความจำเป็นอย่างไรในสภาพแวดล้อมเครือข่ายไร้สาย?**

เป็นการกำหนดช่วงเวลาให้ Client ส่งสัญญาณเพื่อตรวจสอบว่ายังสามารถติดต่อกับ Broker ได้ตามปกติ หากเกิดปัญหาเครือข่ายหรืออุปกรณ์หยุดตอบสนอง Broker จะสามารถตรวจสอบและรับรู้ได้ว่าการเชื่อมต่อมีปัญหา

---

## 5. สรุปผลการทดลอง

จากการทดลองได้เรียนรู้การสร้างโปรเจกต์ Flutter การติดตั้งแพ็กเกจ mqtt_client และการเขียนโปรแกรมเพื่อเชื่อมต่อกับ Mosquitto Broker ผ่านโปรโตคอล MQTT รวมถึงได้ทดลองส่งและรับข้อความระหว่างแอปพลิเคชันกับ Mosquitto CLI ทำให้เข้าใจหลักการสื่อสารระหว่าง Client และ Broker มากขึ้น
