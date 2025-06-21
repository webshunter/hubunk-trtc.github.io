# TUICallKit Flutter - Dokumentasi Lengkap & Contoh Penggunaan

## 📋 Daftar Isi
1. [Pendahuluan](#pendahuluan)
2. [Persiapan & Instalasi](#persiapan--instalasi)
3. [Konfigurasi](#konfigurasi)
4. [Implementasi Dasar](#implementasi-dasar)
5. [Contoh Penggunaan](#contoh-penggunaan)
6. [API Reference](#api-reference)
7. [Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)

## 🎯 Pendahuluan

TUICallKit adalah komponen UI lengkap untuk fitur panggilan audio dan video yang disediakan oleh Tencent Cloud. Komponen ini menyediakan UI siap pakai untuk aplikasi Flutter dengan fitur panggilan yang lengkap.

### Fitur Utama
- ✅ Panggilan Audio & Video 1-on-1
- ✅ Panggilan Grup
- ✅ Floating Window
- ✅ Custom Ringtone
- ✅ Beauty Effects
- ✅ Cloud Recording
- ✅ Push Notification
- ✅ Network Quality Monitoring

## 🚀 Persiapan & Instalasi

### 1. Persyaratan Sistem
```yaml
Flutter: >= 3.0.0
Dart: >= 2.17.0
Android: API 21+
iOS: 11.0+
```

### 2. Instalasi Package
```bash
flutter pub add tencent_calls_uikit
```

### 3. Konfigurasi Environment Variables
Buat file `.env` di root project:
```env
TENCENT_LIVE_SDK_APP_ID="your_sdk_app_id"
TENCENT_LIVE_SDK_SECRET_KEY="your_secret_key"
```

## ⚙️ Konfigurasi

### 1. Konfigurasi Android

#### `android/app/build.gradle`
```gradle
android {
    defaultConfig {
        multiDexEnabled true
        minSdkVersion 21
    }
}
```

#### `android/app/proguard-rules.pro`
```proguard
-keep class com.tencent.** { *; }
```

### 2. Konfigurasi iOS

#### `ios/Runner/Info.plist`
```xml
<key>NSCameraUsageDescription</key>
<string>This app needs camera access for video calls</string>
<key>NSMicrophoneUsageDescription</key>
<string>This app needs microphone access for audio calls</string>
```

### 3. Konfigurasi MaterialApp

#### `lib/main.dart`
```dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';

MaterialApp(
  navigatorObservers: [TUICallKit.navigatorObserver],
  // ... konfigurasi lainnya
)
```

## 🔧 Implementasi Dasar

### 1. Service Class untuk TUICallKit

```dart
// lib/services/tencent_call_service.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:tencent_calls_uikit/debug/generate_test_user_sig.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';

class TencentCallService {
  static final TencentCallService _instance = TencentCallService._internal();
  bool _isInitialized = false;
  bool _isLoggedIn = false;
  Timer? _keepAliveTimer;

  factory TencentCallService() => _instance;
  TencentCallService._internal();

  Future<bool> initialize() async {
    if (_isInitialized) return true;

    try {
      final userId = getStringAsync('userId');
      final userSig = getStringAsync('userSig');
      
      if (userId.isEmpty || userSig.isEmpty) {
        throw Exception('userId or userSig is empty');
      }

      final sdkAppId = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
      
      await TUICallKit.instance.login(sdkAppId, userId, userSig);
      
      _isInitialized = true;
      _isLoggedIn = true;
      _startKeepAliveTimer();
      
      return true;
    } catch (e) {
      print('Error initializing TencentCallService: $e');
      return false;
    }
  }

  Future<bool> checkLoginStatus() async {
    if (!_isInitialized || !_isLoggedIn) {
      return initialize();
    }
    return true;
  }

  Future<void> logout() async {
    try {
      await TUICallKit.instance.logout();
      _isInitialized = false;
      _isLoggedIn = false;
      _keepAliveTimer?.cancel();
    } catch (e) {
      print('Error during logout: $e');
    }
  }

  void _startKeepAliveTimer() {
    _keepAliveTimer?.cancel();
    _keepAliveTimer = Timer.periodic(const Duration(minutes: 5), (timer) {
      checkLoginStatus();
    });
  }

  bool get isInitialized => _isInitialized;
  bool get isLoggedIn => _isLoggedIn;
}
```

### 2. Widget untuk Memulai Panggilan

```dart
// lib/widgets/tencent_call_launcher.dart
import 'package:flutter/material.dart';
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:hub_unk/services/tencent_call_service.dart';

class TencentCallLauncher extends StatelessWidget {
  final String userId;
  final bool isVideo;

  const TencentCallLauncher({
    Key? key,
    required this.userId,
    required this.isVideo,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    Future.microtask(() async {
      try {
        final callService = TencentCallService();
        final isLoggedIn = await callService.checkLoginStatus();
        
        if (!isLoggedIn) {
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Failed to initialize call service')),
          );
          Navigator.pop(context);
          return;
        }

        // Menggunakan API baru yang direkomendasikan
        await TUICallKit.instance.calls(
          [userId],
          isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
        );

        if (context.mounted) {
          Navigator.pop(context);
        }
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Failed to make call: $e')),
        );
        if (context.mounted) {
          Navigator.pop(context);
        }
      }
    });

    return const Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 16),
            Text('Memulai panggilan...'),
          ],
        ),
      ),
    );
  }
}
```

## 📱 Contoh Penggunaan

### 1. Integrasi dengan Login

```dart
// lib/screens/auth/login_page.dart
import 'package:tencent_calls_uikit/debug/generate_test_user_sig.dart';
import 'package:hub_unk/services/tencent_call_service.dart';

class LoginPage extends StatefulWidget {
  @override
  _LoginPageState createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  Future<void> _handleLogin() async {
    try {
      // Login ke aplikasi Anda
      final loginResult = await yourLoginAPI();
      
      if (loginResult.success) {
        // Generate UserSig untuk TUICallKit
        final sdkAppID = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
        final secretKey = dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY'] ?? '';
        final userId = loginResult.userId;
        
        final userSig = GenerateTestUserSig.genTestSig(userId, sdkAppID, secretKey);
        
        // Simpan kredensial
        await setValue('userId', userId);
        await setValue('userSig', userSig);
        
        // Inisialisasi TUICallKit
        final callService = TencentCallService();
        await callService.initialize();
        
        // Navigate ke halaman utama
        Navigator.pushReplacementNamed(context, '/home');
      }
    } catch (e) {
      print('Login error: $e');
    }
  }
}
```

### 2. Tombol Panggilan di Chat

```dart
// lib/screens/chat/chat_page.dart
class ChatPage extends StatelessWidget {
  final String targetUserId;
  final String targetUserName;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(targetUserName),
        actions: [
          // Tombol Panggilan Audio
          IconButton(
            icon: const Icon(Icons.call),
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => TencentCallLauncher(
                    userId: targetUserId,
                    isVideo: false,
                  ),
                ),
              );
            },
          ),
          // Tombol Panggilan Video
          IconButton(
            icon: const Icon(Icons.videocam),
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => TencentCallLauncher(
                    userId: targetUserId,
                    isVideo: true,
                  ),
                ),
              );
            },
          ),
        ],
      ),
      body: ChatMessages(),
    );
  }
}
```

### 3. Panggilan Grup

```dart
// lib/screens/group_call/group_call_page.dart
class GroupCallPage extends StatelessWidget {
  final List<String> userIds;
  final bool isVideo;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Panggilan Grup')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Peserta: ${userIds.length}'),
            const SizedBox(height: 20),
            ElevatedButton(
              onPressed: () async {
                try {
                  final callService = TencentCallService();
                  await callService.checkLoginStatus();
                  
                  await TUICallKit.instance.calls(
                    userIds,
                    isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
                  );
                } catch (e) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('Error: $e')),
                  );
                }
              },
              child: Text('Mulai Panggilan ${isVideo ? 'Video' : 'Audio'}'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 4. Monitoring Status Panggilan

```dart
// lib/services/call_observer.dart
import 'package:tencent_calls_engine/tencent_calls_engine.dart';

class CallObserver {
  static void setupObserver() {
    TUICallObserver observer = TUICallObserver(
      onError: (int code, String message) {
        print('Call Error: $code - $message');
      },
      onCallBegin: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole) {
        print('Call started: Room $roomId, Type: $callMediaType, Role: $callRole');
      },
      onCallEnd: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole, double totalTime) {
        print('Call ended: Duration ${totalTime}s');
      },
      onUserNetworkQualityChanged: (List<TUINetworkQualityInfo> networkQualityList) {
        for (var quality in networkQualityList) {
          print('User ${quality.userId}: Quality ${quality.quality}');
        }
      },
      onCallReceived: (String callerId, List<String> calleeIdList, String groupId, TUICallMediaType callMediaType) {
        print('Incoming call from: $callerId');
      },
    );

    TUICallEngine.instance.addObserver(observer);
  }
}
```

## 📚 API Reference

### API Utama (v2.9+)

| Method | Description | Parameters |
|--------|-------------|------------|
| `login()` | Login ke TUICallKit | `sdkAppId`, `userId`, `userSig` |
| `logout()` | Logout dari TUICallKit | - |
| `calls()` | Memulai panggilan (1-on-1 atau grup) | `userIdList`, `mediaType`, `params?` |
| `join()` | Bergabung ke panggilan | `callId` |
| `setSelfInfo()` | Set info pengguna | `nickname`, `avatar` |

### Fitur Tambahan

| Method | Description |
|--------|-------------|
| `enableMuteMode()` | Aktifkan mode mute default |
| `enableFloatWindow()` | Aktifkan floating window |
| `setCallingBell()` | Set nada dering custom |
| `enableVirtualBackground()` | Aktifkan background virtual |

### API Deprecated (Jangan Gunakan)

- ❌ `call()` - Gunakan `calls()` sebagai gantinya
- ❌ `groupCall()` - Gunakan `calls()` sebagai gantinya
- ❌ `joinInGroupCall()` - Gunakan `join()` sebagai gantinya

## 🔧 Troubleshooting

### Error Umum

#### 1. "not buy the basic call package" (Error -1001)
**Penyebab:** Paket TRTC belum diaktifkan atau tidak valid
**Solusi:**
- Cek status paket di Tencent Cloud Console
- Pastikan SDKAppID benar
- Verifikasi pembayaran akun

#### 2. "not login" (Error -1)
**Penyebab:** Belum login ke TUICallKit
**Solusi:**
- Pastikan `login()` dipanggil sebelum `calls()`
- Cek UserSig tidak expired
- Verifikasi kredensial

#### 3. Permission Denied
**Penyebab:** Izin kamera/mikrofon tidak diberikan
**Solusi:**
- Tambahkan permission di Android/iOS
- Minta izin runtime di Android

### Debug Checklist

```dart
// Debug helper
void debugTUICallKit() {
  final callService = TencentCallService();
  print('Initialized: ${callService.isInitialized}');
  print('Logged In: ${callService.isLoggedIn}');
  print('SDK App ID: ${dotenv.env['TENCENT_LIVE_SDK_APP_ID']}');
  print('User ID: ${getStringAsync('userId')}');
  print('UserSig Length: ${getStringAsync('userSig').length}');
}
```

## 💡 Best Practices

### 1. Keamanan
- ✅ Generate UserSig di server (produksi)
- ✅ Jangan simpan SecretKey di aplikasi
- ✅ Gunakan HTTPS untuk komunikasi

### 2. Performance
- ✅ Inisialisasi service sekali di awal
- ✅ Gunakan keep-alive timer
- ✅ Handle lifecycle dengan benar

### 3. User Experience
- ✅ Tampilkan loading indicator
- ✅ Handle error dengan graceful
- ✅ Berikan feedback yang jelas

### 4. Code Organization
- ✅ Pisahkan service logic
- ✅ Gunakan dependency injection
- ✅ Implement proper error handling

## 🔗 Referensi

- [Dokumentasi Resmi TUICallKit](https://trtc.io/document/54906?platform=flutter&product=call&menulabel=uikit)
- [Tencent Cloud Console](https://console.tencentcloud.com/trtc)
- [GitHub Repository](https://github.com/Tencent-RTC/tls-sig-api-v2-flutter)

## 📞 Support

Jika mengalami masalah:
1. Cek dokumentasi resmi
2. Lihat FAQ di console Tencent
3. Hubungi support: info_rtc@tencent.com
4. Buat issue di GitHub repository

---

**Versi:** 3.1.1  
**Terakhir Update:** $(date)  
**Status:** Production Ready ✅ 