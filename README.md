# a-dictation-file
pip install PyPDF2
import os
import json
import time
import threading
import queue
import PyPDF2
import pyttsx3
import pyaudio
from vosk import Model, KaldiRecognizer

# --- Configuration ---
PDF_FILE_PATH = 'your_novel.pdf'
STARTING_PAGE = 0 
VOSK_MODEL_PATH = "vosk-model-en-us-0.22"

class VoiceJacket:
    def __init__(self, model_path):
        self.engine = pyttsx3.init()
        self.model = Model(model_path)
        self.recognizer = KaldiRecognizer(self.model, 16000)
        
        # Control States
        self.is_running = True
        self.is_paused = False
        self.last_sentence = ""
        self.command_queue = queue.Queue()

    def extract_text(self, pdf_path, page_num):
        if not os.path.exists(pdf_path):
            return None
        with open(pdf_path, 'rb') as f:
            reader = PyPDF2.PdfReader(f)
            if page_num < len(reader.pages):
                return reader.pages[page_num].extract_text()
        return None

    def speak(self, text):
        """Internal helper to speak without blocking the whole system."""
        self.engine.say(text)
        self.engine.runAndWait()

    def tts_worker(self, full_text):
        """The 'Voice' of the Jacket."""
        sentences = [s.strip() + "." for s in full_text.split('.') if s.strip()]
        
        for sentence in sentences:
            while self.is_paused and self.is_running:
                time.sleep(0.1) # Wait while paused
            
            if not self.is_running:
                break

            self.last_sentence = sentence
            print(f"Reading: {sentence}")
            self.speak(sentence)
            time.sleep(0.2) # Breathable gap for commands

    def listen_loop(self):
        """The 'Ears' of the Jacket."""
        p = pyaudio.PyAudio()
        stream = p.open(format=pyaudio.paInt16, channels=1, rate=16000, 
                        input=True, frames_per_buffer=8000)
        stream.start_stream()

        print("\n--- Listening for commands (Pause, Resume, Repeat, Stop) ---")
        
        try:
            while self.is_running:
                data = stream.read(4000, exception_on_overflow=False)
                if self.recognizer.AcceptWaveform(data):
                    result = json.loads(self.recognizer.Result())
                    command = result.get('text', '')

                    if command:
                        self.handle_command(command)
        finally:
            stream.stop_stream()
            stream.close()
            p.terminate()

    def handle_command(self, command):
        print(f"User said: {command}")
        
        if 'pause' in command or 'stop' in command:
            self.is_paused = True
            print("|| Paused")
        
        elif 'resume' in command or 'continue' in command or 'play' in command:
            self.is_paused = False
            print(">> Resuming")
            
        elif 'repeat' in command:
            print("<< Repeating...")
            threading.Thread(target=self.speak, args=(self.last_sentence,)).start()

        elif 'exit' in command:
            self.is_running = False

# --- Execution ---
if __name__ == "__main__":
    jacket = VoiceJacket(VOSK_MODEL_PATH)
    text = jacket.extract_text(PDF_FILE_PATH, STARTING_PAGE)

    if text:
        # Start the TTS in a background thread
        reader_thread = threading.Thread(target=jacket.tts_worker, args=(text,))
        reader_thread.start()

        # Keep the main thread alive for listening
        jacket.listen_loop()
    else:
        print("Could not load PDF. Check the file path!")
