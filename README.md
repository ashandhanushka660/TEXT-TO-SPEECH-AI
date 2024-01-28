# TEXT-TO-SPEECH-AI

import tkinter as tk
from tkinter import messagebox, StringVar, scrolledtext
import requests
import pyttsx3
from googletrans import Translator
import speech_recognition as sr
import sqlite3

class ChatApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Chat Application")
        self.root.geometry("800x400")
        self.root.configure(bg='#FFD700')

        self.translator = Translator()
        self.recognizer = sr.Recognizer()
        self.last_responses = []
        self.selected_response = StringVar()
        self.speech_in_progress = False  # Variable to track speech playback

        # Connect to chat_history.db for past conversations
        self.db_conn = sqlite3.connect('chat_history.db')
        self.create_table('chat_history')

        # Connect to permanent_records.db for permanent records
        self.permanent_conn = sqlite3.connect('permanent_records.db')
        self.create_table('permanent_records')

        frame_left = tk.Frame(root, bg='#FFD700')
        frame_left.place(relx=0.25, rely=0.5, anchor='center', relwidth=0.4, relheight=0.8)

        frame_center = tk.Frame(root, bg='#FFD700')
        frame_center.place(relx=0.5, rely=0.5, anchor='center', relwidth=0.2, relheight=0.8)

        self.create_left_frame(frame_left)
        self.create_center_frame(frame_center)

    def create_table(self, table_name):
        cursor = self.db_conn.cursor() if table_name == 'chat_history' else self.permanent_conn.cursor()

        cursor.execute(f'''
            CREATE TABLE IF NOT EXISTS {table_name} (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                response TEXT
            )
        ''')

        if table_name == 'chat_history':
            self.db_conn.commit()
        else:
            self.permanent_conn.commit()

    def load_past_conversations(self):
        cursor = self.db_conn.cursor()
        cursor.execute("SELECT response FROM chat_history")
        past_conversations = cursor.fetchall()
        self.last_responses = [conversation[0] for conversation in past_conversations]

        for response in self.last_responses:
            self.past_conversations_listbox.insert(tk.END, response)

    def insert_chat_history(self, response):
        cursor = self.db_conn.cursor()
        cursor.execute("INSERT INTO chat_history (response) VALUES (?)", (response,))
        self.db_conn.commit()

    def insert_permanent_record(self, response):
        cursor = self.permanent_conn.cursor()
        cursor.execute("INSERT INTO permanent_records (response) VALUES (?)", (response,))
        self.permanent_conn.commit()

    def create_left_frame(self, frame):
        past_conversations_label = tk.Label(frame, text="Past Conversations:", bg='#FFD700')
        past_conversations_label.pack()

        self.past_conversations_listbox = tk.Listbox(frame, selectmode=tk.SINGLE)
        self.past_conversations_listbox.pack(expand=True, fill=tk.BOTH)

        play_button = tk.Button(frame, text="Play Selected", command=self.play_selected, bg='#87CEFA')
        play_button.pack(pady=10)

        view_button = tk.Button(frame, text="View Full Conversation", command=self.view_full_conversation, bg='#87CEFA')
        view_button.pack(pady=10)

    def create_center_frame(self, frame):
        self.message_label = tk.Label(frame, text="Type your question:", bg='#FFD700')
        self.message_label.pack()

        self.message_entry = tk.Entry(frame, width=20)
        self.message_entry.pack()

        translate_button = tk.Button(frame, text="Translate", command=self.translate_message, bg='#90EE90')
        translate_button.pack(pady=10)

        stop_button = tk.Button(frame, text="Stop", command=self.stop_conversation, bg='#FF6347')
        stop_button.pack(pady=10)

        self.verbal_response_label = scrolledtext.ScrolledText(frame, wrap=tk.WORD, width=30, height=5)
        self.verbal_response_label.pack(pady=10)

    def translate_message(self):
        query = self.message_entry.get()
        if not query:
            messagebox.showinfo("Info", "Please enter a message.")
            return

        target_language = "English"  # Assuming English as the default language

        if target_language == "English":
            self.make_request(query)
        else:
            messagebox.showinfo("Info", "Invalid language selection.")

    def make_request(self, query):
        url = 'https://open-ai-chatgpt.p.rapidapi.com/ask'
        headers = {
            'content-type': 'application/json',
            'X-RapidAPI-Key': 'c2ab6cbaa2msh0bb1b8b6c00710fp11d487jsnff7223be3b55',
            'X-RapidAPI-Host': 'open-ai-chatgpt.p.rapidapi.com',
        }
        data = {'query': query}

        response = requests.post(url, json=data, headers=headers).json()
        verbal_response = response.get('response', '')
        self.verbal_response_label.delete('1.0', tk.END)
        self.verbal_response_label.insert(tk.INSERT, verbal_response)
        self.speak(verbal_response)
        self.last_responses.append(verbal_response)
        self.past_conversations_listbox.insert(tk.END, verbal_response)
        self.insert_chat_history(verbal_response)
        self.insert_permanent_record(verbal_response)

    def speak(self, text):
        self.speech_in_progress = True
        engine = pyttsx3.init()
        engine.say(text)
        engine.runAndWait()

    def stop_conversation(self):
        if self.speech_in_progress:
            if not self.past_conversations_listbox.selection_includes(self.past_conversations_listbox.nearest(
                    self.past_conversations_listbox.winfo_pointerx(), self.past_conversations_listbox.winfo_pointery())):
                pyttsx3.init().stop()
            else:
                self.speech_in_progress = False

    def play_selected(self):
        selected_index = self.past_conversations_listbox.curselection()
        if selected_index:
            selected_index = int(selected_index[0])
            selected_response = self.last_responses[selected_index]
            self.speak(selected_response)

    def view_full_conversation(self):
        selected_index = self.past_conversations_listbox.curselection()
        if selected_index:
            selected_index = int(selected_index[0])
            selected_response = self.last_responses[selected_index]
            self.show_full_conversation(selected_response)

    def show_full_conversation(self, conversation):
        full_window = tk.Toplevel(self.root)
        full_window.title("Full Conversation")
        full_window.geometry("400x200")

        full_label = tk.Label(full_window, text=conversation, bg='#FFD700')
        full_label.pack()

        play_button = tk.Button(full_window, text="Play Conversation", command=lambda: self.speak(conversation),
                                bg='#87CEFA')
        play_button.pack(pady=10)

if __name__ == "__main__":
    root = tk.Tk()
    app = ChatApp(root)
    app.load_past_conversations()
    root.mainloop()
