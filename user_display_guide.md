# Panduan Menampilkan Nama Pengguna pada Panggilan

## Overview

Fitur menampilkan nama pengguna pada panggilan memungkinkan aplikasi untuk menampilkan nama lengkap atau username pengguna yang sedang melakukan panggilan, sehingga memberikan pengalaman pengguna yang lebih personal dan informatif.

## Fitur Utama

### 1. Menampilkan Nama pada Launcher
- Menampilkan nama pengguna yang sedang dipanggil
- Loading screen yang informatif
- Indikator jenis panggilan (audio/video)

### 2. Informasi Pengguna di TUICallKit
- Mengatur nama dan avatar pengguna
- Otomatis terapkan saat login
- Mendukung fallback ke username jika nama lengkap tidak tersedia

### 3. UI yang User-Friendly
- Loading screen dengan nama pengguna
- Icon yang sesuai dengan jenis panggilan
- Pesan status yang jelas

## Implementasi

### 1. Mengatur Informasi Pengguna

```dart
class TencentCallService {
  /// Set user information in TUICallKit
  Future<void> _setUserInfo() async {
    try {
      final currentUser = getCurrentUserData();
      if (currentUser != null) {
        final displayName = getUserDisplayName(currentUser);
        final avatarUrl = getUserAvatarUrl(currentUser);
        
        await TUICallKit.instance.setSelfInfo(displayName, avatarUrl);
        print('User info set successfully in TUICallKit');
      }
    } catch (e) {
      print('Error setting user info in TUICallKit: $e');
    }
  }

  /// Get user display name with fallback logic
  String getUserDisplayName(Usr? user) {
    if (user == null) return 'Unknown User';
    
    // Prioritas 1: Nama lengkap (firstName + lastName)
    if (user.firstName != null && 
        user.firstName!.isNotEmpty && 
        user.lastName != null && 
        user.lastName!.isNotEmpty) {
      return '${user.firstName} ${user.lastName}';
    } 
    // Prioritas 2: Username
    else if (user.username != null && user.username!.isNotEmpty) {
      return user.username!;
    } 
    // Fallback: Unknown User
    else {
      return 'Unknown User';
    }
  }

  /// Get user avatar URL
  String getUserAvatarUrl(Usr? user) {
    if (user?.avatar != null && user!.avatar!.isNotEmpty) {
      return user.avatar!;
    }
    return ''; // Return empty string if no avatar
  }
}
```

### 2. Launcher dengan Nama Pengguna

```dart
class TencentCallLauncher extends StatelessWidget {
  final String userId;
  final bool isVideo;
  final String? userName; // Nama pengguna yang dipanggil

  const TencentCallLauncher({
    Key? key,
    required this.userId,
    required this.isVideo,
    this.userName,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black87,
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Icon panggilan
            Icon(
              isVideo ? Icons.videocam : Icons.call,
              size: 80,
              color: Colors.white,
            ),
            const SizedBox(height: 30),
            
            // Loading indicator
            const CircularProgressIndicator(
              valueColor: AlwaysStoppedAnimation<Color>(Colors.white),
            ),
            const SizedBox(height: 30),
            
            // Nama pengguna
            if (userName != null && userName!.isNotEmpty)
              Text(
                'Memanggil $userName...',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.w500,
                ),
                textAlign: TextAlign.center,
              )
            else
              Text(
                'Memulai panggilan...',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.w500,
                ),
                textAlign: TextAlign.center,
              ),
            
            const SizedBox(height: 10),
            
            // Jenis panggilan
            Text(
              isVideo ? 'Panggilan Video' : 'Panggilan Suara',
              style: const TextStyle(
                color: Colors.white70,
                fontSize: 14,
              ),
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }
}
```

### 3. Menggunakan dari Halaman Chat

```dart
// Dari message_details_page.dart
IconButton(
  icon: const Icon(Icons.call),
  tooltip: 'Voice Call',
  onPressed: () async {
    await Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => TencentCallLauncher(
          userId: widget.chat.userId,
          isVideo: false,
          userName: "${widget.chat.firstName} ${widget.chat.lastName}",
        ),
      ),
    );
  },
),

IconButton(
  icon: const Icon(Icons.videocam),
  tooltip: 'Video Call',
  onPressed: () async {
    await Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => TencentCallLauncher(
          userId: widget.chat.userId,
          isVideo: true,
          userName: "${widget.chat.firstName} ${widget.chat.lastName}",
        ),
      ),
    );
  },
),
```

## Struktur Data Pengguna

### Model Usr
```dart
class Usr {
  String? id;
  String? username;
  String? firstName;
  String? lastName;
  String? avatar;
  // ... properti lainnya
}
```

### Prioritas Nama
1. **Nama Lengkap**: `firstName + lastName` (jika keduanya tersedia)
2. **Username**: `username` (jika nama lengkap tidak tersedia)
3. **Fallback**: "Unknown User" (jika tidak ada data)

## Best Practices

### 1. Error Handling
```dart
try {
  await TUICallKit.instance.setSelfInfo(displayName, avatarUrl);
} catch (e) {
  print('Error setting user info: $e');
  // Fallback ke default atau retry
}
```

### 2. Validasi Data
```dart
String getUserDisplayName(Usr? user) {
  if (user == null) return 'Unknown User';
  
  // Validasi nama lengkap
  if (user.firstName?.isNotEmpty == true && 
      user.lastName?.isNotEmpty == true) {
    return '${user.firstName} ${user.lastName}';
  }
  
  // Validasi username
  if (user.username?.isNotEmpty == true) {
    return user.username!;
  }
  
  return 'Unknown User';
}
```

## Troubleshooting

### 1. Nama Tidak Muncul
- Cek apakah data pengguna tersimpan dengan benar
- Validasi format JSON data pengguna
- Pastikan `setSelfInfo` dipanggil setelah login

### 2. Avatar Tidak Muncul
- Validasi URL avatar
- Cek koneksi internet
- Pastikan URL dapat diakses

### 3. Error saat Set User Info
- Cek status login TUICallKit
- Validasi parameter yang dikirim
- Handle exception dengan graceful

## Kesimpulan

Fitur menampilkan nama pengguna pada panggilan memberikan pengalaman pengguna yang lebih personal dan informatif. Implementasi yang baik meliputi:

1. **Data Management**: Pengelolaan data pengguna yang efisien
2. **UI/UX**: Interface yang user-friendly dan informatif
3. **Error Handling**: Penanganan error yang graceful
4. **Performance**: Optimasi performa dan caching
5. **Maintainability**: Kode yang mudah dipelihara dan diperluas

Dengan implementasi yang tepat, fitur ini akan meningkatkan kualitas pengalaman pengguna dalam aplikasi panggilan Anda.

## Konfigurasi

### 1. Inisialisasi Otomatis di Halaman Beranda
TencentCallService akan diinisialisasi secara otomatis saat pengguna masuk ke halaman beranda (TabsPage):

```dart
// Di lib/screens/tabs/tabs.page.dart
@override
void initState() {
  super.initState();
  // ... kode lainnya
  _initializeTencentCallService();
}

/// Initialize TencentCallService when entering home page
Future<void> _initializeTencentCallService() async {
  try {
    final userId = getStringAsync('userId');
    final userSig = getStringAsync('userSig');
    
    // Only initialize if we have the required credentials
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
```

### 2. Penyimpanan Kredensial saat Login
Kredensial TencentCallService disimpan saat proses login:

```dart
// Di lib/screens/auth/login/login.page.dart
Future<void> login({lat, lon}) async {
  // ... login logic ...
  
  if (res['status'] == 200) {
    // Generate and store TUICallKit credentials
    final sdkAppID = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
    final secretKey = dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY'] ?? '';
    final userId = res["user_id"].toString();
    final userSig = GenerateTestUserSig.genTestSig(userId, sdkAppID, secretKey);
    
    // Store credentials for TUICallKit
    await setValue('userId', userId);
    await setValue('userSig', userSig);
    
    // Login to TUICallKit
    await TUICallKit.instance.login(sdkAppID, userId, userSig);
    
    // ... rest of login logic ...
  }
}
```

### 3. Update Manual
```dart
// Update user info ketika data berubah
Future<void> updateUserInfo() async {
  await _setUserInfo();
}
```
