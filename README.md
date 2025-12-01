
AI Calculator
=============

This is a single-file Python Tkinter app that provides a calculator with optional voice I/O and an AI assistant.

Requirements:
- Python 3.8+
- Recommended packages: SpeechRecognition, pyttsx3, openai, pyaudio

Install (example):
    pip install SpeechRecognition pyttsx3 openai pyaudio

Notes:
- If you don't have a microphone or TTS engine available, the app will disable voice features.
- To enable the AI assistant, set the environment variable OPENAI_API_KEY before running.

Run:
    python ai_calculator.py
