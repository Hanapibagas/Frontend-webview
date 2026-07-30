# Implementasi Jadwal Sholat WebView dengan Notifikasi Background

## 📋 Table of Contents
1. [Overview](#overview)
2. [Arsitektur](#arsitektur)
3. [Struktur Project](#struktur-project)
4. [Setup Flutter](#1-setup-flutter)
5. [Implementasi Backend](#2-implementasi-backend-flutter)
6. [Implementasi Frontend](#3-implementasi-frontend-html)
7. [Testing](#4-testing)
8. [Deployment](#5-deployment)
9. [Troubleshooting](#troubleshooting)

---

## Overview

Dokumentasi ini menjelaskan cara mengimplementasikan aplikasi WebView Jadwal Sholat dengan Flutter yang memiliki fitur:
- ✅ Tampilan jadwal sholat dari HTML/JavaScript
- ✅ Deteksi lokasi otomatis
- ✅ Notifikasi background saat waktu sholat tiba
- ✅ Suara adzan
- ✅ Notifikasi muncul meski app ditutup/device sleep

### Teknologi yang Digunakan
- **Flutter** - Framework untuk mobile app
- **Flutter Local Notifications** - Plugin notifikasi lokal
- **Workmanager** - Background task scheduling
- **InAppWebView** - WebView Flutter dengan JavaScript bridge
- **HTML/JavaScript** - UI dan logika frontend

---

## Arsitektur

```
┌─────────────────────────────────────────────────────────────┐
│                    FLUTTER APP                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              InAppWebView                             │  │
│  │  ┌────────────────────────────────────────────────┐   │  │
│  │  │         index.html (JavaScript)                │   │  │
│  │  │                                                  │   │  │
│  │  │  • UI Jadwal Sholat                            │   │  │
│  │  │  • Geolocation API                             │   │  │
│  │  │  • Prayer Times API                           │   │  │
│  │  │  • Countdown Timer                            │   │  │
│  │  │  • Tampilan jadwal                            │   │  │
│  │  └────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           │ JavaScript Bridge                 │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Notification Service                        │  │
│  │  • Schedule notification                           │  │
│  │  • Cancel notification                             │  │
│  │  • Update settings                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Flutter Local Notifications                 │  │
│  │  • Show notification di system                      │  │
│  │  • Play adzan audio                                │  │
│  │  • Handle notification taps                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Workmanager (Background)                     │  │
│  │  • Refresh jadwal sholat harian                    │  │
│  │  • Re-schedule notifications                       │  │
│  │  • Sync jadwal per jam 12 malam                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    ANDROID/iOS SYSTEM                         │
│  • Notification Bar                                          │
│  • Lock Screen Notification                                  │
│  • Sound & Vibration                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Struktur Project

```
lib/
├── main.dart                          # Entry point app
├── app.dart                           # Root app widget
├── screens/
│   ├── home_screen.dart               # Home screen dengan WebView
│   └── webview_screen.dart           # WebView screen
├── services/
│   ├── notification_service.dart      # Handle notifications
│   ├── prayer_times_service.dart     # API & data jadwal
│   ├── location_service.dart         # Geolocation
│   └── storage_service.dart          # Local storage
├── models/
│   ├── prayer_time.dart               # Model jadwal sholat
│   └── location.dart                 # Model lokasi
├── utils/
│   ├── date_utils.dart               # Helper tanggal
│   └── constants.dart                # Constants app
└── workers/
    └── prayer_sync_worker.dart       # Background worker

assets/
├── html/
│   └── index.html                    # WebView HTML/JS/CSS
├── audio/
│   ├── adzan_mishary.mp3            # Audio adzan
│   └── notification.mp3              # Sound notifikasi
└── images/
    └── notification_icon.png         # Icon notifikasi
```

---

## 1. Setup Flutter

### 1.1. Buat Project Baru

```bash
flutter create prayer_times_app
cd prayer_times_app
```

### 1.2. Update `pubspec.yaml`

```yaml
name: prayer_times_app
description: Aplikasi Jadwal Sholat dengan WebView

version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  # WebView
  flutter_inappwebview: ^6.0.0

  # Notifications
  flutter_local_notifications: ^16.3.0

  # Background tasks
  workmanager: ^0.5.0

  # Storage
  shared_preferences: ^2.2.0

  # Location
  geolocator: ^10.1.0

  # HTTP & API
  http: ^1.1.0

  # State management
  provider: ^6.1.0

  # Utils
  timezone: ^0.9.0
  intl: ^0.18.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true

  assets:
    - assets/html/
    - assets/audio/
    - assets/images/
```

### 1.3. Install Dependencies

```bash
flutter pub get
```

### 1.4. Setup Platform

#### **Android Setup**

Edit `android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.prayer_times_app">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.VIBRATE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />

    <application
        android:label="Jadwal Sholat"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher"
        android:requestLegacyExternalStorage="true">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">

            <meta-data
              android:name="io.flutter.embedding.android.NormalTheme"
              android:resource="@style/NormalTheme" />

            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <!-- Notification Service -->
        <service
            android:name="com.dexterous.flutterlocalnotifications.LocalNotificationService"
            android:exported="false"
            android:priority="-1" />

        <!-- WorkManager Service -->
        <service
            android:name="androidx.work.impl.WorkManagerInitializer"
            android:enabled="true"
            android:exported="false" />
    </application>
</manifest>
```

#### **iOS Setup**

Edit `ios/Runner/Info.plist`:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>We need your location to calculate prayer times</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>We need your location to calculate prayer times</string>
<key>UIBackgroundModes</key>
<array>
    <string>remote-notification</string>
</array>
```

---

## 2. Implementasi Backend (Flutter)

### 2.1. Models

**`lib/models/prayer_time.dart`**

```dart
class PrayerTime {
  final String name;
  final String arabicName;
  final String time;
  final DateTime dateTime;
  bool notified;

  PrayerTime({
    required this.name,
    required this.arabicName,
    required this.time,
    required this.dateTime,
    this.notified = false,
  });

  Map<String, dynamic> toJson() {
    return {
      'name': name,
      'arabicName': arabicName,
      'time': time,
      'dateTime': dateTime.toIso8601String(),
      'notified': notified,
    };
  }

  factory PrayerTime.fromJson(Map<String, dynamic> json) {
    return PrayerTime(
      name: json['name'],
      arabicName: json['arabicName'],
      time: json['time'],
      dateTime: DateTime.parse(json['dateTime']),
      notified: json['notified'] ?? false,
    );
  }
}

class PrayerSchedule {
  final String location;
  final DateTime date;
  final List<PrayerTime> prayers;
  final String hijriDate;

  PrayerSchedule({
    required this.location,
    required this.date,
    required this.prayers,
    required this.hijriDate,
  });
}
```

### 2.2. Notification Service

**`lib/services/notification_service.dart`**

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest.dart' as tz_data;
import 'package:rxdart/subjects.dart';
import '../models/prayer_time.dart';

class NotificationService {
  static final NotificationService _instance = NotificationService._internal();
  factory NotificationService() => _instance;
  NotificationService._internal();

  final FlutterLocalNotificationsPlugin _notifications =
      FlutterLocalNotificationsPlugin();

  final BehaviorSubject<String> notificationResponse = BehaviorSubject();

  // Initialization
  Future<void> initialize() async {
    // Setup timezone
    tz_data.initializeTimeZones();

    // Android settings
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');

    // iOS settings
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    );

    const settings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );

    await _notifications.initialize(
      settings,
      onDidReceiveNotificationResponse: (response) {
        notificationResponse.add(response.payload ?? '');
      },
    );
  }

  // Request permissions
  Future<bool> requestPermissions() async {
    final android = await _notifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();

    if (android != null) {
      final granted = await android.requestPermission();
      return granted;
    }

    return true;
  }

  // Schedule prayer notification
  Future<void> schedulePrayerNotification({
    required PrayerTime prayer,
    required String sound,
  }) async {
    await _notifications.zonedSchedule(
      prayer.dateTime.hashCode,
      'Waktu ${prayer.name} Telah Tiba',
      'Segera lakukan sholat ${prayer.name}. Semangat ibadah! 🕌',
      TZDateTime.from(prayer.dateTime, tz.local),
      NotificationDetails(
        android: AndroidNotificationDetails(
          'prayer_times',
          'Jadwal Sholat',
          channelDescription: 'Notifikasi waktu sholat harian',
          importance: Importance.high,
          priority: Priority.high,
          sound: RawResourceAndroidNotificationSound(sound),
          playSound: true,
          enableVibration: true,
          vibrationPattern: [0, 1000, 500, 1000],
          icon: '@mipmap/ic_launcher',
          largeIcon: const DrawableResourceAndroidBitmap('@mipmap/ic_launcher'),
          styleInformation: const BigTextStyleInformation(''),
          fullScreenIntent: true,
        ),
        iOS: const DarwinNotificationDetails(
          sound: sound,
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      androidAllowWhileIdle: true,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
      matchDateTimeComponents: DateTimeComponents.time,
    );
  }

  // Cancel specific notification
  Future<void> cancelNotification(int id) async {
    await _notifications.cancel(id);
  }

  // Cancel all notifications
  Future<void> cancelAllNotifications() async {
    await _notifications.cancelAll();
  }

  // Get pending notifications
  Future<List<PendingNotificationRequest>> getPendingNotifications() async {
    return await _notifications.pendingNotificationRequests();
  }
}
```

### 2.3. Prayer Times Service

**`lib/services/prayer_times_service.dart`**

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/prayer_time.dart';

class PrayerTimesService {
  static const String _baseUrl = 'https://api.aladhan.com/v1';

  // Get prayer times by coordinates
  Future<PrayerSchedule> getPrayerTimes({
    required double latitude,
    required double longitude,
    DateTime? date,
  }) async {
    date ??= DateTime.now();

    final params = {
      'latitude': latitude.toString(),
      'longitude': longitude.toString(),
      'method': '20', // Kemenag RI
      'school': '1', // Shafi
    };

    final response = await http.get(
      Uri.parse('$_baseUrl/timings')
          .replace(queryParameters: params),
    );

    if (response.statusCode == 200) {
      final data = json.decode(response.body);

      if (data['code'] == 200) {
        return _parsePrayerSchedule(data['data'], latitude, longitude);
      }
    }

    throw Exception('Failed to load prayer times');
  }

  PrayerSchedule _parsePrayerSchedule(
    Map<String, dynamic> data,
    double latitude,
    double longitude,
  ) {
    final timings = data['timings'] as Map<String, dynamic>;
    final date = data['date'] as Map<String, dynamic>;
    final hijri = date['hijri'] as Map<String, dynamic>;

    final now = DateTime.now();
    final prayers = [
      _createPrayerTime('Fajr', 'Subuh', timings['Fajr'], now),
      _createPrayerTime('Dhuhr', 'Dzuhur', timings['Dhuhr'], now),
      _createPrayerTime('Asr', 'Ashar', timings['Asr'], now),
      _createPrayerTime('Maghrib', 'Maghrib', timings['Maghrib'], now),
      _createPrayerTime('Isha', 'Isya', timings['Isha'], now),
    ];

    return PrayerSchedule(
      location: 'Location ($latitude, $longitude)',
      date: now,
      prayers: prayers,
      hijriDate: '${hijri['day']} ${hijri['month']['en']} ${hijri['year']}',
    );
  }

  PrayerTime _createPrayerTime(
    String key,
    String name,
    String time,
    DateTime date,
  ) {
    final parts = time.split(':');
    final prayerDateTime = DateTime(
      date.year,
      date.month,
      date.day,
      int.parse(parts[0]),
      int.parse(parts[1]),
    );

    return PrayerTime(
      name: name,
      arabicName: key,
      time: time,
      dateTime: prayerDateTime,
    );
  }
}
```

### 2.4. WebView Screen

**`lib/screens/webview_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';
import 'dart:convert';
import '../services/notification_service.dart';
import '../services/prayer_times_service.dart';

class WebViewScreen extends StatefulWidget {
  const WebViewScreen({Key? key}) : super(key: key);

  @override
  State<WebViewScreen> createState() => _WebViewScreenState();
}

class _WebViewScreenState extends State<WebViewScreen> {
  late InAppWebViewController _webViewController;
  final NotificationService _notificationService = NotificationService();
  final PrayerTimesService _prayerService = PrayerTimesService();

  bool _isLoading = true;
  String? _errorMessage;

  @override
  void initState() {
    super.initState();
    _initializeNotifications();
  }

  Future<void> _initializeNotifications() async {
    await _notificationService.initialize();
    await _notificationService.requestPermissions();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Jadwal Sholat'),
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: _showSettings,
          ),
        ],
      ),
      body: Stack(
        children: [
          InAppWebView(
            initialFile: "assets/html/index.html",
            initialOptions: InAppWebViewGroupOptions(
              crossPlatform: InAppWebViewOptions(
                useShouldOverrideUrlLoading: true,
                mediaPlaybackRequiresUserGesture: false,
                javaScriptEnabled: true,
              ),
              android: AndroidInAppWebViewOptions(
                geolocationEnabled: true,
              ),
            ),
            onWebViewCreated: (controller) {
              _webViewController = controller;
              _setupJavaScriptHandlers();
            },
            onLoadStop: (controller, url) {
              setState(() => _isLoading = false);
            },
            onConsoleMessage: (controller, consoleMessage) {
              print('Console: ${consoleMessage.message}');
            },
            onLoadError: (controller, url, code, message) {
              setState(() {
                _errorMessage = 'Error: $message';
                _isLoading = false;
              });
            },
          ),
          if (_isLoading)
            const Center(
              child: CircularProgressIndicator(),
            ),
          if (_errorMessage != null)
            Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.error, size: 48, color: Colors.red),
                  const SizedBox(height: 16),
                  Text(_errorMessage!),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: () {
                      setState(() {
                        _errorMessage = null;
                        _isLoading = true;
                      });
                      _webViewController.reload();
                    },
                    child: const Text('Retry'),
                  ),
                ],
              ),
            ),
        ],
      ),
    );
  }

  void _setupJavaScriptHandlers() {
    // Handler untuk schedule prayer notification
    _webViewController.addJavaScriptHandler(
      handlerName: 'schedulePrayer',
      callback: (args) async {
        try {
          final data = json.decode(args[0]);

          // Parse prayer data
          final prayer = PrayerTime(
            name: data['name'],
            arabicName: data['arabicName'],
            time: data['time'],
            dateTime: DateTime.parse(data['dateTime']),
          );

          // Schedule notification
          await _notificationService.schedulePrayerNotification(
            prayer: prayer,
            sound: 'adzan_mishary',
          );

          return {'success': true};
        } catch (e) {
          return {'success': false, 'error': e.toString()};
        }
      },
    );

    // Handler untuk cancel notification
    _webViewController.addJavaScriptHandler(
      handlerName: 'cancelPrayer',
      callback: (args) async {
        try {
          final id = args[0] as int;
          await _notificationService.cancelNotification(id);
          return {'success': true};
        } catch (e) {
          return {'success': false, 'error': e.toString()};
        }
      },
    );

    // Handler untuk get location from native
    _webViewController.addJavaScriptHandler(
      handlerName: 'getLocation',
      callback: (args) async {
        // Implement native location here
        return {
          'latitude': -6.2088, // Example: Jakarta
          'longitude': 106.8456,
        };
      },
    );
  }

  void _showSettings() {
    Navigator.of(context).pushNamed('/settings');
  }
}
```

### 2.5. Background Worker

**`lib/workers/prayer_sync_worker.dart`**

```dart
import 'package:flutter_workmanager/flutter_workmanager.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../services/notification_service.dart';
import '../services/prayer_times_service.dart';

void prayerSyncDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    switch (task) {
      case 'syncPrayerTimes':
        await _syncPrayerTimes();
        return true;
      default:
        return false;
    }
  });
}

Future<void> _syncPrayerTimes() async {
  final prefs = await SharedPreferences.getInstance();

  // Get saved location
  final latitude = prefs.getDouble('latitude') ?? -6.2088;
  final longitude = prefs.getDouble('longitude') ?? 106.8456;

  // Get prayer times
  final prayerService = PrayerTimesService();
  final schedule = await prayerService.getPrayerTimes(
    latitude: latitude,
    longitude: longitude,
  );

  // Schedule notifications for each prayer
  final notificationService = NotificationService();
  await notificationService.initialize();

  for (final prayer in schedule.prayers) {
    // Only schedule if prayer time is in the future
    if (prayer.dateTime.isAfter(DateTime.now())) {
      await notificationService.schedulePrayerNotification(
        prayer: prayer,
        sound: 'adzan_mishary',
      );
    }
  }

  // Save last sync time
  await prefs.setString('lastSync', DateTime.now().toIso8601String());
}
```

### 2.6. Main App

**`lib/main.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_workmanager/flutter_workmanager.dart';
import 'package:provider/provider.dart';
import 'screens/webview_screen.dart';
import 'workers/prayer_sync_worker.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize Workmanager
  await Workmanager().initialize(
    prayerSyncDispatcher,
    isInDebugMode: false,
  );

  // Schedule daily sync at midnight
  await Workmanager().registerPeriodicTask(
    'syncPrayerTimes',
    'syncPrayerTimes',
    frequency: const Duration(hours: 24),
    initialDelay: Duration(
      hours: 24 - DateTime.now().hour,
      minutes: 60 - DateTime.now().minute,
    ),
    constraints: Constraints(
      networkType: NetworkType.connected,
    ),
  );

  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Jadwal Sholat',
      theme: ThemeData(
        primarySwatch: Colors.teal,
        useMaterial3: true,
      ),
      home: const WebViewScreen(),
    );
  }
}
```

---

## 3. Implementasi Frontend (HTML)

### 3.1. Update `index.html`

Hapus atau komentari bagian Web Notifications yang tidak jalan di WebView:

```javascript
// ❌ HAPUS/KOMENTARI - Tidak jalan di WebView tertutup
/*
if (Notification.permission === 'granted') {
    const notification = new Notification(title, {...});
}
*/

// ✅ GANTI dengan komunikasi ke Flutter
function schedulePrayerNotification(prayer) {
    if (window.flutter_inappwebview) {
        window.flutter_inappwebview.callHandler('schedulePrayer', {
            name: prayer.name,
            arabicName: prayer.arabicName,
            time: prayer.time,
            dateTime: prayer.dateTime.toISOString()
        }).then((result) => {
            console.log('Notification scheduled:', result);
        });
    } else {
        console.warn('Flutter WebView not available');
    }
}
```

### 3.2. Update PrayerTimesComponent

```javascript
/**
 * Send notification for prayer time
 */
async sendNotification(prayer) {
    // Kirim ke Flutter untuk schedule notifikasi native
    schedulePrayerNotification({
        name: prayer.name,
        arabicName: prayer.key,
        time: this.prayerTimes[prayer.key],
        dateTime: this.getPrayerDateTime(prayer)
    });

    // Show toast di app
    UI.toast(`Waktu ${prayer.name} telah tiba!`);

    // Play adzan audio if enabled (di WebView)
    if (this.audioEnabled) {
        this.playAdzan(prayer);
    }
}

/**
 * Get prayer date time
 */
getPrayerDateTime(prayer) {
    const time = this.prayerTimes[prayer.key];
    const [hours, minutes] = time.split(':').map(Number);
    const now = new Date();

    const prayerDate = new Date(
        now.getFullYear(),
        now.getMonth(),
        now.getDate(),
        hours,
        minutes,
        0
    );

    // If prayer time has passed, schedule for tomorrow
    if (prayerDate <= now) {
        prayerDate.setDate(prayerDate.getDate() + 1);
    }

    return prayerDate;
}
```

---

## 4. Testing

### 4.1. Test WebView & UI

```bash
flutter run
```

**Checklist:**
- [ ] WebView load index.html dengan benar
- [ ] Geolocation berjalan
- [ ] Jadwal sholat tampil
- [ ] Countdown timer jalan
- [ ] JavaScript handler ke Flutter berjalan

### 4.2. Test Notifications

**Test Manual:**
```bash
# Tap Settings → Test Notification
# Atau tunggu sampai waktu sholat
```

**Test dengan ADB:**
```bash
# Trigger notification manual
adb shell am broadcast -a android.intent.action.BOOT_COMPLETED
```

### 4.3. Test Background

1. Buka app
2. Tunggu jadwal sholat dimuat
3. Tekan Home button (app di background)
4. Tunggu sampai waktu sholat
5. Check apakah notifikasi muncul

### 4.4. Test Audio

- [ ] Adzan audio bermain saat notifikasi
- [ ] Audio bisa distop/pause
- [ ] Audio jalan di background

---

## 5. Deployment

### 5.1. Android Release

```bash
# Build APK
flutter build apk --release

# Build App Bundle (rekomendasi untuk Play Store)
flutter build appbundle --release
```

### 5.2. iOS Release

```bash
# Build iOS
flutter build ios --release
```

### 5.3. Setup Play Store

1. Buat signing key
2. Upload app bundle
3. Setup store listing
4. Add screenshot dan icon

---

## Troubleshooting

### Problem: Notification tidak muncul

**Solution:**
```dart
// Check permissions
await _notificationService.requestPermissions();

// Check pending notifications
final pending = await _notificationService.getPendingNotifications();
print('Pending: ${pending.length}');
```

### Problem: WebView tidak load

**Solution:**
```bash
# Pastikan file ada di assets
flutter:
  assets:
    - assets/html/index.html

# Check pubspec.yaml
flutter pub get
```

### Problem: Audio tidak jalan

**Solution:**
```dart
// Pastikan file audio ada
assets/audio/adzan_mishary.mp3

// Check sound setup
sound: RawResourceAndroidNotificationSound('adzan_mishary')
```

### Problem: Timezone salah

**Solution:**
```dart
// Setup timezone di main()
tz_data.initializeTimeZones();
tz.setLocalLocation(tz.getLocation('Asia/Jakarta'));
```

### Problem: Background worker tidak jalan

**Solution:**
```dart
// Check Workmanager initialization
await Workmanager().initialize(
    prayerSyncDispatcher,
    isInDebugMode: true, // Set true untuk testing
);
```

---

## Resources

- [Flutter Local Notifications](https://pub.dev/packages/flutter_local_notifications)
- [Workmanager](https://pub.dev/packages/workmanager)
- [InAppWebView](https://pub.dev/packages/flutter_inappwebview)
- [Aladhan API](https://aladhan.com/prayer-times-api)

---

## License

MIT License - Feel free to use this code for your project! 🚀
