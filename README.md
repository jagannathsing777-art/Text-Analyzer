import 'dart:async';
import 'dart:math';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:provider/provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:share_plus/share_plus.dart';

// ============================================================================
// 1. 🌍 MULTI-LANGUAGE CONSTANTS & RTL LOGIC
// ============================================================================

const Map<String, Map<String, String>> APP_STRINGS = {
  'en': {
    'app_name': 'Ultra Analyzer',
    'paste_hint': 'Paste massive text here...',
    'analyze_btn': 'ANALYZE TEXT',
    'premium_btn': 'GO PREMIUM',
    'results': 'Analysis Results',
    'chars': 'Characters',
    'words': 'Words',
    'speaking': 'Speaking Time',
    'reading': 'Reading Time',
    'upper': 'Uppercase',
    'lower': 'Lowercase',
    'digits': 'Digits',
    'lines': 'Lines',
    'saved': 'Saved to Watchlist',
    'loading': 'Processing Heavy Load...',
  },
  'hi': {
    'app_name': 'अल्ट्रा विश्लेषक',
    'paste_hint': 'यहाँ विशाल पाठ पेस्ट करें...',
    'analyze_btn': 'विश्लेषण करें',
    'premium_btn': 'प्रीमियम बनें',
    'results': 'विश्लेषण परिणाम',
    'chars': 'अक्षर',
    'words': 'शब्द',
    'speaking': 'बोलने का समय',
    'reading': 'पढ़ने का समय',
    'upper': 'बड़े अक्षर',
    'lower': 'छोटे अक्षर',
    'digits': 'अंक',
    'lines': 'पंक्तियाँ',
    'saved': 'वॉचलिस्ट में सहेजा गया',
    'loading': 'भारी लोड प्रोसेस हो रहा है...',
  },
  'ur': { // RTL Language
    'app_name': 'الٹرا تجزیہ کار',
    'paste_hint': 'یہاں بھاری متن چسپاں کریں...',
    'analyze_btn': 'تجزیہ کریں',
    'premium_btn': 'پریمیم حاصل کریں',
    'results': 'تجزیاتی نتائج',
    'chars': 'حروف',
    'words': 'الفاظ',
    'speaking': 'بولنے کا وقت',
    'reading': 'پڑھنے کا وقت',
    'upper': 'بڑے حروف',
    'lower': 'چھوٹے حروف',
    'digits': 'ہندسے',
    'lines': 'لائنیں',
    'saved': 'واچ لسٹ میں محفوظ کر لیا گیا',
    'loading': 'پروسیسنگ جاری ہے...',
  },
  'ar': { // RTL Language
    'app_name': 'المحلل الفائق',
    'paste_hint': 'الصق النص الضخم هنا...',
    'analyze_btn': 'تحليل',
    'premium_btn': 'اشترك الآن',
    'results': 'نتائج التحليل',
    'chars': 'الشخصيات',
    'words': 'الكلمات',
    'speaking': 'وقت التحدث',
    'reading': 'وقت القراءة',
    'upper': 'أحرف كبيرة',
    'lower': 'أحرف صغيرة',
    'digits': 'أرقام',
    'lines': 'الخطوط',
    'saved': 'تم الحفظ في القائمة',
    'loading': 'جاري المعالجة...',
  },
  'bn': {
    'app_name': 'আল্ট্রা অ্যানালাইজার',
    'paste_hint': 'এখানে বিশাল পাঠ্য পেস্ট করুন...',
    'analyze_btn': 'বিশ্লেষণ করুন',
    'premium_btn': 'প্রিমিয়াম যান',
    'results': 'ফলাফল',
    'chars': 'অক্ষর',
    'words': 'শব্দ',
    'speaking': 'বলার সময়',
    'reading': 'পড়ার সময়',
    'upper': 'বড় হাতের',
    'lower': 'ছোট হাতের',
    'digits': 'সংখ্যা',
    'lines': 'লাইন',
    'saved': 'সংরক্ষণ করা হয়েছে',
    'loading': 'প্রসেসিং হচ্ছে...',
  },
};

// ============================================================================
// 2. 🧠 DATA MODELS & HEAVY LOAD ISOLATE LOGIC
// ============================================================================

class TextStats {
  final int totalChars;
  final int upperCase;
  final int lowerCase;
  final int digits;
  final int spaces;
  final int words;
  final int lines;
  final int paragraphs;
  final int sentences; // full stops
  final int commas;
  final int specialChars;
  final String readingTime;
  final String speakingTime;
  final String contentSnippet; // For watchlist

  TextStats({
    required this.totalChars,
    required this.upperCase,
    required this.lowerCase,
    required this.digits,
    required this.spaces,
    required this.words,
    required this.lines,
    required this.paragraphs,
    required this.sentences,
    required this.commas,
    required this.specialChars,
    required this.readingTime,
    required this.speakingTime,
    required this.contentSnippet,
  });
}

// ⚠️ STATIC FUNCTION FOR BACKGROUND ISOLATE (MUST BE TOP-LEVEL)
TextStats _heavyCompute(String text) {
  if (text.isEmpty) {
    return TextStats(
      totalChars: 0, upperCase: 0, lowerCase: 0, digits: 0, spaces: 0,
      words: 0, lines: 0, paragraphs: 0, sentences: 0, commas: 0,
      specialChars: 0, readingTime: "0s", speakingTime: "0s", contentSnippet: "",
    );
  }

  int upper = 0, lower = 0, digits = 0, spaces = 0, sentences = 0, commas = 0, special = 0;
  
  // O(N) Single Pass Loop for efficiency
  for (int i = 0; i < text.length; i++) {
    var char = text[i];
    if (RegExp(r'[A-Z]').hasMatch(char)) upper++;
    else if (RegExp(r'[a-z]').hasMatch(char)) lower++;
    else if (RegExp(r'[0-9]').hasMatch(char)) digits++;
    else if (char == ' ') spaces++;
    else if (char == '.') sentences++;
    else if (char == ',') commas++;
    else if (RegExp(r'[!@#\$%^&*():";?]').hasMatch(char)) special++;
  }

  // Complex splitting
  List<String> wordsList = text.trim().split(RegExp(r'\s+'));
  wordsList.removeWhere((w) => w.isEmpty);
  int wordCount = wordsList.length;

  int lines = text.split('\n').length;
  int paragraphs = text.split('\n\n').length;

  // Time Calc
  double readMins = wordCount / 200; // 200 wpm
  double speakMins = wordCount / 130; // 130 wpm

  return TextStats(
    totalChars: text.length,
    upperCase: upper,
    lowerCase: lower,
    digits: digits,
    spaces: spaces,
    words: wordCount,
    lines: lines,
    paragraphs: paragraphs,
    sentences: sentences,
    commas: commas,
    specialChars: special,
    readingTime: "${(readMins * 60).toStringAsFixed(1)}s",
    speakingTime: "${(speakMins * 60).toStringAsFixed(1)}s",
    contentSnippet: text.length > 50 ? text.substring(0, 50) + "..." : text,
  );
}

// ============================================================================
// 3. 🤖 AI AD CONTROLLER & APP STATE (PROVIDER)
// ============================================================================

class AppController extends ChangeNotifier {
  // Settings
  String _languageCode = 'en';
  bool _isPremium = false;
  bool _isDarkMode = false;
  
  // Analytics Logic
  bool _isProcessing = false;
  TextStats? _currentStats;
  List<TextStats> _watchlist = [];

  // 🤖 AI Ad Logic Variables
  int _analysisSessionCount = 0;
  DateTime? _lastInterstitialTime;
  final int _interstitialCooldownMin = 3; 

  // Getters
  String get languageCode => _languageCode;
  bool get isPremium => _isPremium;
  bool get isProcessing => _isProcessing;
  bool get isDarkMode => _isDarkMode;
  TextStats? get currentStats => _currentStats;
  List<TextStats> get watchlist => _watchlist;
  
  // Check RTL
  bool get isRTL => _languageCode == 'ur' || _languageCode == 'ar';

  AppController() {
    _loadPrefs();
  }

  // Load Settings
  Future<void> _loadPrefs() async {
    final prefs = await SharedPreferences.getInstance();
    _isPremium = prefs.getBool('isPremium') ?? false;
    _languageCode = prefs.getString('language') ?? 'en';
    notifyListeners();
  }

  // 🌍 Change Language
  void setLanguage(String code) {
    _languageCode = code;
    SharedPreferences.getInstance().then((p) => p.setString('language', code));
    notifyListeners();
  }

  // 💎 Activate Premium (Simulation)
  void upgradeToPremium() {
    _isPremium = true;
    SharedPreferences.getInstance().then((p) => p.setBool('isPremium', true));
    notifyListeners();
  }

  // ⚙️ Heavy Load Execution
  Future<void> analyzeText(String text) async {
    // 1. UI LOCK & AD PAUSE
    _isProcessing = true;
    notifyListeners(); 

    // 2. RUN ISOLATE (Background Thread)
    // Simulate slight delay to show loader for demo purposes on small text
    if (text.length < 5000) await Future.delayed(const Duration(milliseconds: 800));
    
    final result = await compute(_heavyCompute, text);
    
    // 3. UPDATE STATE
    _currentStats = result;
    _isProcessing = false;
    _analysisSessionCount++;
    notifyListeners();
  }

  // ⭐ Watchlist Logic
  void addToWatchlist() {
    if (_currentStats != null) {
      _watchlist.add(_currentStats!);
      notifyListeners();
    }
  }

  // 🤖 SMART AD DECISION ENGINE
  // Returns TRUE if we should show an Interstitial Ad right now
  bool shouldShowInterstitial() {
    if (_isPremium) return false;
    
    // Rule: Never show on first analysis (UX)
    if (_analysisSessionCount <= 1) return false;

    // Rule: Cooldown Check
    if (_lastInterstitialTime != null) {
      final diff = DateTime.now().difference(_lastInterstitialTime!);
      if (diff.inMinutes < _interstitialCooldownMin) return false;
    }

    // Rule: Frequency (Every 3rd analysis)
    if (_analysisSessionCount % 3 == 0) {
      _lastInterstitialTime = DateTime.now();
      return true;
    }

    return false;
  }
}

// ============================================================================
// 4. 📱 FLUTTER UI IMPLEMENTATION
// ============================================================================

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  // MobileAds.instance.initialize(); // Uncomment for Real Ads
  
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AppController()),
      ],
      child: const UltraAnalyzerApp(),
    ),
  );
}

class UltraAnalyzerApp extends StatelessWidget {
  const UltraAnalyzerApp({super.key});

  @override
  Widget build(BuildContext context) {
    final controller = Provider.of<AppController>(context);
    
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Ultra Text Analyzer',
      theme: ThemeData.light(useMaterial3: true).copyWith(
        primaryColor: Colors.indigo,
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
      ),
      darkTheme: ThemeData.dark(useMaterial3: true),
      themeMode: controller.isDarkMode ? ThemeMode.dark : ThemeMode.system,
      // 🌍 Dynamic RTL Handling
      builder: (context, child) {
        return Directionality(
          textDirection: controller.isRTL ? TextDirection.rtl : TextDirection.ltr,
          child: child!,
        );
      },
      home: const HomeScreen(),
    );
  }
}

// ------------------- HOME SCREEN -------------------

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final TextEditingController _textController = TextEditingController();

  String t(BuildContext context, String key) {
    final lang = Provider.of<AppController>(context).languageCode;
    return APP_STRINGS[lang]?[key] ?? key;
  }

  void _handleAnalyze() async {
    final controller = Provider.of<AppController>(context, listen: false);
    
    // Trigger Analysis
    await controller.analyzeText(_textController.text);

    if (!mounted) return;

    // 🤖 AI SMART NAVIGATOR
    // Check if Interstitial is needed
    if (controller.shouldShowInterstitial()) {
      // Simulate Ad Show (In real app: InterstitialAd.show())
      showDialog(
        context: context,
        barrierDismissible: false,
        builder: (c) => const MockAdDialog(),
      ).then((_) {
        // Navigate after Ad
        Navigator.push(context, MaterialPageRoute(builder: (_) => const ResultScreen()));
      });
    } else {
      // Direct Navigation (Premium or Safe Zone)
      Navigator.push(context, MaterialPageRoute(builder: (_) => const ResultScreen()));
    }
  }

  @override
  Widget build(BuildContext context) {
    final controller = Provider.of<AppController>(context);

    return Scaffold(
      appBar: AppBar(
        title: Text(t(context, 'app_name'), style: const TextStyle(fontWeight: FontWeight.bold)),
        actions: [
          // Language Switcher
          DropdownButton<String>(
            value: controller.languageCode,
            underline: const SizedBox(),
            icon: const Icon(Icons.language, color: Colors.white),
            dropdownColor: Colors.indigo,
            onChanged: (val) => controller.setLanguage(val!),
            items: ['en', 'hi', 'ur', 'ar', 'bn'].map((l) {
              return DropdownMenuItem(value: l, child: Text(l.toUpperCase()));
            }).toList(),
          ),
          const SizedBox(width: 10),
          // Premium Icon
          if (!controller.isPremium)
            IconButton(
              icon: const Icon(Icons.workspace_premium, color: Colors.amber),
              onPressed: () => _showPremiumDialog(context),
            ),
        ],
        backgroundColor: Colors.indigo,
        foregroundColor: Colors.white,
      ),
      body: Column(
        children: [
          // 📝 TEXT INPUT AREA
          Expanded(
            child: Container(
              margin: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Theme.of(context).cardColor,
                borderRadius: BorderRadius.circular(12),
                boxShadow: [BoxShadow(color: Colors.black12, blurRadius: 5)],
              ),
              child: TextField(
                controller: _textController,
                maxLines: null,
                expands: true,
                style: const TextStyle(fontSize: 16),
                decoration: InputDecoration(
                  contentPadding: const EdgeInsets.all(16),
                  hintText: t(context, 'paste_hint'),
                  border: InputBorder.none,
                ),
              ),
            ),
          ),
          
          // 🚀 ACTION BUTTON (Disable during processing)
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: SizedBox(
              width: double.infinity,
              height: 60,
              child: ElevatedButton(
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.indigo,
                  foregroundColor: Colors.white,
                  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
                  elevation: 5,
                ),
                onPressed: controller.isProcessing ? null : _handleAnalyze,
                child: controller.isProcessing
                    ? Row(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(color: Colors.white, strokeWidth: 2)),
                          const SizedBox(width: 15),
                          Text(t(context, 'loading')),
                        ],
                      )
                    : Text(t(context, 'analyze_btn'), style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
              ),
            ),
          ),
          
          const SizedBox(height: 10),

          // 📊 SMART BANNER AREA
          // (Hidden if Premium OR Heavy Processing)
          if (!controller.isPremium && !controller.isProcessing)
            Container(
              height: 50,
              width: double.infinity,
              color: Colors.grey[200],
              alignment: Alignment.center,
              child: const Text("📢 SMART AD BANNER (Refreshes 60s)", style: TextStyle(color: Colors.grey)),
            ),
        ],
      ),
    );
  }

  void _showPremiumDialog(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (c) => Container(
        padding: const EdgeInsets.all(20),
        height: 250,
        child: Column(
          children: [
            const Icon(Icons.diamond, color: Colors.amber, size: 50),
            const SizedBox(height: 10),
            const Text("Go Premium", style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
            const Text("No Ads. Unlimited Heavy Load. Priority Support."),
            const Spacer(),
            ElevatedButton(
              onPressed: () {
                Provider.of<AppController>(context, listen: false).upgradeToPremium();
                Navigator.pop(context);
              },
              child: const Text("UNLOCK NOW (Simulated)"),
            )
          ],
        ),
      ),
    );
  }
}

// ------------------- RESULT SCREEN -------------------

class ResultScreen extends StatelessWidget {
  const ResultScreen({super.key});

  String t(BuildContext context, String key) {
    final lang = Provider.of<AppController>(context).languageCode;
    return APP_STRINGS[lang]?[key] ?? key;
  }

  @override
  Widget build(BuildContext context) {
    final controller = Provider.of<AppController>(context);
    final stats = controller.currentStats!;

    return Scaffold(
      appBar: AppBar(
        title: Text(t(context, 'results')),
        actions: [
          IconButton(
            icon: const Icon(Icons.favorite_border),
            onPressed: () {
               controller.addToWatchlist();
               ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(t(context, 'saved'))));
            },
          ),
          IconButton(
            icon: const Icon(Icons.share),
            onPressed: () {
              Share.share("Analyzed Text: ${stats.words} Words, ${stats.speakingTime} Speaking Time.");
            },
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // TOP STATS ROW
          Row(
            children: [
              _buildBigCard(context, t(context, 'words'), "${stats.words}", Colors.blueAccent),
              const SizedBox(width: 10),
              _buildBigCard(context, t(context, 'chars'), "${stats.totalChars}", Colors.orangeAccent),
            ],
          ),
          const SizedBox(height: 15),

          // SPEAKING TIME
          Card(
            elevation: 3,
            child: ListTile(
              leading: const Icon(Icons.timer, color: Colors.green),
              title: Text(t(context, 'speaking')),
              trailing: Text(stats.speakingTime, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            ),
          ),

          // NATIVE AD SLOT (If not premium)
          if (!controller.isPremium)
            Container(
              margin: const EdgeInsets.symmetric(vertical: 10),
              height: 100,
              decoration: BoxDecoration(
                color: Colors.grey[100],
                border: Border.all(color: Colors.grey[300]!),
                borderRadius: BorderRadius.circular(8),
              ),
              alignment: Alignment.center,
              child: const Text("📢 NATIVE AD (Result Screen)", style: TextStyle(color: Colors.grey)),
            ),

          // DETAILED GRID
          GridView.count(
            crossAxisCount: 2,
            shrinkWrap: true,
            physics: const NeverScrollableScrollPhysics(),
            childAspectRatio: 2.5,
            crossAxisSpacing: 10,
            mainAxisSpacing: 10,
            children: [
              _buildMiniStat(t(context, 'lines'), "${stats.lines}"),
              _buildMiniStat(t(context, 'paragraphs'), "${stats.paragraphs}"),
              _buildMiniStat(t(context, 'digits'), "${stats.digits}"),
              _buildMiniStat(t(context, 'sentences'), "${stats.sentences}"),
              _buildMiniStat(t(context, 'upper'), "${stats.upperCase}"),
              _buildMiniStat(t(context, 'lower'), "${stats.lowerCase}"),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildBigCard(BuildContext context, String title, String value, Color color) {
    return Expanded(
      child: Container(
        padding: const EdgeInsets.all(20),
        decoration: BoxDecoration(
          color: color,
          borderRadius: BorderRadius.circular(15),
          boxShadow: [const BoxShadow(color: Colors.black26, blurRadius: 4, offset: Offset(2, 2))],
        ),
        child: Column(
          children: [
            Text(value, style: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold, color: Colors.white)),
            Text(title, style: const TextStyle(fontSize: 14, color: Colors.white70)),
          ],
        ),
      ),
    );
  }

  Widget _buildMiniStat(String title, String value) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12),
      decoration: BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.circular(8),
        border: Border.all(color: Colors.black12),
      ),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(title, style: const TextStyle(fontSize: 12, color: Colors.grey)),
          Text(value, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
        ],
      ),
    );
  }
}

// ------------------- SIMULATED AD DIALOG -------------------

class MockAdDialog extends StatefulWidget {
  const MockAdDialog({super.key});

  @override
  State<MockAdDialog> createState() => _MockAdDialogState();
}

class _MockAdDialogState extends State<MockAdDialog> {
  int countdown = 3;

  @override
  void initState() {
    super.initState();
    Timer.periodic(const Duration(seconds: 1), (timer) {
      if (countdown == 0) {
        timer.cancel();
        Navigator.pop(context); // Close Ad
      } else {
        setState(() => countdown--);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Dialog(
      backgroundColor: Colors.black,
      insetPadding: EdgeInsets.zero,
      child: SizedBox(
        width: double.infinity,
        height: double.infinity,
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.ad_units, color: Colors.white, size: 60),
            const SizedBox(height: 20),
            const Text("INTERSTITIAL AD", style: TextStyle(color: Colors.white, fontSize: 24, fontWeight: FontWeight.bold)),
            const SizedBox(height: 10),
            Text("Closing in $countdown seconds...", style: const TextStyle(color: Colors.white70)),
            const SizedBox(height: 40),
            if (countdown == 0)
              IconButton(onPressed: () => Navigator.pop(context), icon: const Icon(Icons.close, color: Colors.white, size: 40))
          ],
        ),
      ),
    );
  }
}
