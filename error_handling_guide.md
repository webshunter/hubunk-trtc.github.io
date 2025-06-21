# Panduan Penanganan Error TUICallKit Flutter

## Error -1001: Paket Basic Call Belum Dibeli

### Deskripsi Error
Error ini muncul ketika aplikasi mencoba melakukan panggilan audio/video tetapi paket basic call dari Tencent Cloud belum dibeli atau diaktifkan.

### Pesan Error
```
Error -1001: "not buy the basic call package"
```

### Penanganan Error

#### 1. Implementasi di TencentCallService

```dart
/// Get user-friendly error message based on the error
String getErrorMessage(dynamic error) {
  String errorString = error.toString().toLowerCase();
  
  if (errorString.contains('-1001') || 
      errorString.contains('not buy the basic call package') ||
      errorString.contains('basic call package')) {
    return 'Panggilan masih belum dapat dilakukan. Paket panggilan dasar belum dibeli.';
  } else if (errorString.contains('network') || errorString.contains('connection')) {
    return 'Gagal melakukan panggilan. Periksa koneksi internet Anda.';
  } else if (errorString.contains('permission') || errorString.contains('camera') || errorString.contains('microphone')) {
    return 'Gagal melakukan panggilan. Izin kamera atau mikrofon diperlukan.';
  } else if (errorString.contains('login') || errorString.contains('auth')) {
    return 'Gagal melakukan panggilan. Silakan login ulang.';
  } else {
    return 'Gagal melakukan panggilan. Silakan coba lagi.';
  }
}

/// Make a call with proper error handling
Future<bool> makeCall(String userId, bool isVideo) async {
  try {
    // Check login status first
    final isLoggedIn = await checkLoginStatus();
    if (!isLoggedIn) {
      log('Failed to ensure login status');
      toast('Gagal menginisialisasi layanan panggilan');
      return false;
    }

    log('Making ${isVideo ? 'video' : 'audio'} call to user: $userId');

    await TUICallKit.instance.call(
      userId,
      isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
    );

    return true;
  } catch (e) {
    log('Error making call: $e');
    String errorMessage = getErrorMessage(e);
    toast(errorMessage);
    return false;
  }
}
```

#### 2. Implementasi di TencentCallLauncher

```dart
// Use the new makeCall method from service
final success = await callService.makeCall(userId, isVideo);

if (success) {
  if (context.mounted) {
    Navigator.pop(context); // Tutup launcher setelah panggilan dimulai
  }
} else {
  // Error message already shown by the service
  if (context.mounted) {
    Navigator.pop(context);
  }
}
```

### Solusi untuk Error -1001

#### 1. Aktifkan Paket Basic Call di Tencent Cloud Console

1. Login ke [Tencent Cloud Console](https://console.cloud.tencent.com/)
2. Pilih produk **TRTC (Tencent Real-Time Communication)**
3. Pilih aplikasi Anda
4. Buka menu **Billing** atau **Pricing**
5. Aktifkan **Basic Call Package**
6. Pilih paket yang sesuai dengan kebutuhan Anda

#### 2. Verifikasi Konfigurasi

Pastikan konfigurasi berikut sudah benar:

```dart
// Di file .env
TENCENT_LIVE_SDK_APP_ID=your_sdk_app_id
TENCENT_LIVE_SDK_SECRET_KEY=your_secret_key

// Di TencentCallService
final sdkAppId = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
final userId = getStringAsync('userId');
final userSig = getStringAsync('userSig');

await TUICallKit.instance.login(sdkAppId, userId, userSig);
```

#### 3. Cek Status Paket

```dart
// Tambahkan method untuk cek status paket
Future<bool> checkPackageStatus() async {
  try {
    // Implementasi cek status paket dari Tencent Cloud API
    // atau dari server Anda sendiri
    return true; // atau false jika paket tidak aktif
  } catch (e) {
    log('Error checking package status: $e');
    return false;
  }
}
```

### Error Lainnya yang Umum

#### 1. Error Jaringan
- **Pesan**: "Gagal melakukan panggilan. Periksa koneksi internet Anda."
- **Solusi**: Pastikan koneksi internet stabil

#### 2. Error Izin
- **Pesan**: "Gagal melakukan panggilan. Izin kamera atau mikrofon diperlukan."
- **Solusi**: Minta izin kamera dan mikrofon di runtime

#### 3. Error Login
- **Pesan**: "Gagal melakukan panggilan. Silakan login ulang."
- **Solusi**: Re-login ke TUICallKit

### Best Practices

1. **Selalu handle error** dengan pesan yang user-friendly
2. **Log error** untuk debugging
3. **Gunakan try-catch** di semua operasi panggilan
4. **Validasi status** sebelum melakukan panggilan
5. **Beri feedback visual** kepada pengguna

### Testing Error Handling

```dart
// Test error handling
void testErrorHandling() async {
  final callService = TencentCallService();
  
  // Simulate error -1001
  try {
    throw Exception('Error -1001: not buy the basic call package');
  } catch (e) {
    String message = callService.getErrorMessage(e);
    print('Error message: $message');
    // Expected: "Panggilan masih belum dapat dilakukan. Paket panggilan dasar belum dibeli."
  }
}
```

### Referensi

- [Tencent Cloud TRTC Documentation](https://trtc.io/document/59851?platform=flutter&product=call&menulabel=uikit)
- [TUICallKit API Reference](https://trtc.io/document/54906?platform=flutter&product=call&menulabel=uikit)
- [Tencent Cloud Console](https://console.cloud.tencent.com/) 