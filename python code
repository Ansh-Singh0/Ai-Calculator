"""
AI Calculator - ai_calculator.py
Features:
- GUI using Tkinter
- Voice input (SpeechRecognition) and voice output (pyttsx3)
- Basic & scientific calculator operations
- AI assistant (OpenAI API optional; reads OPENAI_API_KEY from environment)
- History (in-memory + save/load JSON)
- Customizable themes (Light / Dark / Blue) and font size
- Auto error handling with friendly messages and suggestions

How to run:
1. Install dependencies:
   pip install SpeechRecognition pyttsx3 openai pyaudio
   (On some systems, pyaudio is tricky to install — see platform docs)
2. (Optional) Set environment variable OPENAI_API_KEY for AI assistant features.
3. python ai_calculator.py

Notes:
- Voice input requires a working microphone. If no mic found, the button will be disabled.
- AI assistant uses OpenAI's completions API; it's optional and degrades gracefully if key not found.
"""

import os
import json
import math
import threading
from datetime import datetime
from functools import partial

import tkinter as tk
from tkinter import ttk, messagebox, filedialog

# Optional packages
try:
    import speech_recognition as sr
except Exception:
    sr = None

try:
    import pyttsx3
except Exception:
    pyttsx3 = None

try:
    import openai
except Exception:
    openai = None

# ----------------------- Helper utilities -----------------------

def safe_eval(expr):
    """Safely evaluate simple math expressions. Supports + - * / ** % (), math functions."""
    # Allowed names
    allowed_names = {k: getattr(math, k) for k in dir(math) if not k.startswith("_")}
    # Add builtins we allow
    allowed_names.update({
        'abs': abs,
        'round': round,
        'min': min,
        'max': max,
        'sqrt': math.sqrt,
    })
    # Replace accidental characters
    expr = expr.replace('^', '**')
    try:
        code = compile(expr, "<string>", "eval")
        for name in code.co_names:
            if name not in allowed_names:
                raise NameError(f"Use of '{name}' not allowed")
        return eval(code, {"__builtins__": {}}, allowed_names)
    except Exception as e:
        raise

def speak_async(text, engine):
    def _speak():
        try:
            engine.say(str(text))
            engine.runAndWait()
        except Exception:
            pass

    t = threading.Thread(target=_speak, daemon=True)
    t.start()

# ----------------------- Main App -----------------------

class AICalculator(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("AI Calculator")
        self.geometry("420x620")
        self.resizable(False, False)

        self.history = []  # list of dicts: {time, expr, result, error}
        self.history_file = "calc_history.json"

        # Theme settings
        self.themes = {
            'Light': {'bg': '#f7f7f7', 'fg': '#000', 'entry_bg': '#fff'},
            'Dark': {'bg': '#0f1724', 'fg': '#e6eef8', 'entry_bg': '#111827'},
            'Blue': {'bg': '#e8f1fb', 'fg': '#042b52', 'entry_bg': '#ffffff'},
        }
        self.current_theme = tk.StringVar(value='Dark')
        self.font_size = tk.IntVar(value=16)

        # Voice engines
        self.recognizer = sr.Recognizer() if sr else None
        self.mic_available = False
        if sr:
            try:
                with sr.Microphone() as mic:
                    self.mic_available = True
            except Exception:
                self.mic_available = False

        self.tts_engine = None
        if pyttsx3:
            try:
                self.tts_engine = pyttsx3.init()
            except Exception:
                self.tts_engine = None

        # AI assistant config
        self.openai_key = os.getenv('OPENAI_API_KEY')
        if openai and self.openai_key:
            openai.api_key = self.openai_key
        else:
            # degrade gracefully
            pass

        self._build_ui()
        self._apply_theme()

        # Load history if present
        self._load_history()

    def _build_ui(self):
        # Top frame: entry and controls
        top = ttk.Frame(self)
        top.pack(fill='x', padx=10, pady=8)

        self.entry_var = tk.StringVar()
        entry = tk.Entry(top, textvariable=self.entry_var, font=(None, self.font_size.get()), relief='flat')
        entry.pack(fill='x', ipady=10)
        entry.bind('<Return>', lambda e: self._on_equal())

        controls = ttk.Frame(self)
        controls.pack(fill='x', padx=0, pady=6)

        btn_calc = ttk.Button(controls, text='=', command=self._on_equal, width=6)
        btn_calc.pack(side='left')

        btn_clear = ttk.Button(controls, text='C', command=self._on_clear, width=6)
        btn_clear.pack(side='left', padx=4)

        btn_ans = ttk.Button(controls, text='Ans', command=self._on_ans, width=6)
        btn_ans.pack(side='left')

        # Voice buttons
        self.voice_rec_btn = ttk.Button(controls, text='🎙️ Speak', command=self._voice_input, width=8)
        self.voice_rec_btn.pack(side='left', padx=4)
        if not self.mic_available or not sr:
            self.voice_rec_btn.configure(state='disabled')

        self.voice_out_btn = ttk.Button(controls, text='🔊 Say', command=self._voice_output, width=8)
        self.voice_out_btn.pack(side='left', padx=4)
        if not self.tts_engine:
            self.voice_out_btn.configure(state='disabled')

        # Buttons frame
        btns = ttk.Frame(self)
        btns.pack(fill='both', expand=False, padx=10)

        button_layout = [
            ['7', '8', '9', '/'],
            ['4', '5', '6', '*'],
            ['1', '2', '3', '-'],
            ['0', '.', '(', '+'],
            [')', '^', '%', 'sqrt'],
            ['sin', 'cos', 'tan', 'log'],
            ['history', 'theme', 'ai', 'export']
        ]

        for r, row in enumerate(button_layout):
            rowf = ttk.Frame(btns)
            rowf.pack(fill='x', pady=2)
            for c, key in enumerate(row):
                cmd = partial(self._on_button, key)
                b = ttk.Button(rowf, text=key, command=cmd)
                b.pack(side='left', expand=True, fill='x', padx=2)

        # History / assistant frame
        bottom = ttk.Frame(self)
        bottom.pack(fill='both', expand=True, padx=10, pady=8)

        self.log_text = tk.Text(bottom, height=10, state='disabled', wrap='word')
        self.log_text.pack(fill='both', expand=True)

        # Status bar
        self.status_var = tk.StringVar(value='Ready')
        status = ttk.Label(self, textvariable=self.status_var, anchor='w')
        status.pack(fill='x')

    # ----------------------- Button handlers -----------------------

    def _on_button(self, key):
        if key == 'C':
            self._on_clear()
            return
        if key == '=':
            self._on_equal()
            return
        if key == 'Ans':
            self._on_ans()
            return
        if key == 'sqrt':
            self.entry_var.set(self.entry_var.get() + 'sqrt(')
            return
        if key in ('sin', 'cos', 'tan', 'log'):
            self.entry_var.set(self.entry_var.get() + f'{key}(')
            return
        if key == 'history':
            self._show_history()
            return
        if key == 'theme':
            self._open_theme_dialog()
            return
        if key == 'ai':
            self._ai_assistant()
            return
        if key == 'export':
            self._export_history()
            return
        # normal char
        self.entry_var.set(self.entry_var.get() + key)

    def _on_clear(self):
        self.entry_var.set('')
        self.status_var.set('Cleared')

    def _on_ans(self):
        if not self.history:
            self.status_var.set('No answers yet')
            return
        last = self.history[-1]
        if last.get('result') is not None:
            self.entry_var.set(self.entry_var.get() + str(last['result']))
            self.status_var.set('Inserted Ans')

    def _on_equal(self):
        expr = self.entry_var.get().strip()
        if not expr:
            self.status_var.set('Nothing to evaluate')
            return

        expr_for_eval = expr.replace('sqrt(', 'sqrt(')

        try:
            result = None
            result = safe_eval(expr_for_eval)
            self._add_history(expr, result)
            self.entry_var.set(str(result))
            self.status_var.set('OK')
            if self.tts_engine:
                speak_async(f"Result is {result}", self.tts_engine)
        except Exception as e:
            suggestion = self._generate_error_suggestion(expr, e)
            self._add_history(expr, None, error=str(e), suggestion=suggestion)
            self.status_var.set('Error: ' + str(e))
            messagebox.showerror('Evaluation Error', f'Could not evaluate expression:\\n{e}\\n\\nSuggestion: {suggestion}')

    def _generate_error_suggestion(self, expr, exc):
        msg = str(exc)
        if 'division by zero' in msg:
            return 'Check for division by zero in your expression.'
        if 'unexpected EOF' in msg or 'invalid syntax' in msg:
            return 'Check parentheses and syntax; try adding missing parentheses.'
        if 'Use of' in msg:
            return 'You used a function or name that is not allowed.'
        return 'Check the expression for typos or unsupported functions.'

    # ----------------------- History management -----------------------

    def _add_history(self, expr, result=None, error=None, suggestion=None):
        entry = {
            'time': datetime.now().isoformat(),
            'expr': expr,
            'result': result,
            'error': error,
            'suggestion': suggestion,
        }
        self.history.append(entry)
        self._append_log(entry)
        try:
            with open(self.history_file, 'w') as f:
                json.dump(self.history, f, indent=2)
        except Exception:
            pass

    def _append_log(self, entry):
        self.log_text.configure(state='normal')
        t = entry['time'][:19].replace('T', ' ')
        if entry['error']:
            self.log_text.insert('end', f"[{t}] {entry['expr']}  -> ERROR: {entry['error']}\\nSuggestion: {entry.get('suggestion')}\\n\\n")
        else:
            self.log_text.insert('end', f"[{t}] {entry['expr']}  = {entry['result']}\\n")
        self.log_text.see('end')
        self.log_text.configure(state='disabled')

    def _load_history(self):
        try:
            if os.path.exists(self.history_file):
                with open(self.history_file, 'r') as f:
                    self.history = json.load(f)
                for e in self.history:
                    self._append_log(e)
        except Exception:
            self.history = []

    def _show_history(self):
        win = tk.Toplevel(self)
        win.title('History')
        win.geometry('480x400')
        txt = tk.Text(win)
        txt.pack(fill='both', expand=True)
        for e in self.history:
            t = e['time'][:19].replace('T', ' ')
            if e.get('error'):
                txt.insert('end', f"[{t}] {e['expr']} -> ERROR: {e['error']}\\nSuggestion: {e.get('suggestion')}\\n\\n")
            else:
                txt.insert('end', f"[{t}] {e['expr']} = {e.get('result')}\\n")

        btn_frame = ttk.Frame(win)
        btn_frame.pack(fill='x')
        ttk.Button(btn_frame, text='Export JSON', command=self._export_history).pack(side='left', padx=4)
        ttk.Button(btn_frame, text='Clear History', command=self._clear_history).pack(side='left', padx=4)

    def _export_history(self):
        try:
            path = filedialog.asksaveasfilename(defaultextension='.json', filetypes=[('JSON', '*.json')])
            if not path:
                return
            with open(path, 'w') as f:
                json.dump(self.history, f, indent=2)
            messagebox.showinfo('Exported', f'History exported to {path}')
        except Exception as e:
            messagebox.showerror('Export Error', str(e))

    def _clear_history(self):
        if messagebox.askyesno('Confirm', 'Clear history?'):
            self.history = []
            try:
                if os.path.exists(self.history_file):
                    os.remove(self.history_file)
            except Exception:
                pass
            self.log_text.configure(state='normal')
            self.log_text.delete('1.0', 'end')
            self.log_text.configure(state='disabled')

    # ----------------------- Voice features -----------------------

    def _voice_input(self):
        if not self.recognizer or not self.mic_available:
            messagebox.showwarning('No microphone', 'Microphone not available')
            return

        def _listen():
            with sr.Microphone() as source:
                self.status_var.set('Listening...')
                try:
                    audio = self.recognizer.listen(source, timeout=5, phrase_time_limit=8)
                    self.status_var.set('Recognizing...')
                    text = self.recognizer.recognize_google(audio)
                    self.entry_var.set(self.entry_var.get() + text)
                    self.status_var.set('Recognized: ' + text)
                except sr.WaitTimeoutError:
                    self.status_var.set('Listening timed out')
                except sr.UnknownValueError:
                    self.status_var.set('Could not understand audio')
                except sr.RequestError:
                    self.status_var.set('Speech recognition service error')
                except Exception as e:
                    self.status_var.set('Voice input error')

        threading.Thread(target=_listen, daemon=True).start()

    def _voice_output(self):
        if not self.tts_engine:
            messagebox.showwarning('No TTS', 'Text-to-speech engine not available')
            return
        text = self.entry_var.get()
        if not text:
            messagebox.showinfo('Nothing', 'Enter something to speak')
            return
        speak_async(text, self.tts_engine)

    # ----------------------- Theme & Settings -----------------------

    def _open_theme_dialog(self):
        win = tk.Toplevel(self)
        win.title('Theme')
        ttk.Label(win, text='Choose theme:').pack(padx=8, pady=6)
        for t in self.themes.keys():
            ttk.Radiobutton(win, text=t, value=t, variable=self.current_theme, command=self._apply_theme).pack(anchor='w', padx=12)
        ttk.Label(win, text='Font size:').pack(padx=8, pady=6)
        ttk.Scale(win, from_=12, to=28, variable=self.font_size, orient='horizontal', command=lambda e: self._apply_theme()).pack(fill='x', padx=12, pady=8)

    def _apply_theme(self):
        th = self.themes.get(self.current_theme.get(), self.themes['Dark'])
        self.configure(bg=th['bg'])
        for child in self.winfo_children():
            try:
                child.configure(background=th['bg'])
            except Exception:
                pass
        # Entry and text
        for widget in self.winfo_children():
            for w in widget.winfo_children():
                try:
                    w.configure(background=th['entry_bg'], foreground=th['fg'], font=(None, self.font_size.get()))
                except Exception:
                    pass

    # ----------------------- AI assistant -----------------------

    def _ai_assistant(self):
        win = tk.Toplevel(self)
        win.title('AI Assistant')
        win.geometry('480x320')
        prompt_var = tk.StringVar()
        ttk.Label(win, text='Ask the assistant for help (explain expression, suggest fix):').pack(fill='x', padx=8, pady=6)
        prompt_entry = tk.Entry(win, textvariable=prompt_var)
        prompt_entry.pack(fill='x', padx=8, pady=4)

        result_box = tk.Text(win, height=10)
        result_box.pack(fill='both', expand=True, padx=8, pady=4)

        def _call_ai():
            q = prompt_var.get().strip()
            if not q:
                return
            result_box.delete('1.0', 'end')
            result_box.insert('end', 'Thinking...')
            threading.Thread(target=self._ai_thread, args=(q, result_box), daemon=True).start()

        ttk.Button(win, text='Ask', command=_call_ai).pack(pady=6)

    def _ai_thread(self, question, result_box):
        if not openai or not self.openai_key:
            result = 'OpenAI not configured. Set OPENAI_API_KEY environment variable to use the assistant.'
            self._set_text_threadsafe(result_box, result)
            return
        try:
            resp = openai.Completion.create(
                model='text-davinci-003',
                prompt=f"You are a helpful calculator assistant. User question: {question}\\nProvide a concise helpful answer, suggestions, and examples if appropriate.",
                max_tokens=300,
                temperature=0.2,
            )
            text = resp.choices[0].text.strip()
            self._set_text_threadsafe(result_box, text)
            if self.tts_engine:
                speak_async(text, self.tts_engine)
        except Exception as e:
            self._set_text_threadsafe(result_box, f'AI error: {e}')

    def _set_text_threadsafe(self, widget, text):
        def _set():
            widget.delete('1.0', 'end')
            widget.insert('end', text)
        self.after(1, _set)


# ----------------------- Entry point -----------------------

if __name__ == '__main__':
    app = AICalculator()
    app.mainloop()
