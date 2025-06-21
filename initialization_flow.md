# Alur Inisialisasi TencentCallService

## Overview

Dokumentasi ini menjelaskan alur inisialisasi TencentCallService yang telah dioptimalkan untuk memastikan layanan panggilan tersedia segera setelah pengguna masuk ke halaman beranda aplikasi.

## Alur Inisialisasi

### 1. Aplikasi Dimulai (main.dart)
```dart
void main() async {
  // ... inisialisasi lainnya ...
  
  // TencentCallService akan diinisialisasi di TabsPage
  // Tidak ada inisialisasi di sini untuk menghindari blocking
  
  runApp(App(isDark: isDark));
}
```

### 2. Halaman Splash
- Memuat pengaturan aplikasi
- Menentukan navigasi berdasarkan status login
- Jika user sudah login → navigasi ke TabsPage
- Jika user belum login → navigasi ke LoginPage

### 3. Proses Login
```dart
// Di lib/screens/auth/login/login.page.dart
Future<void> login({lat, lon}) async {
  // ... login API call ...
  
  if (res['status'] == 200) {
    // Generate dan simpan kredensial TencentCallService
    final sdkAppID = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
    final secretKey = dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY'] ?? '';
    final userId = res["user_id"].toString();
    final userSig = GenerateTestUserSig.genTestSig(userId, sdkAppID, secretKey);
    
    // Simpan kredensial
    await setValue('userId', userId);
    await setValue('userSig', userSig);
    
    // Login ke TUICallKit
    await TUICallKit.instance.login(sdkAppID, userId, userSig);
    
    // Ambil data pengguna dan navigasi ke TabsPage
    getUserDataFunc(res["user_id"]);
  }
}
```

### 4. Halaman Beranda (TabsPage)
```dart
// Di lib/screens/tabs/tabs.page.dart
class TabsPageState extends State<TabsPage> {
  @override
  void initState() {
    super.initState();
    // ... inisialisasi lainnya ...
    _initializeTencentCallService();
  }

  /// Initialize TencentCallService when entering home page
  Future<void> _initializeTencentCallService() async {
    try {
      final userId = getStringAsync('userId');
      final userSig = getStringAsync('userSig');
      
      // Hanya inisialisasi jika kredensial tersedia
      if (userId.isNotEmpty && userSig.isNotEmpty) {
        log('Initializing TencentCallService in TabsPage...');
        final callService = TencentCallService();
        final success = await callService.initialize();
        
        if (success) {
          log('TencentCallService initialized successfully in TabsPage');
        } else {
          log('Failed to initialize TencentCallService in TabsPage');
        }
      } else {
        log('TencentCallService credentials not available in TabsPage');
      }
    } catch (e) {
      log('Error initializing TencentCallService in TabsPage: $e');
    }
  }
}
```

### 5. TencentCallService Initialize
```dart
// Di lib/services/tencent_call_service.dart
Future<bool> initialize() async {
  if (_isInitialized) {
    log('TencentCallService already initialized');
    return true;
  }

  try {
    log('Initializing TencentCallService...');

    final userId = getStringAsync('userId');
    final userSig = getStringAsync('userSig');

    if (userId.isEmpty || userSig.isEmpty) {
      log('Error: userId or userSig is empty');
      throw Exception('userId or userSig is empty');
    }

    final sdkAppId = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
    
    if (sdkAppId == 0) {
      log('Error: Invalid SDK App ID');
      throw Exception('Invalid SDK App ID');
    }

    log('Logging in to TUICallKit...');
    await TUICallKit.instance.login(sdkAppId, userId, userSig);
    log('TUICallKit login successful');

    // Set user info after login
    await _setUserInfo();

    _isInitialized = true;
    _isLoggedIn = true;
    _startKeepAliveTimer();
    log('TencentCallService initialized successfully');
    return true;
  } catch (e) {
    log('Error initializing TencentCallService: $e');
    _isInitialized = false;
    _isLoggedIn = false;
    return false;
  }
}
```

## Keuntungan Alur Baru

### 1. **Non-Blocking**
- Inisialisasi tidak memblokir startup aplikasi
- Pengguna dapat menggunakan aplikasi sementara TencentCallService diinisialisasi

### 2. **Lazy Loading**
- TencentCallService hanya diinisialisasi saat diperlukan
- Menghemat resources jika pengguna tidak menggunakan fitur panggilan

### 3. **Error Handling yang Lebih Baik**
- Logging yang detail untuk debugging
- Fallback yang graceful jika inisialisasi gagal

### 4. **User Experience yang Lebih Baik**
- Aplikasi startup lebih cepat
- Tidak ada delay saat login
- Panggilan tersedia segera setelah masuk beranda

## Troubleshooting

### 1. TencentCallService Tidak Terinisialisasi
**Gejala**: Panggilan gagal dengan error login
**Solusi**:
- Cek log untuk melihat apakah inisialisasi berhasil
- Pastikan kredensial tersimpan dengan benar
- Restart aplikasi untuk mencoba inisialisasi ulang

### 2. Kredensial Tidak Tersedia
**Gejala**: Log "TencentCallService credentials not available"
**Solusi**:
- Pastikan proses login berhasil
- Cek apakah `userId` dan `userSig` tersimpan di storage
- Coba login ulang

### 3. SDK App ID Invalid
**Gejala**: Log "Error: Invalid SDK App ID"
**Solusi**:
- Periksa file `.env` untuk `TENCENT_LIVE_SDK_APP_ID`
- Pastikan nilai tidak 0 atau kosong
- Hubungi admin untuk konfigurasi yang benar

## Best Practices

### 1. **Monitoring**
- Selalu monitor log untuk status inisialisasi
- Implementasikan retry mechanism jika diperlukan

### 2. **Error Recovery**
- Berikan feedback ke pengguna jika inisialisasi gagal
- Implementasikan fallback untuk fitur panggilan

### 3. **Performance**
- Jangan inisialisasi berulang kali
- Gunakan singleton pattern untuk TencentCallService

### 4. **Testing**
- Test alur login dan inisialisasi
- Test dengan kredensial yang valid dan tidak valid
- Test dengan koneksi internet yang lambat

## Kesimpulan

Alur inisialisasi yang baru memberikan pengalaman pengguna yang lebih baik dengan:
- Startup aplikasi yang lebih cepat
- Inisialisasi yang non-blocking
- Error handling yang lebih robust
- Logging yang detail untuk debugging

Dengan implementasi ini, TencentCallService akan tersedia segera setelah pengguna masuk ke halaman beranda, memastikan fitur panggilan dapat digunakan tanpa delay.
