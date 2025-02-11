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
// final isListening = StateProvider<bool>((ref) => false); 


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
      // ref.read(isListening.notifier).state = true;
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
      // ref.read(isListening.notifier).state = false;
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
