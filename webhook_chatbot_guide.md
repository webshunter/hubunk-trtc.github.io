# Panduan Implementasi Chatbot dengan Webhook di Flutter

## 📋 Overview

Dokumentasi ini menjelaskan implementasi chatbot Flutter yang menggunakan webhook untuk komunikasi dengan AI service, menggantikan implementasi Ollama sebelumnya.

## 🔧 Konfigurasi Webhook

### URL Webhook
```dart
static const String webhookUrl = "https://hook.gugusdarmayanto.my.id/webhook/0a4ca5b0-3d99-43d8-abca-ad792570f670";
```

### Format Request
```dart
{
  "sessionId": "string",
  "action": "sendMessage", 
  "chatInput": "string"
}
```

### Format Response
```dart
[
  {
    "aiResponse": "string",
    "sessionId": "string"
  }
]
```

## 📁 Struktur File

### 1. APIs Service (`lib/screens/openai/openai_apis.dart`)

```dart
import 'dart:convert';
import 'dart:developer';
import 'package:http/http.dart';
import 'package:nb_utils/nb_utils.dart';

class APIs {
  // Webhook URL
  static const String webhookUrl = "https://hook.gugusdarmayanto.my.id/webhook/0a4ca5b0-3d99-43d8-abca-ad792570f670";
  
  // Generate session ID
  static String generateSessionId() {
    final timestamp = DateTime.now().millisecondsSinceEpoch.toRadixString(36);
    final randomStr = (DateTime.now().microsecondsSinceEpoch % 1000000).toRadixString(36);
    return "${timestamp}-${randomStr}";
  }

  // Get or create session ID
  static String getSessionId() {
    try {
      String? sessionId = getStringAsync('chatbot_session_id');
      if (sessionId == null || sessionId.isEmpty) {
        sessionId = generateSessionId();
        setValue('chatbot_session_id', sessionId);
      }
      return sessionId;
    } catch (e) {
      log('Error getting session ID: $e');
      return generateSessionId();
    }
  }

  // Send message to webhook
  static Future<String> getAnswer({required String question}) async {
    try {
      final sessionId = getSessionId();
      
      log('Sending message to webhook: $question');
      log('Session ID: $sessionId');

      final res = await post(
        Uri.parse(webhookUrl),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode([{
          'sessionId': sessionId,
          'action': 'sendMessage',
          'chatInput': question
        }]),
      );

      if (res.statusCode != 200) {
        log('Error status code: ${res.statusCode}');
        return 'Server error: ${res.statusCode}';
      }

      final data = jsonDecode(res.body);
      log('Webhook response: $data');

      if (data == null || data.isEmpty) {
        return 'Invalid response from server';
      }

      final aiResponse = data[0]?['aiResponse'] ?? 'No response from server';
      return aiResponse;
    } catch (e) {
      log('getAnswer error: $e');
      return 'Something went wrong (Try again in sometime)';
    }
  }

  // Clear chat history
  static Future<void> clearChatHistory() async {
    try {
      await removeKey('chatbot_session_id');
      log('Chat history cleared');
    } catch (e) {
      log('Error clearing chat history: $e');
    }
  }
}
```

### 2. Chat Controller (`lib/screens/openai/chatbot/chat_controller.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:hub_unk/screens/openai/openai_apis.dart';
import 'package:nb_utils/nb_utils.dart';

class Message {
  String msg;
  final MessageType msgType;

  Message({required this.msg, required this.msgType});
}

enum MessageType { user, bot }

class ChatController extends ChangeNotifier {
  final scrollC = ScrollController();

  void disposeScrollController() {
    scrollC.dispose();
    notifyListeners();
  }

  bool isLoading = false;
  List<Message> list = <Message>[
    Message(
      msg: 'Selamat datang! Saya INA, asisten virtual dari Hubunk. Saya siap membantu Anda dengan informasi seputar layanan kami. Ada yang bisa saya bantu?',
      msgType: MessageType.bot
    )
  ];

  // Initialize chat with session management
  Future<void> initializeChat() async {
    try {
      final sessionId = APIs.getSessionId();
      log('Chat initialized with session ID: $sessionId');
      
      if (list.length == 1 && list[0].msgType == MessageType.bot) {
        return;
      }
    } catch (e) {
      log('Error initializing chat: $e');
    }
  }

  // Clear chat history
  Future<void> clearChat() async {
    try {
      await APIs.clearChatHistory();
      list.clear();
      list.add(Message(
        msg: 'Selamat datang! Saya INA, asisten virtual dari Hubunk. Saya siap membantu Anda dengan informasi seputar layanan kami. Ada yang bisa saya bantu?',
        msgType: MessageType.bot
      ));
      notifyListeners();
      _scrollDown();
    } catch (e) {
      log('Error clearing chat: $e');
    }
  }

  // Send message to webhook
  Future<void> askQuestion({required String textC}) async {
    if (textC.trim().isNotEmpty) {
      isLoading = true;
      notifyListeners();
      
      // Add user message
      list.add(Message(msg: textC, msgType: MessageType.user));
      list.add(Message(msg: '', msgType: MessageType.bot));
      _scrollDown();

      try {
        // Get answer from webhook
        final res = await APIs.getAnswer(question: textC);

        // Update bot response
        list.removeLast();
        list.add(Message(msg: res, msgType: MessageType.bot));
        _scrollDown();
      } catch (e) {
        log('Error getting answer: $e');
        list.removeLast();
        list.add(Message(
          msg: 'Maaf, terjadi kesalahan. Silakan coba lagi.',
          msgType: MessageType.bot
        ));
        _scrollDown();
      }

      textC = '';
    } else {
      toast('Tulis pesan terlebih dahulu!');
    }
    
    isLoading = false;
    notifyListeners();
  }

  // For moving to end message
  void _scrollDown() {
    scrollC.animateTo(scrollC.position.maxScrollExtent,
        duration: const Duration(milliseconds: 500), curve: Curves.ease);
  }
}
```

### 3. Chat UI (`lib/screens/openai/chatbot/ai_chatbot.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:hub_unk/consts/colors.dart';
import 'package:hub_unk/localization/localization_constant.dart';
import 'package:hub_unk/screens/openai/chatbot/chat_controller.dart';
import 'package:hub_unk/screens/openai/chatbot/message_card.dart';
import 'package:nb_utils/nb_utils.dart';
import 'package:provider/provider.dart';

class AiChatBot extends StatefulWidget {
  const AiChatBot({super.key});

  @override
  State<AiChatBot> createState() => _AiChatBotState();
}

class _AiChatBotState extends State<AiChatBot> {
  final TextEditingController textC = TextEditingController();

  @override
  void initState() {
    super.initState();
    // Initialize chat with session management
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Provider.of<ChatController>(context, listen: false).initializeChat();
    });
  }

  @override
  void dispose() {
    textC.dispose();
    Provider.of<ChatController>(context, listen: false)
        .disposeScrollController();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Consumer<ChatController>(
      builder: (context, _c, child) => Scaffold(
        appBar: AppBar(
          title: Row(
            children: [
              SizedBox(
                height: 20,
                width: 100,
                child: Image(
                  image: NetworkImage(
                    getStringAsync("appLogo"),
                  ),
                ),
              ),
              Text(
                translate(context, "chatbot")!,
                style: const TextStyle(
                  fontStyle: FontStyle.italic,
                  fontSize: 14,
                  fontWeight: FontWeight.w400,
                ),
              ),
            ],
          ),
          actions: [
            // Clear chat button
            IconButton(
              icon: const Icon(Icons.clear_all),
              onPressed: () {
                showDialog(
                  context: context,
                  builder: (BuildContext context) {
                    return AlertDialog(
                      title: const Text('Hapus Riwayat Chat'),
                      content: const Text('Apakah Anda yakin ingin menghapus semua riwayat chat?'),
                      actions: [
                        TextButton(
                          onPressed: () => Navigator.of(context).pop(),
                          child: const Text('Batal'),
                        ),
                        TextButton(
                          onPressed: () {
                            Navigator.of(context).pop();
                            _c.clearChat();
                          },
                          child: const Text('Hapus', style: TextStyle(color: Colors.red)),
                        ),
                      ],
                    );
                  },
                );
              },
              tooltip: 'Hapus riwayat chat',
            ),
          ],
        ),

        floatingActionButtonLocation: FloatingActionButtonLocation.centerFloat,
        floatingActionButton: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 8),
          child: Row(
            children: [
              Expanded(
                  child: TextFormField(
                controller: textC,
                onTapOutside: (e) => FocusScope.of(context).unfocus(),
                decoration: InputDecoration(
                    suffixIcon: _c.isLoading
                        ? const Padding(
                            padding: EdgeInsets.all(8.0),
                            child: CircularProgressIndicator(),
                          )
                        : IconButton(
                            onPressed: () {
                              _c.askQuestion(textC: textC.text);
                              textC.clear();
                            },
                            icon: const Icon(Icons.send,
                                color: AppColors.primaryColor, size: 28),
                          ),
                    fillColor: Theme.of(context).scaffoldBackgroundColor,
                    filled: true,
                    isDense: true,
                    hintText: translate(context, 'ask_me_anything'),
                    hintStyle: const TextStyle(fontSize: 14),
                    border: const OutlineInputBorder(
                        borderRadius: BorderRadius.all(Radius.circular(50)))),
              )),
            ],
          ),
        ),

        body: ListView.builder(
          itemBuilder: (context, index) {
            return MessageCard(
              message: _c.list[index],
              index: index,
            );
          },
          itemCount: _c.list.length,
          physics: const BouncingScrollPhysics(),
          controller: _c.scrollC,
          padding: EdgeInsets.only(
              top: MediaQuery.sizeOf(context).height * .02,
              bottom: MediaQuery.sizeOf(context).height * .1),
        ),
      ),
    );
  }
}
```

## 🔄 Perbandingan dengan JavaScript

### JavaScript Implementation
```javascript
// Send message to webhook
const res = await fetch("https://hook.gugusdarmayanto.my.id/webhook/0a4ca5b0-3d99-43d8-abca-ad792570f670", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify([{
        sessionId: this.currentSessionId,
        action: "sendMessage",
        chatInput: text
    }]),
});

const data = await res.json();
const aiResponse = data[0]?.aiResponse || "Tidak ada balasan.";
```

### Flutter Implementation
```dart
// Send message to webhook
final res = await post(
  Uri.parse(webhookUrl),
  headers: {'Content-Type': 'application/json'},
  body: jsonEncode([{
    'sessionId': sessionId,
    'action': 'sendMessage',
    'chatInput': question
  }]),
);

final data = jsonDecode(res.body);
final aiResponse = data[0]?['aiResponse'] ?? 'No response from server';
```

## 🎯 Fitur yang Diimplementasi

### ✅ Session Management
- Generate unique session ID
- Store session ID in local storage
- Maintain conversation context

### ✅ Welcome Message
- Custom welcome message in Indonesian
- Consistent with JavaScript implementation

### ✅ Error Handling
- Network error handling
- Server error handling
- User-friendly error messages

### ✅ UI Features
- Loading indicators
- Clear chat functionality
- Auto-scroll to bottom
- Responsive design

### ✅ Local Storage
- Session persistence
- Chat history management
- Clear chat functionality

## 🚀 Cara Menggunakan

### 1. Navigasi ke Chatbot
```dart
Navigator.push(
  context,
  MaterialPageRoute(builder: (context) => const AiChatBot()),
);
```

### 2. Kirim Pesan
```dart
// Pesan akan otomatis dikirim ke webhook
// dan response akan ditampilkan
```

### 3. Hapus Riwayat
```dart
// Klik tombol clear di app bar
// atau gunakan method clearChat()
```

## 🔧 Konfigurasi Tambahan

### Environment Variables
```dart
// Tambahkan di .env jika diperlukan
WEBHOOK_URL=https://hook.gugusdarmayanto.my.id/webhook/0a4ca5b0-3d99-43d8-abca-ad792570f670
```

### Dependencies
```yaml
dependencies:
  http: ^1.1.0
  nb_utils: ^latest_version
```

## 📊 Testing

### Test Webhook Connection
```dart
void testWebhook() async {
  try {
    final response = await APIs.getAnswer(question: "Hello");
    print('Response: $response');
  } catch (e) {
    print('Error: $e');
  }
}
```

### Test Session Management
```dart
void testSession() {
  final sessionId1 = APIs.getSessionId();
  final sessionId2 = APIs.getSessionId();
  print('Session IDs match: ${sessionId1 == sessionId2}');
}
```

## 🐛 Troubleshooting

### Error: "Server error: 404"
- Periksa URL webhook
- Pastikan endpoint tersedia

### Error: "Invalid response from server"
- Periksa format response dari webhook
- Pastikan response sesuai format yang diharapkan

### Error: "Something went wrong"
- Periksa koneksi internet
- Periksa log untuk detail error

## 📝 Changelog

### v2.0.0 (Current)
- ✅ Migrasi dari Ollama ke Webhook
- ✅ Implementasi session management
- ✅ Welcome message dalam bahasa Indonesia
- ✅ Clear chat functionality
- ✅ Error handling yang lebih baik

### v1.0.0 (Previous)
- ⚠️ Menggunakan Ollama API
- ⚠️ Tidak ada session management
- ⚠️ Welcome message dalam bahasa Inggris

## 🔗 Referensi

- [Webhook Documentation](https://hook.gugusdarmayanto.my.id/)
- [HTTP Package](https://pub.dev/packages/http)
- [nb_utils Package](https://pub.dev/packages/nb_utils)
- [Flutter State Management](https://flutter.dev/docs/development/data-and-backend/state-mgmt/simple)

---

**Last Updated:** $(date)  
**Version:** 2.0.0  
**Status:** ✅ Complete & Ready for Production 