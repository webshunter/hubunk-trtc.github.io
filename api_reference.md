# TUICallKit Flutter - API Reference

## 📚 Overview

TUICallKit menyediakan API lengkap untuk implementasi fitur panggilan audio dan video di aplikasi Flutter. Dokumentasi ini mencakup semua API yang tersedia di versi 3.1.1.

## 🔧 Core API

### TUICallKit.instance

Singleton instance untuk mengakses semua fungsi TUICallKit.

```dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';

// Access instance
final callKit = TUICallKit.instance;
```

## 🔐 Authentication API

### login()

Login ke layanan TUICallKit.

```dart
Future<TUIResult> login(
  int sdkAppId,
  String userId,
  String userSig,
)
```

**Parameters:**
- `sdkAppId` (int): SDK App ID dari Tencent Cloud Console
- `userId` (String): ID unik pengguna
- `userSig` (String): Signature keamanan yang di-generate dari server

**Returns:**
- `TUIResult`: Object yang berisi status login

**Example:**
```dart
try {
  final result = await TUICallKit.instance.login(
    80000314, // SDK App ID
    "user123", // User ID
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...", // UserSig
  );
  
  if (result.code.isEmpty) {
    print('Login successful');
  } else {
    print('Login failed: ${result.code} - ${result.message}');
  }
} catch (e) {
  print('Login error: $e');
}
```

### logout()

Logout dari layanan TUICallKit.

```dart
Future<void> logout()
```

**Example:**
```dart
try {
  await TUICallKit.instance.logout();
  print('Logout successful');
} catch (e) {
  print('Logout error: $e');
}
```

## 📞 Call Management API

### calls() ⭐ **RECOMMENDED**

Memulai panggilan (1-on-1 atau grup). **API baru yang direkomendasikan.**

```dart
Future<void> calls(
  List<String> userIdList,
  TUICallMediaType mediaType, {
  TUICallParams? params,
})
```

**Parameters:**
- `userIdList` (List<String>): Daftar ID pengguna yang akan dipanggil
- `mediaType` (TUICallMediaType): Jenis panggilan (audio/video)
- `params` (TUICallParams?, optional): Parameter tambahan

**TUICallMediaType:**
```dart
enum TUICallMediaType {
  audio,  // Panggilan audio
  video,  // Panggilan video
}
```

**TUICallParams:**
```dart
class TUICallParams {
  final String? callId;        // ID panggilan custom
  final int? timeout;          // Timeout dalam detik
  final String? groupId;       // ID grup (untuk grup call)
  final Map<String, dynamic>? extraData; // Data tambahan
}
```

**Examples:**

**1-on-1 Audio Call:**
```dart
await TUICallKit.instance.calls(
  ['user123'],
  TUICallMediaType.audio,
);
```

**1-on-1 Video Call:**
```dart
await TUICallKit.instance.calls(
  ['user123'],
  TUICallMediaType.video,
);
```

**Group Call:**
```dart
await TUICallKit.instance.calls(
  ['user1', 'user2', 'user3'],
  TUICallMediaType.video,
  params: TUICallParams(
    groupId: 'group123',
    timeout: 30,
  ),
);
```

**Call with Custom Parameters:**
```dart
await TUICallKit.instance.calls(
  ['user123'],
  TUICallMediaType.video,
  params: TUICallParams(
    callId: 'custom_call_123',
    timeout: 60,
    extraData: {
      'callType': 'business',
      'priority': 'high',
    },
  ),
);
```

### join()

Bergabung ke panggilan yang sudah berlangsung.

```dart
Future<void> join(String callId)
```

**Parameters:**
- `callId` (String): ID unik dari panggilan yang ingin digabungi

**Example:**
```dart
try {
  await TUICallKit.instance.join('call_123456');
  print('Successfully joined call');
} catch (e) {
  print('Failed to join call: $e');
}
```

## 👤 User Management API

### setSelfInfo()

Mengatur informasi pengguna saat ini.

```dart
Future<void> setSelfInfo(
  String nickname,
  String avatar,
)
```

**Parameters:**
- `nickname` (String): Nama panggilan pengguna
- `avatar` (String): URL foto profil pengguna

**Example:**
```dart
await TUICallKit.instance.setSelfInfo(
  'John Doe',
  'https://example.com/avatar.jpg',
);
```

## ⚙️ Configuration API

### enableMuteMode()

Mengaktifkan/menonaktifkan mode mute default.

```dart
Future<void> enableMuteMode(bool enable)
```

**Parameters:**
- `enable` (bool): true untuk mengaktifkan mute default, false untuk menonaktifkan

**Example:**
```dart
// Aktifkan mute default
await TUICallKit.instance.enableMuteMode(true);

// Nonaktifkan mute default
await TUICallKit.instance.enableMuteMode(false);
```

### enableFloatWindow()

Mengaktifkan/menonaktifkan floating window.

```dart
Future<void> enableFloatWindow(bool enable)
```

**Parameters:**
- `enable` (bool): true untuk mengaktifkan floating window, false untuk menonaktifkan

**Example:**
```dart
// Aktifkan floating window
await TUICallKit.instance.enableFloatWindow(true);

// Nonaktifkan floating window
await TUICallKit.instance.enableFloatWindow(false);
```

### setCallingBell()

Mengatur nada dering custom.

```dart
Future<void> setCallingBell(String assetName)
```

**Parameters:**
- `assetName` (String): Nama file aset (harus ada di pubspec.yaml)

**Example:**
```dart
// Pastikan file ada di assets/audio/custom_ringtone.mp3
// dan terdaftar di pubspec.yaml
await TUICallKit.instance.setCallingBell('audio/custom_ringtone.mp3');
```

**pubspec.yaml configuration:**
```yaml
flutter:
  assets:
    - assets/audio/custom_ringtone.mp3
```

### enableVirtualBackground()

Mengaktifkan/menonaktifkan background virtual (blur effect).

```dart
Future<void> enableVirtualBackground(bool enable)
```

**Parameters:**
- `enable` (bool): true untuk mengaktifkan background virtual, false untuk menonaktifkan

**Example:**
```dart
// Aktifkan background blur
await TUICallKit.instance.enableVirtualBackground(true);

// Nonaktifkan background blur
await TUICallKit.instance.enableVirtualBackground(false);
```

## 🎯 Observer API

### TUICallObserver

Interface untuk menerima callback dari TUICallKit.

```dart
TUICallObserver observer = TUICallObserver(
  onError: (int code, String message) {
    // Handle error
  },
  onCallBegin: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole) {
    // Call started
  },
  onCallEnd: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole, double totalTime) {
    // Call ended
  },
  onUserNetworkQualityChanged: (List<TUINetworkQualityInfo> networkQualityList) {
    // Network quality changed
  },
  onCallReceived: (String callerId, List<String> calleeIdList, String groupId, TUICallMediaType callMediaType) {
    // Incoming call
  },
);
```

### addObserver()

Menambahkan observer untuk menerima callback.

```dart
TUICallEngine.instance.addObserver(observer);
```

### removeObserver()

Menghapus observer.

```dart
TUICallEngine.instance.removeObserver(observer);
```

## 📱 Navigation API

### navigatorObserver

Observer untuk navigasi yang harus ditambahkan ke MaterialApp.

```dart
MaterialApp(
  navigatorObservers: [TUICallKit.navigatorObserver],
  // ... other configuration
)
```

## 🚫 Deprecated API

### ⚠️ call() - DEPRECATED

**Jangan gunakan lagi.** Gunakan `calls()` sebagai gantinya.

```dart
// ❌ DEPRECATED - Jangan gunakan
await TUICallKit.instance.call(userId, mediaType);

// ✅ RECOMMENDED - Gunakan ini
await TUICallKit.instance.calls([userId], mediaType);
```

### ⚠️ groupCall() - DEPRECATED

**Jangan gunakan lagi.** Gunakan `calls()` dengan parameter grup.

```dart
// ❌ DEPRECATED - Jangan gunakan
await TUICallKit.instance.groupCall(userIds, mediaType, groupId);

// ✅ RECOMMENDED - Gunakan ini
await TUICallKit.instance.calls(userIds, mediaType, 
  params: TUICallParams(groupId: groupId));
```

### ⚠️ joinInGroupCall() - DEPRECATED

**Jangan gunakan lagi.** Gunakan `join()` sebagai gantinya.

```dart
// ❌ DEPRECATED - Jangan gunakan
await TUICallKit.instance.joinInGroupCall(callId);

// ✅ RECOMMENDED - Gunakan ini
await TUICallKit.instance.join(callId);
```

## 📊 Data Types

### TUIResult

Object yang berisi hasil operasi.

```dart
class TUIResult {
  final String code;      // Error code (empty if success)
  final String message;   // Error message
}
```

### TUIRoomId

ID ruang panggilan.

```dart
typedef TUIRoomId = String;
```

### TUICallRole

Role pengguna dalam panggilan.

```dart
enum TUICallRole {
  caller,   // Penelepon
  callee,   // Yang ditelepon
}
```

### TUINetworkQualityInfo

Informasi kualitas jaringan.

```dart
class TUINetworkQualityInfo {
  final String userId;    // ID pengguna
  final int quality;      // Kualitas (1-5, 1=terbaik, 5=terburuk)
}
```

## 🔄 Complete Example

```dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:tencent_calls_engine/tencent_calls_engine.dart';

class CallService {
  static final CallService _instance = CallService._internal();
  factory CallService() => _instance;
  CallService._internal();

  Future<void> initialize() async {
    // Setup observer
    TUICallObserver observer = TUICallObserver(
      onError: (int code, String message) {
        print('Call Error: $code - $message');
      },
      onCallBegin: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole) {
        print('Call started: Room $roomId');
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

    // Configure call settings
    await TUICallKit.instance.enableMuteMode(false);
    await TUICallKit.instance.enableFloatWindow(true);
    await TUICallKit.instance.enableVirtualBackground(true);
  }

  Future<bool> login(int sdkAppId, String userId, String userSig) async {
    try {
      final result = await TUICallKit.instance.login(sdkAppId, userId, userSig);
      return result.code.isEmpty;
    } catch (e) {
      print('Login error: $e');
      return false;
    }
  }

  Future<bool> makeCall(List<String> userIds, bool isVideo) async {
    try {
      await TUICallKit.instance.calls(
        userIds,
        isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
      );
      return true;
    } catch (e) {
      print('Call error: $e');
      return false;
    }
  }

  Future<bool> joinCall(String callId) async {
    try {
      await TUICallKit.instance.join(callId);
      return true;
    } catch (e) {
      print('Join call error: $e');
      return false;
    }
  }

  Future<void> logout() async {
    try {
      await TUICallKit.instance.logout();
    } catch (e) {
      print('Logout error: $e');
    }
  }
}
```

## 📋 API Summary

| API | Status | Description |
|-----|--------|-------------|
| `login()` | ✅ Active | Login ke TUICallKit |
| `logout()` | ✅ Active | Logout dari TUICallKit |
| `calls()` | ✅ Active | Memulai panggilan (recommended) |
| `join()` | ✅ Active | Bergabung ke panggilan |
| `setSelfInfo()` | ✅ Active | Set info pengguna |
| `enableMuteMode()` | ✅ Active | Konfigurasi mute default |
| `enableFloatWindow()` | ✅ Active | Konfigurasi floating window |
| `setCallingBell()` | ✅ Active | Set nada dering custom |
| `enableVirtualBackground()` | ✅ Active | Konfigurasi background virtual |
| `call()` | ❌ Deprecated | Panggilan lama (use `calls()`) |
| `groupCall()` | ❌ Deprecated | Panggilan grup lama (use `calls()`) |
| `joinInGroupCall()` | ❌ Deprecated | Join grup lama (use `join()`) |

---

**Version:** TUICallKit 3.1.1  
**Last Updated:** $(date)  
**Status:** ✅ Active Documentation 