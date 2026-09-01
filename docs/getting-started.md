name: assignment_reminder
description: Assignment reminder app
publish_to: "none"

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  path_provider: ^2.1.5
  flutter_local_notifications: ^18.0.1
  timezone: ^0.10.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  hive_generator: ^2.0.1
  build_runner: ^2.4.13 
import 'package:hive/hive.dart';

part 'assignment.g.dart';

@HiveType(typeId: 0)
class Assignment extends HiveObject {
  @HiveField(0)
  String title;

  @HiveField(1)
  String subject;

  @HiveField(2)
  DateTime dueDate;

  @HiveField(3)
  bool done;

  Assignment({
    required this.title,
    required this.subject,
    required this.dueDate,
    this.done = false,
  });
}import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest.dart' as tz;

class NotificationService {
  NotificationService._();
  static final NotificationService instance = NotificationService._();

  final FlutterLocalNotificationsPlugin plugin =
      FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    tz.initializeTimeZones();

    const android = AndroidInitializationSettings('@mipmap/ic_launcher');
    const ios = DarwinInitializationSettings();

    const settings = InitializationSettings(android: android, iOS: ios);
    await plugin.initialize(settings);

    final androidImpl =
        plugin.resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();
    await androidImpl?.requestNotificationsPermission();
  }

  Future<void> scheduleReminder({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledDate,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      'assignment_channel',
      'Assignment Reminders',
      channelDescription: 'Notifications for assignment deadlines',
      importance: Importance.high,
      priority: Priority.high,
    );

    const details = NotificationDetails(
      android: androidDetails,
      iOS: DarwinNotificationDetails(),
    );

    await plugin.zonedSchedule(
      id,
      title,
      body,
      tz.TZDateTime.from(scheduledDate, tz.local),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }
}import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'models/assignment.dart';
import 'services/notification_service.dart';
import 'pages/home_page.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  Hive.registerAdapter(AssignmentAdapter());
  await Hive.openBox<Assignment>('assignments');
  await NotificationService.instance.init();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Assignment Reminder',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}
import 'package:flutter/material.dart';
import 'package:hive/hive.dart';
import '../models/assignment.dart';
import '../services/notification_service.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final box = Hive.box<Assignment>('assignments');
  final titleController = TextEditingController();
  final subjectController = TextEditingController();
  DateTime? pickedDateTime;

  Future<void> pickDateTime() async {
    final date = await showDatePicker(
      context: context,
      firstDate: DateTime.now(),
      lastDate: DateTime(2100),
      initialDate: DateTime.now(),
    );
    if (date == null) return;

    final time = await showTimePicker(
      context: context,
      initialTime: TimeOfDay.now(),
    );
    if (time == null) return;

    setState(() {
      pickedDateTime = DateTime(
        date.year,
        date.month,
        date.day,
        time.hour,
        time.minute,
      );
    });
  }

  Future<void> addAssignment() async {
    if (titleController.text.isEmpty ||
        subjectController.text.isEmpty ||
        pickedDateTime == null) return;

    final item = Assignment(
      title: titleController.text,
      subject: subjectController.text,
      dueDate: pickedDateTime!,
    );

    await box.add(item);

    final id = DateTime.now().millisecondsSinceEpoch.remainder(100000);
    await NotificationService.instance.scheduleReminder(
      id: id,
      title: 'ถึงเวลาส่งงาน',
      body: '${item.subject} - ${item.title}',
      scheduledDate: item.dueDate,
    );

    titleController.clear();
    subjectController.clear();
    setState(() {
      pickedDateTime = null;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Assignment Reminder')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              controller: subjectController,
              decoration: const InputDecoration(
                labelText: 'วิชา',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 12),
            TextField(
              controller: titleController,
              decoration: const InputDecoration(
                labelText: 'ชื่องาน',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 12),
            Row(
              children: [
                Expanded(
                  child: Text(
                    pickedDateTime == null
                        ? 'ยังไม่ได้เลือกวันเวลา'
                        : 'เตือน: ${pickedDateTime.toString()}',
                  ),
                ),
                ElevatedButton(
                  onPressed: pickDateTime,
                  child: const Text('เลือกวันเวลา'),
                ),
              ],
            ),
            const SizedBox(height: 12),
            SizedBox(
              width: double.infinity,
              child: ElevatedButton(
                onPressed: addAssignment,
                child: const Text('บันทึกงาน'),
              ),
            ),
            const SizedBox(height: 20),
            Expanded(
              child: ValueListenableBuilder(
                valueListenable: box.listenable(),
                builder: (context, Box<Assignment> box, _) {
                  final items = box.values.toList().reversed.toList();
                  return ListView.builder(
                    itemCount: items.length,
                    itemBuilder: (_, i) {
                      final a = items[i];
                      return Card(
                        child: ListTile(
                          title: Text(a.title),
                          subtitle: Text('${a.subject}
${a.dueDate}'),
                          isThreeLine: true,
                        ),
                      );
                    },
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }
}<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
