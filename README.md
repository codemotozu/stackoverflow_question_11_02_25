# stackoverflow_question_11_02_25


# prompt_screen.dart

```dart

import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/repositories/translation_repository.dart';
import '../providers/audio_recorder_provider.dart';
import '../providers/voice_command_provider.dart';
import '../widgets/voice_command_status_inficator.dart';

// Add state provider for listening state
final isListeningProvider = StateProvider<bool>((ref) => false);


class PromptScreen extends ConsumerStatefulWidget {
  const PromptScreen({super.key});

  @override
  ConsumerState<PromptScreen> createState() => _PromptScreenState();
}

class _PromptScreenState extends ConsumerState<PromptScreen> {
  late final TextEditingController _textController;
  late final AudioRecorder _recorder;

  @override
  void initState() {
    super.initState();
    _textController = TextEditingController();
    _recorder = ref.read(audioRecorderProvider);

    _initializeRecorder();
  }

  Future<void> _initializeRecorder() async {
    try {
      await _recorder.init();
    } catch (e) {
      debugPrint('Recorder init error: $e');
    }
  }

  void _handleVoiceCommand(VoiceCommandState state) {
    if (!mounted) return;
    setState(() {}); // Force UI update
    
    
    if (state.lastCommand?.toLowerCase() == "open") {
      _startVoiceRecording();
    } else if (state.lastCommand?.toLowerCase() == "stop") {
      _stopVoiceRecording();
    }
    
    if (state.error != null) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error!))
      );
    }
  }
  
  Future<void> _startVoiceRecording() async {
    try {
      await _recorder.startListening("open");
      ref.read(isListeningProvider.notifier).state = true;
      final currentState = ref.read(voiceCommandProvider);
      ref.read(voiceCommandProvider.notifier).state = currentState.copyWith(
        isListening: true
      );
    } catch (e) {
      debugPrint('Recording start error: $e');
    }
  }
  
  Future<void> _stopVoiceRecording() async {
    try {
      final path = await _recorder.stopListening();
      if (path != null) {
        final text = await ref.read(translationRepositoryProvider)
            .processAudioInput(path);
        _textController.text = text;
      }
    } catch (e) {
      debugPrint('Recording stop error: $e');
    } finally {
      ref.read(isListeningProvider.notifier).state = false;
      final currentState = ref.read(voiceCommandProvider);
      ref.read(voiceCommandProvider.notifier).state = currentState.copyWith(
        isListening: false
      );
    }
  }

  @override
  void dispose() {
    _recorder.dispose();
    _textController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final voiceState = ref.watch(voiceCommandProvider);

    // Add listener for voice commands
    ref.listen<VoiceCommandState>(
      voiceCommandProvider, 
      (_, state) {
        if (!mounted) return;
        _handleVoiceCommand(state);
      }
    );

    return Scaffold(
      backgroundColor: const Color(0xFF000000),
      appBar: CupertinoNavigationBar(
        backgroundColor: const Color(0xFF1C1C1E),
        border: null,
        middle: const Text('AI Chat Assistant',
            style: TextStyle(
                color: Colors.white,
                fontSize: 17,
                fontWeight: FontWeight.w600)),
        trailing: CupertinoButton(
          padding: EdgeInsets.zero,
          child: const Icon(CupertinoIcons.gear,
              color: CupertinoColors.systemGrey, size: 28),
          onPressed: () => Navigator.pushNamed(context, '/settings'),
        ),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            VoiceCommandStatusIndicator(
              isListening: voiceState.isListening,
            ),
            const SizedBox(height: 12),
            Expanded(
              child: Align(
                alignment: Alignment.topLeft,
                child: CupertinoTextField(
                  controller: _textController,
                  maxLines: null,
                  style: const TextStyle(color: Colors.white, fontSize: 17),
                  placeholder: 'write your prompt here',
                  placeholderStyle: const TextStyle(
                      color: CupertinoColors.placeholderText, fontSize: 17),
                  decoration: BoxDecoration(
                    color: const Color(0xFF2C2C2E),
                    borderRadius: BorderRadius.circular(12),
                    border: Border.all(
                      color: const Color(0xFF3A3A3C),
                      width: 0.5,
                    ),
                  ),
                  padding: const EdgeInsets.all(16),
                ),
              ),
            ),
            const SizedBox(height: 20),
            Row(
              children: [
                Expanded(
                  child: ElevatedButton(
                    onPressed: () {
                      if (_textController.text.isNotEmpty) {
                        Navigator.pushNamed(
                          context,
                          '/conversation',
                          arguments: _textController.text,
                        ).then((_) => _textController.clear());
                      }
                    },
                    style: ElevatedButton.styleFrom(
                      backgroundColor: const Color.fromARGB(255, 61, 62, 63),
                      minimumSize: const Size(double.infinity, 50),
                    ),
                    child: const Text('start conversation', 
                        style: TextStyle(color: Colors.white)),
                  ),
                ),
                const SizedBox(width: 16),
                Consumer(
                builder: (context, ref, child) {
                  final voiceState = ref.watch(voiceCommandProvider);
                  return ElevatedButton(
                    onPressed: () => _toggleRecording(voiceState.isListening),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: voiceState.isListening ? Colors.red : Colors.white,
                      shape: const CircleBorder(),
                      padding: const EdgeInsets.all(16),
                    ),
                    child: const Icon(Icons.mic, size: 28),
                  );
                },
              ),
              ],
            ),
          ],
        ),
      ),
    );
  }

  Future<void> _toggleRecording(bool isCurrentlyListening) async {
    if (isCurrentlyListening) {
      await _stopVoiceRecording();
    } else {
      await _startVoiceRecording();
    }
  }
}

```

# voice_command_provider.dart


```dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../domain/repositories/translation_repository.dart';
import 'audio_recorder_provider.dart';
import 'prompt_screen_provider.dart';

class VoiceCommandState {
  final bool isListening;
  final String? lastCommand;
  final String? error;
  final bool isProcessing; 

  VoiceCommandState({
    this.isListening = false,
    this.lastCommand,
    this.error,
    this.isProcessing = false,
  });

  
  VoiceCommandState copyWith({
    bool? isListening,
    String? lastCommand,
    String? error,
    bool? isProcessing,
  }) {
    return VoiceCommandState(
      isListening: isListening ?? this.isListening,
      lastCommand: lastCommand ?? this.lastCommand,
      error: error ?? this.error,
      isProcessing: isProcessing ?? this.isProcessing,
    );
  }
}

class VoiceCommandNotifier extends StateNotifier<VoiceCommandState> {
  final AudioRecorder _recorder;
  final TranslationRepository _repository;
  final Ref _ref;

  VoiceCommandNotifier(this._recorder, this._repository, this._ref)
      : super(VoiceCommandState());

  Future<void> processVoiceCommand(String command) async {
    try {
      final commandLower = command.toLowerCase();
      
      if (commandLower == "open") {
        // First update prompt screen state
        _ref.read(promptScreenProvider.notifier).setListening(true);
        
        // Start recording first
        try {
          await _recorder.startListening(command);
          // Only update state after successful start of listening
          state = state.copyWith(
            isListening: true,
            lastCommand: command,
            isProcessing: false
          );
        } catch (e) {
          // If recording fails, update both states accordingly
          _ref.read(promptScreenProvider.notifier).setListening(false);
          state = state.copyWith(
            isListening: false,
            error: e.toString(),
            isProcessing: false
          );
          throw e; // Re-throw to be caught by outer try-catch
        }
      } else if (commandLower == "stop") {
        if (state.isListening) {
          try {
            final audioPath = await _recorder.stopListening();
            _ref.read(promptScreenProvider.notifier).setListening(false);
            
            if (audioPath != null) {
              state = state.copyWith(isProcessing: true);
              final text = await _repository.processAudioInput(audioPath);
              _ref.read(promptScreenProvider.notifier).updateText(text);
              
              state = state.copyWith(
                isListening: false,
                lastCommand: text,
                isProcessing: false
              );
            } else {
              state = state.copyWith(
                isListening: false,
                error: "Failed to get audio path",
                isProcessing: false
              );
            }
          } catch (e) {
            state = state.copyWith(
              isListening: false,
              error: e.toString(),
              isProcessing: false
            );
          }
        }
      }
    } catch (e) {
      state = state.copyWith(
        isListening: false,
        error: e.toString(),
        isProcessing: false
      );
    }
  }

  Future<void> handleSpeechRecognition(String audioPath) async {
    try {
      final text = await _repository.processAudioInput(audioPath);
      if (text.toLowerCase() == "open") {
        await processVoiceCommand("open");
      } else if (text.toLowerCase() == "stop") {
        await processVoiceCommand("stop");
      }
    } catch (e) {
      state = state.copyWith(
        isListening: false,
        error: e.toString(),
        isProcessing: false
      );
    }
  }
}

final voiceCommandProvider = StateNotifierProvider<VoiceCommandNotifier, VoiceCommandState>((ref) {
  return VoiceCommandNotifier(
    ref.watch(audioRecorderProvider),
    ref.watch(translationRepositoryProvider),
    ref,
  );
});

```


# audio_recorder_provider.dart

```dart



import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_sound/flutter_sound.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';

// Add state provider for listening state
final isListeningProvider = StateProvider<bool>((ref) => false);

final audioRecorderProvider = Provider<AudioRecorder>((ref) => AudioRecorder(ref));

class AudioRecorder {
  final FlutterSoundRecorder _recorder = FlutterSoundRecorder();
  bool _isInitialized = false;
  String? _path;
  final Ref _ref;

  AudioRecorder(this._ref);

  bool get isListening => _ref.read(isListeningProvider);

  Future<void> init() async {
    if (!_isInitialized) {
      final status = await Permission.microphone.request();
      if (status != PermissionStatus.granted) {
        throw RecordingPermissionException('Microphone permission not granted');
      }
      await _recorder.openRecorder();
      _isInitialized = true;
    }
  }

  Future<void> startListening(String command) async {
    if (!_isInitialized) await init();
    
    if (command.toLowerCase() == "open") {
      try {
        final dir = await getTemporaryDirectory();
        _path = '${dir.path}/audio_${DateTime.now().millisecondsSinceEpoch}.aac';
        await _recorder.startRecorder(
          toFile: _path,
          codec: Codec.aacADTS,
        );
        _ref.read(isListeningProvider.notifier).state = true;
      } catch (e) {
        debugPrint('Error starting recording: $e');
      }
    }
  }

  Future<String?> stopListening() async {
    try {
      if (_recorder.isRecording) {
        await _recorder.stopRecorder();
        _ref.read(isListeningProvider.notifier).state = false;
        return _path;
      }
      return null;
    } catch (e) {
      debugPrint('Error stopping recording: $e');
      return null;
    }
  }

  Future<void> start() async {
    if (!_isInitialized) await init();
    try {
      final dir = await getTemporaryDirectory();
      _path = '${dir.path}/audio_${DateTime.now().millisecondsSinceEpoch}.aac';
      await _recorder.startRecorder(
        toFile: _path,
        codec: Codec.aacADTS,
      );
      _ref.read(isListeningProvider.notifier).state = true;
    } catch (e) {
      debugPrint('Error recording audio: $e');
    }
  }

  Future<String?> stop() async {
    try {
      if (_recorder.isRecording) {
        await _recorder.stopRecorder();
        _ref.read(isListeningProvider.notifier).state = false;
        return _path;
      }
      return null;
    } catch (e) {
      debugPrint('Error stopping recording: $e');
      return null;
    }
  }

  Future<bool> isRecording() async {
    return _recorder.isRecording;
  }

  Future<void> dispose() async {
    if (_isInitialized) {
      await _recorder.closeRecorder();
      _isInitialized = false;
    }
  }
}

```


