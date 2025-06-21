# TUICallKit Flutter - Troubleshooting Guide

## 🚨 Error Codes & Solutions

### Error -1001: "not buy the basic call package"

**Penyebab:**
- Paket TRTC belum diaktifkan di Tencent Cloud Console
- SDKAppID tidak terdaftar untuk layanan panggilan
- Akun menunggak pembayaran
- Region/server tidak sesuai

**Solusi:**
1. **Cek Status Paket di Console:**
   ```
   1. Login ke https://console.tencentcloud.com/trtc
   2. Pilih aplikasi Anda
   3. Cek status "Package Status"
   4. Pastikan status "Active" atau "Trial"
   ```

2. **Verifikasi SDKAppID:**
   ```dart
   // Debug helper
   void debugSDKAppID() {
     final sdkAppId = dotenv.env['TENCENT_LIVE_SDK_APP_ID'];
     print('SDK App ID: $sdkAppId');
     print('Type: ${sdkAppId.runtimeType}');
   }
   ```

3. **Cek Pembayaran Akun:**
   - Pastikan tidak ada tagihan yang menunggak
   - Verifikasi metode pembayaran aktif

4. **Hubungi Support Tencent:**
   - Jika semua sudah benar, hubungi info_rtc@tencent.com
   - Sertakan SDKAppID dan log error

### Error -1: "not login"

**Penyebab:**
- Belum login ke TUICallKit sebelum melakukan panggilan
- UserSig expired atau tidak valid
- Login gagal karena kredensial salah

**Solusi:**
1. **Pastikan Login Sequence:**
   ```dart
   // Urutan yang benar
   await TUICallKit.instance.login(sdkAppId, userId, userSig);
   // Tunggu login selesai
   await TUICallKit.instance.calls(userIds, mediaType);
   ```

2. **Cek UserSig:**
   ```dart
   void debugUserSig() {
     final userSig = getStringAsync('userSig');
     print('UserSig length: ${userSig.length}');
     print('UserSig: $userSig');
     
     // UserSig harus > 100 karakter
     if (userSig.length < 100) {
       print('WARNING: UserSig terlalu pendek!');
     }
   }
   ```

3. **Regenerate UserSig:**
   ```dart
   // Generate ulang UserSig
   final userSig = GenerateTestUserSig.genTestSig(userId, sdkAppId, secretKey);
   await setValue('userSig', userSig);
   ```

### Error -1002: "invalid parameter"

**Penyebab:**
- Parameter yang dikirim ke API tidak valid
- userId kosong atau null
- mediaType tidak sesuai

**Solusi:**
1. **Validasi Parameter:**
   ```dart
   Future<bool> validateCallParameters(String userId, TUICallMediaType mediaType) {
     if (userId.isEmpty) {
       print('ERROR: userId is empty');
       return false;
     }
     
     if (mediaType != TUICallMediaType.audio && 
         mediaType != TUICallMediaType.video) {
       print('ERROR: Invalid mediaType');
       return false;
     }
     
     return true;
   }
   ```

2. **Debug Call Parameters:**
   ```dart
   void debugCallParameters(List<String> userIds, TUICallMediaType mediaType) {
     print('=== CALL PARAMETERS DEBUG ===');
     print('UserIDs: $userIds');
     print('UserIDs count: ${userIds.length}');
     print('MediaType: $mediaType');
     
     for (int i = 0; i < userIds.length; i++) {
       print('UserID[$i]: "${userIds[i]}" (length: ${userIds[i].length})');
     }
     print('=============================');
   }
   ```

### Error -1003: "network error"

**Penyebab:**
- Koneksi internet tidak stabil
- Firewall memblokir koneksi
- DNS resolution gagal

**Solusi:**
1. **Cek Koneksi Internet:**
   ```dart
   import 'package:connectivity_plus/connectivity_plus.dart';
   
   Future<bool> checkInternetConnection() async {
     var connectivityResult = await Connectivity().checkConnectivity();
     return connectivityResult != ConnectivityResult.none;
   }
   ```

2. **Implement Retry Logic:**
   ```dart
   Future<bool> makeCallWithRetry({
     required List<String> userIds,
     required TUICallMediaType mediaType,
     int maxRetries = 3,
   }) async {
     for (int i = 0; i < maxRetries; i++) {
       try {
         await TUICallKit.instance.calls(userIds, mediaType);
         return true;
       } catch (e) {
         print('Call attempt ${i + 1} failed: $e');
         if (i < maxRetries - 1) {
           await Future.delayed(Duration(seconds: 2 * (i + 1)));
         }
       }
     }
     return false;
   }
   ```

## 🔧 Common Issues & Fixes

### Issue 1: Call UI tidak muncul

**Gejala:** Panggilan berhasil dimulai tapi UI tidak muncul

**Penyebab:**
- `navigatorObserver` tidak ditambahkan ke MaterialApp
- Widget tree tidak di-render dengan benar

**Solusi:**
```dart
// Pastikan di main.dart
MaterialApp(
  navigatorObservers: [TUICallKit.navigatorObserver], // ← Ini wajib!
  // ...
)
```

### Issue 2: Kamera/Mikrofon tidak berfungsi

**Gejala:** Panggilan video/audio tidak ada suara/gambar

**Penyebab:**
- Permission tidak diberikan
- Device tidak mendukung
- Hardware error

**Solusi:**
1. **Cek Permission di Android:**
   ```xml
   <!-- android/app/src/main/AndroidManifest.xml -->
   <uses-permission android:name="android.permission.CAMERA" />
   <uses-permission android:name="android.permission.RECORD_AUDIO" />
   <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
   ```

2. **Request Permission Runtime:**
   ```dart
   import 'package:permission_handler/permission_handler.dart';
   
   Future<bool> requestCallPermissions() async {
     Map<Permission, PermissionStatus> statuses = await [
       Permission.camera,
       Permission.microphone,
     ].request();
     
     return statuses[Permission.camera]!.isGranted &&
            statuses[Permission.microphone]!.isGranted;
   }
   ```

### Issue 3: Call tidak bisa di-answer

**Gejala:** Panggilan masuk tapi tidak bisa dijawab

**Penyebab:**
- Observer tidak terpasang dengan benar
- UI callback tidak terhandle

**Solusi:**
```dart
// Pastikan observer terpasang
TUICallObserver observer = TUICallObserver(
  onCallReceived: (String callerId, List<String> calleeIdList, 
                   String groupId, TUICallMediaType callMediaType) {
    print('Incoming call from: $callerId');
    // Handle incoming call UI
  },
  // ... other callbacks
);

TUICallEngine.instance.addObserver(observer);
```

### Issue 4: Call drop setelah beberapa detik

**Gejala:** Panggilan terputus otomatis

**Penyebab:**
- Keep-alive timer tidak berfungsi
- Network timeout
- Service di-kill oleh OS

**Solusi:**
```dart
// Implement proper keep-alive
class TencentCallService {
  Timer? _keepAliveTimer;
  
  void _startKeepAliveTimer() {
    _keepAliveTimer?.cancel();
    _keepAliveTimer = Timer.periodic(const Duration(minutes: 5), (timer) {
      _pingServer();
    });
  }
  
  Future<void> _pingServer() async {
    try {
      // Simple ping to keep connection alive
      await TUICallKit.instance.setSelfInfo('', '');
    } catch (e) {
      print('Keep-alive ping failed: $e');
    }
  }
}
```

## 🐛 Debug Tools

### Debug Helper Class

```dart
// lib/utils/call_debug_helper.dart
class CallDebugHelper {
  static void printFullDebugInfo() {
    print('=== TUICALLKIT DEBUG INFO ===');
    
    // Environment variables
    final sdkAppId = dotenv.env['TENCENT_LIVE_SDK_APP_ID'];
    final secretKey = dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY'];
    print('SDK App ID: $sdkAppId');
    print('Secret Key length: ${secretKey?.length ?? 0}');
    
    // Stored values
    final userId = getStringAsync('userId');
    final userSig = getStringAsync('userSig');
    print('Stored User ID: $userId');
    print('Stored UserSig length: ${userSig.length}');
    
    // Service status
    final callService = TencentCallService();
    print('Service Initialized: ${callService.isInitialized}');
    print('Service Logged In: ${callService.isLoggedIn}');
    
    // Device info
    print('Platform: ${Platform.operatingSystem}');
    print('Platform Version: ${Platform.operatingSystemVersion}');
    
    print('==============================');
  }
  
  static void validateConfiguration() {
    final errors = <String>[];
    
    // Check environment variables
    if (dotenv.env['TENCENT_LIVE_SDK_APP_ID']?.isEmpty ?? true) {
      errors.add('TENCENT_LIVE_SDK_APP_ID is empty');
    }
    
    if (dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY']?.isEmpty ?? true) {
      errors.add('TENCENT_LIVE_SDK_SECRET_KEY is empty');
    }
    
    // Check stored values
    if (getStringAsync('userId').isEmpty) {
      errors.add('userId is not stored');
    }
    
    if (getStringAsync('userSig').isEmpty) {
      errors.add('userSig is not stored');
    }
    
    if (errors.isNotEmpty) {
      print('❌ CONFIGURATION ERRORS:');
      for (final error in errors) {
        print('  - $error');
      }
    } else {
      print('✅ Configuration looks good!');
    }
  }
}
```

### Log Analysis Script

```bash
#!/bin/bash
# debug_logs.sh - Analyze Flutter logs for TUICallKit issues

echo "=== TUICallKit Log Analysis ==="

# Check for common error patterns
echo "Checking for error patterns..."

if grep -q "not buy the basic call package" logs.txt; then
    echo "❌ Found: 'not buy the basic call package'"
    echo "   Solution: Check Tencent Cloud Console package status"
fi

if grep -q "not login" logs.txt; then
    echo "❌ Found: 'not login'"
    echo "   Solution: Ensure login is called before making calls"
fi

if grep -q "invalid parameter" logs.txt; then
    echo "❌ Found: 'invalid parameter'"
    echo "   Solution: Validate call parameters"
fi

if grep -q "network error" logs.txt; then
    echo "❌ Found: 'network error'"
    echo "   Solution: Check internet connection and firewall"
fi

# Check for successful patterns
if grep -q "Login success" logs.txt; then
    echo "✅ Found: Successful login"
fi

if grep -q "Call started" logs.txt; then
    echo "✅ Found: Successful call initiation"
fi

echo "=== Analysis Complete ==="
```

## 📞 Support Resources

### Official Documentation
- [TUICallKit Flutter Documentation](https://trtc.io/document/54906?platform=flutter&product=call&menulabel=uikit)
- [Tencent Cloud Console](https://console.tencentcloud.com/trtc)
- [API Reference](https://trtc.io/document/35166?platform=flutter&product=call&menulabel=uikit)

### Community Resources
- [GitHub Issues](https://github.com/Tencent-RTC/tls-sig-api-v2-flutter/issues)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/tencent-rtc)
- [Tencent Developer Forum](https://cloud.tencent.com/developer/forum)

### Contact Support
- **Email:** info_rtc@tencent.com
- **Response Time:** 24-48 hours
- **Required Info:** SDKAppID, Error Code, Log Files

## 🔄 Update Checklist

Setelah melakukan troubleshooting, pastikan:

- [ ] Environment variables sudah benar
- [ ] Permission sudah diberikan
- [ ] NavigatorObserver sudah ditambahkan
- [ ] Login sequence sudah benar
- [ ] Error handling sudah implement
- [ ] Test di device fisik
- [ ] Log sudah dianalisis
- [ ] Support sudah dihubungi (jika perlu)

---

**Last Updated:** $(date)  
**Version:** TUICallKit 3.1.1  
**Status:** ✅ Active Maintenance 