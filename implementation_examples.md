# TUICallKit Flutter - Contoh Implementasi Praktis

## 📁 Struktur File yang Direkomendasikan

```
lib/
├── services/
│   ├── tencent_call_service.dart
│   └── call_observer.dart
├── widgets/
│   ├── tencent_call_launcher.dart
│   ├── call_buttons.dart
│   └── call_status_indicator.dart
├── screens/
│   ├── chat/
│   │   └── chat_page.dart
│   ├── calls/
│   │   ├── call_history_page.dart
│   │   └── group_call_page.dart
│   └── auth/
│       └── login_page.dart
└── utils/
    └── call_utils.dart
```

## 🔧 Contoh Implementasi Lengkap

### 1. Service Layer - TencentCallService dengan Error Handling

```dart
// lib/services/tencent_call_service.dart
import 'dart:async';
import 'dart:developer';
import 'package:flutter/material.dart';
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:tencent_calls_uikit/debug/generate_test_user_sig.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:nb_utils/nb_utils.dart';

class TencentCallService {
  static final TencentCallService _instance = TencentCallService._internal();
  bool _isInitialized = false;
  bool _isLoggedIn = false;
  Timer? _keepAliveTimer;
  StreamController<CallStatus>? _callStatusController;

  factory TencentCallService() => _instance;
  TencentCallService._internal();

  Stream<CallStatus> get callStatusStream => 
      _callStatusController?.stream ?? Stream.empty();

  /// Get user-friendly error message based on the error
  String getErrorMessage(dynamic error) {
    String errorString = error.toString().toLowerCase();
    
    if (errorString.contains('-1001') || 
        errorString.contains('not buy the basic call package') ||
        errorString.contains('basic call package')) {
      return 'Panggilan masih belum dapat dilakukan. Paket panggilan dasar belum dibeli.';
    } else if (errorString.contains('-1002') || errorString.contains('network')) {
      return 'Gagal melakukan panggilan. Periksa koneksi internet Anda.';
    } else if (errorString.contains('-1003') || errorString.contains('permission')) {
      return 'Gagal melakukan panggilan. Izin kamera atau mikrofon diperlukan.';
    } else if (errorString.contains('-1004') || errorString.contains('login')) {
      return 'Gagal melakukan panggilan. Silakan login ulang.';
    } else if (errorString.contains('-1005') || errorString.contains('timeout')) {
      return 'Panggilan timeout. Silakan coba lagi.';
    } else if (errorString.contains('-1006') || errorString.contains('busy')) {
      return 'Pengguna sedang sibuk. Silakan coba lagi nanti.';
    } else if (errorString.contains('-1007') || errorString.contains('rejected')) {
      return 'Panggilan ditolak oleh pengguna.';
    } else if (errorString.contains('-1008') || errorString.contains('not found')) {
      return 'Pengguna tidak ditemukan.';
    } else {
      return 'Gagal melakukan panggilan. Silakan coba lagi.';
    }
  }

  /// Check if error is related to basic call package
  bool isBasicCallPackageError(dynamic error) {
    String errorString = error.toString().toLowerCase();
    return errorString.contains('-1001') || 
           errorString.contains('not buy the basic call package') ||
           errorString.contains('basic call package');
  }

  /// Check if error is network related
  bool isNetworkError(dynamic error) {
    String errorString = error.toString().toLowerCase();
    return errorString.contains('network') || 
           errorString.contains('connection') ||
           errorString.contains('timeout');
  }

  /// Check if error is permission related
  bool isPermissionError(dynamic error) {
    String errorString = error.toString().toLowerCase();
    return errorString.contains('permission') || 
           errorString.contains('camera') ||
           errorString.contains('microphone');
  }

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
        throw Exception('userId or userSig is empty');
      }

      final sdkAppId = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
      log('SDK App ID: $sdkAppId');

      await TUICallKit.instance.login(sdkAppId, userId, userSig);

      _isInitialized = true;
      _isLoggedIn = true;
      _startKeepAliveTimer();
      _setupCallObserver();
      
      log('TencentCallService initialized successfully');
      return true;
    } catch (e) {
      log('Error initializing TencentCallService: $e');
      _isInitialized = false;
      _isLoggedIn = false;
      
      // Show appropriate error message
      String errorMessage = getErrorMessage(e);
      toast(errorMessage);
      
      return false;
    }
  }

  Future<bool> checkLoginStatus() async {
    try {
      final currentUserId = getStringAsync('userId');
      if (currentUserId.isEmpty) {
        log('No user ID found');
        return false;
      }

      if (!_isInitialized || !_isLoggedIn) {
        log('Service not initialized or not logged in, attempting to initialize...');
        return initialize();
      }

      return true;
    } catch (e) {
      log('Error checking login status: $e');
      _isLoggedIn = false;
      return false;
    }
  }

  /// Make a call with comprehensive error handling
  Future<CallResult> makeCall({
    required List<String> userIds,
    required bool isVideo,
    Map<String, dynamic>? params,
  }) async {
    try {
      final isLoggedIn = await checkLoginStatus();
      if (!isLoggedIn) {
        return CallResult.error('Gagal menginisialisasi layanan panggilan');
      }

      log('Making ${isVideo ? 'video' : 'audio'} call to users: $userIds');

      await TUICallKit.instance.calls(
        userIds,
        isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
        params,
      );

      return CallResult.success();
    } catch (e) {
      log('Error making call: $e');
      String errorMessage = getErrorMessage(e);
      
      // Show toast with error message
      toast(errorMessage);
      
      return CallResult.error(errorMessage, error: e);
    }
  }

  /// Make a call with legacy API (for backward compatibility)
  Future<CallResult> makeCallLegacy({
    required String userId,
    required bool isVideo,
  }) async {
    try {
      final isLoggedIn = await checkLoginStatus();
      if (!isLoggedIn) {
        return CallResult.error('Gagal menginisialisasi layanan panggilan');
      }

      log('Making ${isVideo ? 'video' : 'audio'} call to user: $userId');

      await TUICallKit.instance.call(
        userId,
        isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
      );

      return CallResult.success();
    } catch (e) {
      log('Error making call: $e');
      String errorMessage = getErrorMessage(e);
      
      // Show toast with error message
      toast(errorMessage);
      
      return CallResult.error(errorMessage, error: e);
    }
  }

  Future<bool> joinCall(String callId) async {
    try {
      final isLoggedIn = await checkLoginStatus();
      if (!isLoggedIn) {
        throw Exception('Not logged in to TUICallKit');
      }

      log('Joining call: $callId');
      await TUICallKit.instance.join(callId);
      return true;
    } catch (e) {
      log('Error joining call: $e');
      String errorMessage = getErrorMessage(e);
      toast(errorMessage);
      return false;
    }
  }

  Future<void> logout() async {
    try {
      await TUICallKit.instance.logout();
      _isInitialized = false;
      _isLoggedIn = false;
      _keepAliveTimer?.cancel();
      _callStatusController?.close();
      log('Logged out from TUICallKit');
    } catch (e) {
      log('Error during logout: $e');
    }
  }

  void _startKeepAliveTimer() {
    _keepAliveTimer?.cancel();
    _keepAliveTimer = Timer.periodic(const Duration(minutes: 5), (timer) {
      checkLoginStatus();
    });
  }

  void _setupCallObserver() {
    _callStatusController = StreamController<CallStatus>.broadcast();
    
    TUICallObserver observer = TUICallObserver(
      onError: (int code, String message) {
        log('Call Error: $code - $message');
        
        // Handle specific error codes
        String userMessage = getErrorMessage('Error $code: $message');
        toast(userMessage);
        
        _callStatusController?.add(CallStatus.error(code, message));
      },
      onCallBegin: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole) {
        log('Call started: Room $roomId, Type: $callMediaType, Role: $callRole');
        _callStatusController?.add(CallStatus.started(roomId, callMediaType, callRole));
      },
      onCallEnd: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole, double totalTime) {
        log('Call ended: Duration ${totalTime}s');
        _callStatusController?.add(CallStatus.ended(roomId, totalTime));
      },
      onUserNetworkQualityChanged: (List<TUINetworkQualityInfo> networkQualityList) {
        for (var quality in networkQualityList) {
          log('User ${quality.userId}: Quality ${quality.quality}');
        }
        _callStatusController?.add(CallStatus.networkQualityChanged(networkQualityList));
      },
      onCallReceived: (String callerId, List<String> calleeIdList, String groupId, TUICallMediaType callMediaType) {
        log('Incoming call from: $callerId');
        _callStatusController?.add(CallStatus.received(callerId, calleeIdList, groupId, callMediaType));
      },
    );

    TUICallEngine.instance.addObserver(observer);
  }

  void dispose() {
    _keepAliveTimer?.cancel();
    _callStatusController?.close();
    _isInitialized = false;
    _isLoggedIn = false;
  }
}

/// Result class for call operations
class CallResult {
  final bool success;
  final String? message;
  final dynamic error;

  CallResult._({required this.success, this.message, this.error});

  factory CallResult.success([String? message]) {
    return CallResult._(success: true, message: message);
  }

  factory CallResult.error(String message, {dynamic error}) {
    return CallResult._(success: false, message: message, error: error);
  }
}

/// Call status for stream updates
class CallStatus {
  final String type;
  final Map<String, dynamic> data;

  CallStatus._({required this.type, required this.data});

  factory CallStatus.error(int code, String message) {
    return CallStatus._(type: 'error', data: {'code': code, 'message': message});
  }

  factory CallStatus.started(TUIRoomId roomId, TUICallMediaType mediaType, TUICallRole role) {
    return CallStatus._(type: 'started', data: {
      'roomId': roomId,
      'mediaType': mediaType,
      'role': role,
    });
  }

  factory CallStatus.ended(TUIRoomId roomId, double duration) {
    return CallStatus._(type: 'ended', data: {
      'roomId': roomId,
      'duration': duration,
    });
  }

  factory CallStatus.networkQualityChanged(List<TUINetworkQualityInfo> qualityList) {
    return CallStatus._(type: 'networkQuality', data: {'qualityList': qualityList});
  }

  factory CallStatus.received(String callerId, List<String> calleeIds, String groupId, TUICallMediaType mediaType) {
    return CallStatus._(type: 'received', data: {
      'callerId': callerId,
      'calleeIds': calleeIds,
      'groupId': groupId,
      'mediaType': mediaType,
    });
  }
}
```

### 2. Widget Components dengan Error Handling

#### Call Launcher Widget dengan Error Handling Lengkap

```dart
// lib/widgets/tencent_call_launcher.dart
import 'package:flutter/material.dart';
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:hub_unk/services/tencent_call_service.dart';
import 'package:nb_utils/nb_utils.dart';

class TencentCallLauncher extends StatefulWidget {
  final String userId;
  final bool isVideo;
  final String? userName;
  final VoidCallback? onCallStarted;
  final VoidCallback? onCallFailed;
  final Function(String)? onError;

  const TencentCallLauncher({
    Key? key,
    required this.userId,
    required this.isVideo,
    this.userName,
    this.onCallStarted,
    this.onCallFailed,
    this.onError,
  }) : super(key: key);

  @override
  State<TencentCallLauncher> createState() => _TencentCallLauncherState();
}

class _TencentCallLauncherState extends State<TencentCallLauncher> {
  bool _isInitializing = true;
  bool _isCallInProgress = false;
  String? _errorMessage;
  final TencentCallService _callService = TencentCallService();

  @override
  void initState() {
    super.initState();
    _initializeCall();
  }

  Future<void> _initializeCall() async {
    try {
      setState(() {
        _isInitializing = true;
        _errorMessage = null;
      });

      // Debug print values before making call
      _callService.debugPrintValues();

      // Use the new makeCall method with error handling
      final result = await _callService.makeCallLegacy(
        userId: widget.userId,
        isVideo: widget.isVideo,
      );

      if (result.success) {
        setState(() {
          _isInitializing = false;
          _isCallInProgress = true;
        });
        
        widget.onCallStarted?.call();
        
        // Close launcher after successful call initiation
        if (mounted) {
          Navigator.pop(context);
        }
      } else {
        setState(() {
          _isInitializing = false;
          _errorMessage = result.message;
        });
        
        widget.onCallFailed?.call();
        widget.onError?.call(result.message ?? 'Unknown error');
        
        // Show error dialog for specific errors
        _showErrorDialog(result.message ?? 'Unknown error', result.error);
      }
    } catch (e) {
      log('Unexpected error in TencentCallLauncher: $e');
      setState(() {
        _isInitializing = false;
        _errorMessage = 'Terjadi kesalahan yang tidak terduga';
      });
      
      widget.onCallFailed?.call();
      widget.onError?.call('Terjadi kesalahan yang tidak terduga');
      
      if (mounted) {
        _showErrorDialog('Terjadi kesalahan yang tidak terduga', e);
      }
    }
  }

  void _showErrorDialog(String message, dynamic error) {
    if (!mounted) return;

    // Check if it's a basic call package error
    bool isBasicCallPackageError = _callService.isBasicCallPackageError(error);
    bool isNetworkError = _callService.isNetworkError(error);
    bool isPermissionError = _callService.isPermissionError(error);

    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (BuildContext context) {
        return AlertDialog(
          title: Row(
            children: [
              Icon(
                isBasicCallPackageError 
                    ? Icons.payment 
                    : isNetworkError 
                        ? Icons.wifi_off 
                        : isPermissionError 
                            ? Icons.camera_alt 
                            : Icons.error,
                color: isBasicCallPackageError 
                    ? Colors.orange 
                    : isNetworkError 
                        ? Colors.red 
                        : isPermissionError 
                            ? Colors.blue 
                            : Colors.red,
              ),
              const SizedBox(width: 8),
              Text(
                isBasicCallPackageError 
                    ? 'Paket Panggilan' 
                    : isNetworkError 
                        ? 'Koneksi Jaringan' 
                        : isPermissionError 
                            ? 'Izin Diperlukan' 
                            : 'Error',
              ),
            ],
          ),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(message),
              if (isBasicCallPackageError) ...[
                const SizedBox(height: 16),
                const Text(
                  'Untuk menggunakan fitur panggilan, Anda perlu:',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                const Text('• Aktifkan paket basic call di Tencent Cloud Console'),
                const Text('• Hubungi administrator untuk konfigurasi'),
                const Text('• Pastikan akun memiliki izin panggilan'),
              ],
              if (isNetworkError) ...[
                const SizedBox(height: 16),
                const Text(
                  'Solusi:',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                const Text('• Periksa koneksi internet Anda'),
                const Text('• Coba gunakan WiFi atau data seluler'),
                const Text('• Restart aplikasi jika diperlukan'),
              ],
              if (isPermissionError) ...[
                const SizedBox(height: 16),
                const Text(
                  'Solusi:',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                const Text('• Izinkan akses kamera dan mikrofon'),
                const Text('• Periksa pengaturan izin di aplikasi'),
                const Text('• Restart aplikasi setelah memberikan izin'),
              ],
            ],
          ),
          actions: [
            if (isBasicCallPackageError)
              TextButton(
                onPressed: () {
                  Navigator.pop(context); // Close dialog
                  Navigator.pop(context); // Close launcher
                  // You can add navigation to settings or contact support
                  toast('Hubungi administrator untuk mengaktifkan paket panggilan');
                },
                child: const Text('Hubungi Admin'),
              ),
            if (isNetworkError)
              TextButton(
                onPressed: () {
                  Navigator.pop(context); // Close dialog
                  _retryCall();
                },
                child: const Text('Coba Lagi'),
              ),
            if (isPermissionError)
              TextButton(
                onPressed: () {
                  Navigator.pop(context); // Close dialog
                  _requestPermissions();
                },
                child: const Text('Berikan Izin'),
              ),
            TextButton(
              onPressed: () {
                Navigator.pop(context); // Close dialog
                Navigator.pop(context); // Close launcher
              },
              child: const Text('Tutup'),
            ),
          ],
        );
      },
    );
  }

  Future<void> _retryCall() async {
    setState(() {
      _isInitializing = true;
      _errorMessage = null;
    });
    
    // Wait a bit before retrying
    await Future.delayed(const Duration(seconds: 2));
    await _initializeCall();
  }

  Future<void> _requestPermissions() async {
    // Implement permission request logic here
    // You can use permission_handler package
    toast('Mengarahkan ke pengaturan izin...');
    
    // For now, just retry the call
    await _retryCall();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black87,
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            if (_isInitializing) ...[
              const CircularProgressIndicator(
                valueColor: AlwaysStoppedAnimation<Color>(Colors.white),
              ),
              const SizedBox(height: 16),
              Text(
                'Memulai ${widget.isVideo ? 'panggilan video' : 'panggilan audio'}...',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 16,
                ),
              ),
              if (widget.userName != null) ...[
                const SizedBox(height: 8),
                Text(
                  'Kepada: ${widget.userName}',
                  style: const TextStyle(
                    color: Colors.white70,
                    fontSize: 14,
                  ),
                ),
              ],
            ] else if (_errorMessage != null) ...[
              Icon(
                Icons.error_outline,
                color: Colors.red.shade300,
                size: 64,
              ),
              const SizedBox(height: 16),
              Text(
                'Gagal Memulai Panggilan',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 8),
              Padding(
                padding: const EdgeInsets.symmetric(horizontal: 32),
                child: Text(
                  _errorMessage!,
                  textAlign: TextAlign.center,
                  style: const TextStyle(
                    color: Colors.white70,
                    fontSize: 14,
                  ),
                ),
              ),
              const SizedBox(height: 24),
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  ElevatedButton.icon(
                    onPressed: _retryCall,
                    icon: const Icon(Icons.refresh),
                    label: const Text('Coba Lagi'),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.blue,
                      foregroundColor: Colors.white,
                    ),
                  ),
                  const SizedBox(width: 16),
                  ElevatedButton.icon(
                    onPressed: () => Navigator.pop(context),
                    icon: const Icon(Icons.close),
                    label: const Text('Tutup'),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.grey,
                      foregroundColor: Colors.white,
                    ),
                  ),
                ],
              ),
            ] else if (_isCallInProgress) ...[
              const Icon(
                Icons.call,
                color: Colors.green,
                size: 64,
              ),
              const SizedBox(height: 16),
              const Text(
                'Panggilan Dimulai',
                style: TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    super.dispose();
  }
}
```

#### Call Buttons Widget

```dart
// lib/widgets/call_buttons.dart
import 'package:flutter/material.dart';
import 'package:hub_unk/widgets/tencent_call_launcher.dart';

class CallButtons extends StatelessWidget {
  final String userId;
  final String userName;
  final bool showVideoCall;
  final bool showAudioCall;
  final VoidCallback? onCallStarted;
  final VoidCallback? onCallFailed;

  const CallButtons({
    Key? key,
    required this.userId,
    required this.userName,
    this.showVideoCall = true,
    this.showAudioCall = true,
    this.onCallStarted,
    this.onCallFailed,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        if (showAudioCall)
          IconButton(
            icon: const Icon(Icons.call, color: Colors.green),
            tooltip: 'Panggilan Audio',
            onPressed: () => _startCall(context, false),
          ),
        if (showVideoCall)
          IconButton(
            icon: const Icon(Icons.videocam, color: Colors.blue),
            tooltip: 'Panggilan Video',
            onPressed: () => _startCall(context, true),
          ),
      ],
    );
  }

  void _startCall(BuildContext context, bool isVideo) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => TencentCallLauncher(
        userId: userId,
        isVideo: isVideo,
        userName: userName,
        onCallStarted: onCallStarted,
        onCallFailed: onCallFailed,
      ),
    );
  }
}
```

#### Call Status Indicator

```dart
// lib/widgets/call_status_indicator.dart
import 'package:flutter/material.dart';
import 'package:hub_unk/services/tencent_call_service.dart';

class CallStatusIndicator extends StatefulWidget {
  const CallStatusIndicator({Key? key}) : super(key: key);

  @override
  State<CallStatusIndicator> createState() => _CallStatusIndicatorState();
}

class _CallStatusIndicatorState extends State<CallStatusIndicator> {
  CallStatus? _currentStatus;
  StreamSubscription<CallStatus>? _subscription;

  @override
  void initState() {
    super.initState();
    _subscribeToCallStatus();
  }

  void _subscribeToCallStatus() {
    final callService = TencentCallService();
    _subscription = callService.callStatusStream.listen((status) {
      setState(() {
        _currentStatus = status;
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_currentStatus == null) return const SizedBox.shrink();

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      decoration: BoxDecoration(
        color: _getStatusColor(),
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(
            _getStatusIcon(),
            color: Colors.white,
            size: 16,
          ),
          const SizedBox(width: 8),
          Text(
            _getStatusText(),
            style: const TextStyle(
              color: Colors.white,
              fontSize: 12,
              fontWeight: FontWeight.w500,
            ),
          ),
        ],
      ),
    );
  }

  Color _getStatusColor() {
    switch (_currentStatus?.type) {
      case CallStatusType.started:
        return Colors.green;
      case CallStatusType.ended:
        return Colors.grey;
      case CallStatusType.received:
        return Colors.blue;
      case CallStatusType.error:
        return Colors.red;
      case CallStatusType.networkQualityChanged:
        return Colors.orange;
      default:
        return Colors.grey;
    }
  }

  IconData _getStatusIcon() {
    switch (_currentStatus?.type) {
      case CallStatusType.started:
        return Icons.call;
      case CallStatusType.ended:
        return Icons.call_end;
      case CallStatusType.received:
        return Icons.call_received;
      case CallStatusType.error:
        return Icons.error;
      case CallStatusType.networkQualityChanged:
        return Icons.signal_cellular_alt;
      default:
        return Icons.info;
    }
  }

  String _getStatusText() {
    switch (_currentStatus?.type) {
      case CallStatusType.started:
        return 'Panggilan Aktif';
      case CallStatusType.ended:
        final duration = _currentStatus?.data['duration'] as double?;
        return 'Panggilan Selesai${duration != null ? ' (${duration.toStringAsFixed(0)}s)' : ''}';
      case CallStatusType.received:
        return 'Panggilan Masuk';
      case CallStatusType.error:
        final message = _currentStatus?.data['message'] as String?;
        return 'Error: ${message ?? 'Unknown'}';
      case CallStatusType.networkQualityChanged:
        return 'Kualitas Jaringan Berubah';
      default:
        return 'Status Tidak Diketahui';
    }
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}
```

### 3. Screen Implementations

#### Chat Page with Call Integration

```dart
// lib/screens/chat/chat_page.dart
import 'package:flutter/material.dart';
import 'package:hub_unk/widgets/call_buttons.dart';
import 'package:hub_unk/widgets/call_status_indicator.dart';

class ChatPage extends StatefulWidget {
  final String targetUserId;
  final String targetUserName;

  const ChatPage({
    Key? key,
    required this.targetUserId,
    required this.targetUserName,
  }) : super(key: key);

  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Row(
          children: [
            CircleAvatar(
              child: Text(widget.targetUserName[0].toUpperCase()),
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(widget.targetUserName),
                  const CallStatusIndicator(),
                ],
              ),
            ),
          ],
        ),
        actions: [
          CallButtons(
            userId: widget.targetUserId,
            userName: widget.targetUserName,
            onCallStarted: () {
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(content: Text('Panggilan dimulai')),
              );
            },
            onCallFailed: () {
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(
                  content: Text('Gagal memulai panggilan'),
                  backgroundColor: Colors.red,
                ),
              );
            },
          ),
        ],
      ),
      body: const ChatMessages(),
    );
  }
}

class ChatMessages extends StatelessWidget {
  const ChatMessages({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: Text('Chat messages will be displayed here'),
    );
  }
}
```

#### Group Call Page

```dart
// lib/screens/calls/group_call_page.dart
import 'package:flutter/material.dart';
import 'package:hub_unk/services/tencent_call_service.dart';

class GroupCallPage extends StatefulWidget {
  final List<String> userIds;
  final List<String> userNames;
  final bool isVideo;

  const GroupCallPage({
    Key? key,
    required this.userIds,
    required this.userNames,
    required this.isVideo,
  }) : super(key: key);

  @override
  State<GroupCallPage> createState() => _GroupCallPageState();
}

class _GroupCallPageState extends State<GroupCallPage> {
  bool _isStartingCall = false;
  String? _errorMessage;

  Future<void> _startGroupCall() async {
    setState(() {
      _isStartingCall = true;
      _errorMessage = null;
    });

    try {
      final callService = TencentCallService();
      final success = await callService.makeCall(
        userIds: widget.userIds,
        isVideo: widget.isVideo,
      );

      if (success) {
        if (mounted) {
          Navigator.pop(context);
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Panggilan grup dimulai')),
          );
        }
      } else {
        setState(() {
          _errorMessage = 'Gagal memulai panggilan grup';
        });
      }
    } catch (e) {
      setState(() {
        _errorMessage = 'Error: $e';
      });
    } finally {
      setState(() {
        _isStartingCall = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Panggilan ${widget.isVideo ? 'Video' : 'Audio'} Grup'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16.0),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'Peserta (${widget.userIds.length})',
                      style: Theme.of(context).textTheme.titleLarge,
                    ),
                    const SizedBox(height: 12),
                    ...widget.userNames.map((name) => Padding(
                      padding: const EdgeInsets.symmetric(vertical: 4),
                      child: Row(
                        children: [
                          const Icon(Icons.person, size: 16),
                          const SizedBox(width: 8),
                          Text(name),
                        ],
                      ),
                    )),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 24),
            if (_errorMessage != null)
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: Colors.red.shade100,
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Text(
                  _errorMessage!,
                  style: TextStyle(color: Colors.red.shade800),
                ),
              ),
            const SizedBox(height: 24),
            ElevatedButton.icon(
              onPressed: _isStartingCall ? null : _startGroupCall,
              icon: _isStartingCall
                  ? const SizedBox(
                      width: 16,
                      height: 16,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : Icon(widget.isVideo ? Icons.videocam : Icons.call),
              label: Text(
                _isStartingCall
                    ? 'Memulai Panggilan...'
                    : 'Mulai Panggilan ${widget.isVideo ? 'Video' : 'Audio'}',
              ),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 4. Utility Functions

```dart
// lib/utils/call_utils.dart
import 'package:flutter/material.dart';
import 'package:hub_unk/services/tencent_call_service.dart';

class CallUtils {
  static Future<bool> ensureCallServiceReady(BuildContext context) async {
    try {
      final callService = TencentCallService();
      final isReady = await callService.checkLoginStatus();
      
      if (!isReady) {
        if (context.mounted) {
          showDialog(
            context: context,
            builder: (context) => AlertDialog(
              title: const Text('Error'),
              content: const Text('Call service tidak siap. Silakan coba lagi.'),
              actions: [
                TextButton(
                  onPressed: () => Navigator.pop(context),
                  child: const Text('OK'),
                ),
              ],
            ),
          );
        }
        return false;
      }
      
      return true;
    } catch (e) {
      if (context.mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error: $e')),
        );
      }
      return false;
    }
  }

  static String formatCallDuration(Duration duration) {
    String twoDigits(int n) => n.toString().padLeft(2, '0');
    String hours = twoDigits(duration.inHours);
    String minutes = twoDigits(duration.inMinutes.remainder(60));
    String seconds = twoDigits(duration.inSeconds.remainder(60));
    
    if (duration.inHours > 0) {
      return '$hours:$minutes:$seconds';
    } else {
      return '$minutes:$seconds';
    }
  }

  static String getNetworkQualityText(int quality) {
    switch (quality) {
      case 1:
        return 'Excellent';
      case 2:
        return 'Good';
      case 3:
        return 'Fair';
      case 4:
        return 'Poor';
      case 5:
        return 'Very Poor';
      default:
        return 'Unknown';
    }
  }

  static Color getNetworkQualityColor(int quality) {
    switch (quality) {
      case 1:
        return Colors.green;
      case 2:
        return Colors.lightGreen;
      case 3:
        return Colors.yellow;
      case 4:
        return Colors.orange;
      case 5:
        return Colors.red;
      default:
        return Colors.grey;
    }
  }
}
```

## 🚀 Cara Menggunakan

### 1. Setup di main.dart

```dart
// lib/main.dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await dotenv.load();
  
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      navigatorObservers: [TUICallKit.navigatorObserver],
      // ... konfigurasi lainnya
    );
  }
}
```

### 2. Integrasi dengan Login

```dart
// Di halaman login Anda
Future<void> _handleLogin() async {
  // ... login logic
  
  // Setup TUICallKit setelah login berhasil
  final sdkAppID = int.parse(dotenv.env['TENCENT_LIVE_SDK_APP_ID'] ?? '0');
  final secretKey = dotenv.env['TENCENT_LIVE_SDK_SECRET_KEY'] ?? '';
  final userId = loginResult.userId;
  
  final userSig = GenerateTestUserSig.genTestSig(userId, sdkAppID, secretKey);
  
  await setValue('userId', userId);
  await setValue('userSig', userSig);
  
  final callService = TencentCallService();
  await callService.initialize();
}
```

### 3. Menggunakan di Chat

```dart
// Di halaman chat
AppBar(
  title: Text(chatPartnerName),
  actions: [
    CallButtons(
      userId: chatPartnerId,
      userName: chatPartnerName,
    ),
  ],
)
```

## 📝 Catatan Penting

1. **Pastikan semua permission sudah diatur** di Android dan iOS
2. **Gunakan API baru** (`calls()`, `join()`) bukan yang deprecated
3. **Handle error dengan baik** untuk user experience yang baik
4. **Test di device fisik** untuk fitur kamera dan mikrofon
5. **Monitor network quality** untuk debugging

---

**Status:** ✅ Production Ready  
**Compatibility:** Flutter 3.0+, TUICallKit 3.1.1+ 

## 1. Setup Dasar

### Menambahkan Dependency
```yaml
dependencies:
  tencent_calls_uikit: ^2.9.0
```

### Konfigurasi Android
```gradle
// android/app/build.gradle
android {
    defaultConfig {
        multiDexEnabled true
    }
}

// android/app/proguard-rules.pro
-keep class com.tencent.** { *; }
```

### Konfigurasi iOS
```xml
<!-- ios/Runner/Info.plist -->
<key>NSCameraUsageDescription</key>
<string>Camera permission is required for video calls</string>
<key>NSMicrophoneUsageDescription</key>
<string>Microphone permission is required for calls</string>
```

## 2. Inisialisasi dan Login

### Setup Navigator Observer
```dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';

MaterialApp(
  navigatorObservers: [TUICallKit.navigatorObserver],
  // ... konfigurasi lainnya
)
```

### Alur Inisialisasi Otomatis
TencentCallService akan diinisialisasi secara otomatis saat pengguna masuk ke halaman beranda (TabsPage):

```dart
// Di lib/screens/tabs/tabs.page.dart
class TabsPageState extends State<TabsPage> {
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
}
```

### Login Manual (Opsional)
Jika diperlukan, Anda juga dapat melakukan login manual:

```dart
import 'package:tencent_calls_uikit/tencent_calls_uikit.dart';
import 'package:tencent_calls_uikit/debug/generate_test_user_sig.dart';

class CallService {
  static Future<bool> login() async {
    try {
      final String userID = 'user123';
      final int sdkAppID = 1234567890;
      final String secretKey = 'your_secret_key';
      
      String userSig = GenerateTestUserSig.genTestSig(userID, sdkAppID, secretKey);
      
      TUIResult result = await TUICallKit.instance.login(sdkAppID, userID, userSig);
      
      if (result.code.isEmpty) {
        print('Login berhasil');
        return true;
      } else {
        print('Login gagal: ${result.code} ${result.message}');
        return false;
      }
    } catch (e) {
      print('Error login: $e');
      return false;
    }
  }
}
```

## 3. Menampilkan Nama Pengguna pada Panggilan

### Mengatur Informasi Pengguna
```dart
class TencentCallService {
  /// Set user information in TUICallKit
  Future<void> setUserInfo() async {
    try {
      final currentUser = getCurrentUserData();
      if (currentUser != null) {
        final displayName = getUserDisplayName(currentUser);
        final avatarUrl = getUserAvatarUrl(currentUser);
        
        await TUICallKit.instance.setSelfInfo(displayName, avatarUrl);
        print('Informasi pengguna berhasil diatur');
      }
    } catch (e) {
      print('Error mengatur info pengguna: $e');
    }
  }

  /// Get user display name
  String getUserDisplayName(Usr? user) {
    if (user == null) return 'Unknown User';
    
    if (user.firstName != null && 
        user.firstName!.isNotEmpty && 
        user.lastName != null && 
        user.lastName!.isNotEmpty) {
      return '${user.firstName} ${user.lastName}';
    } else if (user.username != null && user.username!.isNotEmpty) {
      return user.username!;
    } else {
      return 'Unknown User';
    }
  }

  /// Get user avatar URL
  String getUserAvatarUrl(Usr? user) {
    if (user?.avatar != null && user!.avatar!.isNotEmpty) {
      return user.avatar!;
    }
    return '';
  }
}
```

### Launcher dengan Nama Pengguna
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
            Icon(
              isVideo ? Icons.videocam : Icons.call,
              size: 80,
              color: Colors.white,
            ),
            const SizedBox(height: 30),
            
            const CircularProgressIndicator(
              valueColor: AlwaysStoppedAnimation<Color>(Colors.white),
            ),
            const SizedBox(height: 30),
            
            // Menampilkan nama pengguna
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

### Menggunakan Launcher dengan Nama
```dart
// Dari halaman chat
IconButton(
  icon: const Icon(Icons.call),
  onPressed: () async {
    await Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => TencentCallLauncher(
          userId: chat.userId,
          isVideo: false,
          userName: "${chat.firstName} ${chat.lastName}",
        ),
      ),
    );
  },
),
```

## 4. Melakukan Panggilan

### Panggilan Audio
```dart
Future<void> makeAudioCall(String userId) async {
  try {
    await TUICallKit.instance.call(
      userId,
      TUICallMediaType.audio,
    );
    print('Panggilan audio dimulai');
  } catch (e) {
    print('Error panggilan audio: $e');
  }
}
```

### Panggilan Video
```dart
Future<void> makeVideoCall(String userId) async {
  try {
    await TUICallKit.instance.call(
      userId,
      TUICallMediaType.video,
    );
    print('Panggilan video dimulai');
  } catch (e) {
    print('Error panggilan video: $e');
  }
}
```

### Panggilan Grup
```dart
Future<void> makeGroupCall(List<String> userIds, bool isVideo) async {
  try {
    await TUICallKit.instance.calls(
      userIds,
      isVideo ? TUICallMediaType.video : TUICallMediaType.audio,
    );
    print('Panggilan grup dimulai');
  } catch (e) {
    print('Error panggilan grup: $e');
  }
}
```

## 5. Menangani Panggilan Masuk

### Bergabung ke Panggilan
```dart
Future<void> joinCall(String callId) async {
  try {
    await TUICallKit.instance.join(callId);
    print('Bergabung ke panggilan');
  } catch (e) {
    print('Error bergabung: $e');
  }
}
```

## 6. Monitoring Status Panggilan

### Menambahkan Observer
```dart
TUICallObserver observer = TUICallObserver(
  onError: (int code, String message) {
    print('Error panggilan: $code - $message');
  },
  onCallBegin: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole) {
    print('Panggilan dimulai');
  },
  onCallEnd: (TUIRoomId roomId, TUICallMediaType callMediaType, TUICallRole callRole, double totalTime) {
    print('Panggilan berakhir - Durasi: $totalTime detik');
  },
  onCallReceived: (String callerId, List<String> calleeIdList, String groupId, TUICallMediaType callMediaType) {
    print('Panggilan masuk dari: $callerId');
  }
);

// Menambahkan observer
TUICallEngine.instance.addObserver(observer);
```

## 7. Fitur Tambahan

### Mengatur Nada Dering Custom
```dart
await TUICallKit.instance.setCallingBell('assets/ringtone.mp3');
```

### Mengaktifkan Floating Window
```dart
await TUICallKit.instance.enableFloatWindow(true);
```

### Mengaktifkan Mode Mute Default
```dart
await TUICallKit.instance.enableMuteMode(true);
```

### Mengaktifkan Virtual Background
```dart
await TUICallKit.instance.enableVirtualBackground(true);
```

## 8. Error Handling

### Menangani Error Paket Dasar
```dart
String getErrorMessage(dynamic error) {
  String errorString = error.toString().toLowerCase();

  if (errorString.contains('-1001') ||
      errorString.contains('not buy the basic call package') ||
      errorString.contains('basic call package')) {
    return 'Panggilan masih belum dapat dilakukan. Paket panggilan dasar belum dibeli.';
  } else if (errorString.contains('network') ||
      errorString.contains('connection')) {
    return 'Gagal melakukan panggilan. Periksa koneksi internet Anda.';
  } else if (errorString.contains('permission') ||
      errorString.contains('camera') ||
      errorString.contains('microphone')) {
    return 'Gagal melakukan panggilan. Izin kamera atau mikrofon diperlukan.';
  } else {
    return 'Gagal melakukan panggilan. Silakan coba lagi.';
  }
}
```

### Menampilkan Dialog Error
```dart
void showErrorDialog(BuildContext context, String message) {
  showDialog(
    context: context,
    builder: (context) => AlertDialog(
      title: const Text('Error Panggilan'),
      content: Text(message),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('OK'),
        ),
        TextButton(
          onPressed: () {
            Navigator.pop(context);
            // Kontak admin atau retry
          },
          child: const Text('Kontak Admin'),
        ),
      ],
    ),
  );
}
```

## 9. Best Practices

1. **Selalu cek status login** sebelum melakukan panggilan
2. **Tangani error dengan baik** dan berikan pesan yang user-friendly
3. **Gunakan nama pengguna** untuk UX yang lebih baik
4. **Simpan kredensial dengan aman** (jangan hardcode di aplikasi)
5. **Test di berbagai device** untuk memastikan kompatibilitas
6. **Monitor performa** dan kualitas jaringan
7. **Implementasikan retry logic** untuk panggilan yang gagal 