import os
import subprocess
import requests
import smtplib
import speech_recognition as sr
import pyttsx3
from email.message import EmailMessage
from decouple import config
import wikipedia
import logging
from datetime import datetime
from transformers import pipeline  
import tensorflow as tf


logging.basicConfig(filename='ester_log.txt', level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Load environment variables
USER = config('USER', default='Poornesh') 
BOTNAME = config('BOTNAME', default='Ester') 
USER_ALIAS = config('USER_ALIAS', default='Boss')  
BOT_ALIAS = config('BOT_ALIAS', default='Darling')  
EMAIL = config('EMAIL')
PASSWORD = config('PASSWORD')
OPENWEATHER_APP_ID = config('OPENWEATHER_APP_ID')
TMDB_API_KEY = config('TMDB_API_KEY')
NEWS_API_KEY = config('NEWS_API_KEY')

# Initialize the text-to-speech engine
engine = pyttsx3.init()

# Initialize the text generation model
try:
    chatbot = pipeline("text-generation", model="microsoft/DialoGPT-medium")
except Exception as e:
    logging.error(f"Error initializing the chatbot: {e}")
    raise RuntimeError("Failed to load the model. Please ensure TensorFlow or PyTorch is installed.")

# Function to speak text
def speak(text):
    engine.say(text)
    engine.runAndWait()

# Function to greet based on the time of day
def greet_user():
    current_hour = datetime.now().hour
    if current_hour < 12:
        greeting = "Good morning"
    elif 12 <= current_hour < 18:
        greeting = "Good afternoon"
    else:
        greeting = "Good evening"
    speak(f"{greeting} {USER_ALIAS}, How can I assist you today?")

# Function to tell the current time
def tell_time():
    now = datetime.now()
    current_time = now.strftime("%H:%M")
    speak(f"The current time is {current_time} boss.")

# Function to send a WhatsApp message
def send_whatsapp_message(contact, message):
    speak(f"Sending message to {contact}: {message}")
    subprocess.call(["open", f"https://wa.me/{contact}?text={message}"])

# Function to get weather report
def get_weather_report(city):
    response = requests.get(f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={OPENWEATHER_APP_ID}&units=metric")
    data = response.json()
    if data.get("cod") != 200:  # Check if the response code is not 200
        return f"Error: {data.get('message', 'Unable to fetch weather data.')}"
    
    weather = data["weather"][0]["description"]
    temperature = data["main"]["temp"]
    return f"The weather in {city} is currently {weather} with a temperature of {temperature}°C boss."

# Function to send an email
def send_email(receiver_address, subject, message):
    email = EmailMessage()
    email['To'] = receiver_address
    email['Subject'] = subject
    email['From'] = EMAIL
    email.set_content(message)
    with smtplib.SMTP("smtp.gmail.com", 587) as s:
        s.starttls()
        s.login(EMAIL, PASSWORD)
        s.send_message(email)

# Function to get the latest news
def get_latest_news():
    response = requests.get(f"https://newsapi.org/v2/top-headlines?country=us&apiKey={NEWS_API_KEY}")
    articles = response.json()["articles"]
    return [article["title"] for article in articles[:5]]

# Function to search Wikipedia
def search_on_wikipedia(query):
    return wikipedia.summary(query, sentences=2)

# Function to open an application using launch command
def start_application(app_name):
    app_name = app_name.lower()
    try:
        subprocess.call(["open", "-a", app_name])
        logging.info(f"Opened application: {app_name}")
    except Exception as e:
        print(f"Could not open '{app_name}'. Error: {e}")
        logging.error(f"Error opening application '{app_name}': {e}")

# Function to close an application
def close_application(app_name):
    app_name = app_name.lower()
    try:
        subprocess.call(["pkill", app_name])
        logging.info(f"Closed application: {app_name}")
    except Exception as e: 
        print(f"Could not close '{app_name}'. Error: {e}")
        logging.error(f"Error closing application '{app_name}': {e}")

# Function to create a folder
def create_folder(folder_name):
    try:
        os.makedirs(folder_name, exist_ok=True)
        logging.info(f"Created folder: {folder_name} boss")
    except Exception as e:
        print(f"Could not create folder '{folder_name}'. Error: {e}")
        logging.error(f"Error creating folder '{folder_name}' boss: {e}")

# Function to delete a file
def delete_file(file_name):
    try:
        os.remove(file_name)
        logging.info(f"Deleted file: {file_name} boss")
    except Exception as e:
        print(f"Could not delete '{file_name}'. Error: {e}")
        logging.error(f"Error deleting file '{file_name}': {e}")

# Function to search on Spotlight
def search_on_spotlight(query):
    try:
        subprocess.call(["open", "-a", "Spotlight", query])
        logging.info(f"Searched on Spotlight for: {query}")
    except Exception as e:
        print(f"Could not search on Spotlight for '{query}'. Error: {e}")
        logging.error(f"Error searching on Spotlight for '{query}': {e}")

# Function to search on Safari
def search_on_safari(query):
    try:
        subprocess.call(["open", f"https://www.google.com/search?q={query}"])
        logging.info(f"Searched on Safari for: {query}")
    except Exception as e:
        print(f"Could not search on Safari for '{query}'. Error: {e}")
        logging.error(f"Error searching on Safari for '{query}': {e}")

# Function to get AI response
def get_ai_response(user_input):
    try:
        response = chatbot(user_input, max_length=100, num_return_sequences=1)
        return response[0]['generated_text'] 
    except Exception as e:
        logging.error(f"Error getting AI response: {e}")
        return "I'm sorry, I couldn't generate a response."

# Main function to listen for commands
def main():
    recognizer = sr.Recognizer()
    greet_user()  # Greet the user based on the time of day
    tell_time()   # Tell the current time

    is_paused = False  # Variable to track the pause state

    while True:
        if not is_paused:  # Only listen for commands if not paused
            try:
                with sr.Microphone() as source:
                    print("Listening...")
                    audio = recognizer.listen(source)
                    command = recognizer.recognize_google(audio).lower()
                    print(f"You said: {command} boss")

                    if "pause the chat" in command:
                        is_paused = True
                        speak("Voice assistant paused boss. Say 'resume' to continue.")
                    
                    elif "resume" in command:
                        is_paused = False
                        speak("Voice assistant resumed boss.")

                    elif "weather" in command:
                        city = command.split("in")[-1].strip()
                        weather_report = get_weather_report(city)
                        speak(weather_report)

                    elif "email" in command:
                        speak("Who do you want to send the email to boss?")
                        with sr.Microphone() as source:
                            audio = recognizer.listen(source)
                            receiver = recognizer.recognize_google(audio)
                        speak("What is the subject boss?")
                        with sr.Microphone() as source:
                            audio = recognizer.listen(source)
                            subject = recognizer.recognize_google(audio)
                        speak("What is the message boss?")
                        with sr.Microphone() as source:
                            audio = recognizer.listen(source)
                            message = recognizer.recognize_google(audio)
                        send_email(receiver, subject, message)
                        speak("Email sent boss.")

                    elif "news" in command:
                        news = get_latest_news()
                        speak("Here are the latest news headlines boss:")
                        for article in news:
                            speak(article)

                    elif "wikipedia" in command:
                        query = command.split("about")[-1].strip()
                        summary = search_on_wikipedia(query)
                        speak(summary)

                    elif "search spotlight" in command:
                        query = command.split("search spotlight")[-1].strip()
                        search_on_spotlight(query)

                    elif "search safari" in command:
                        query = command.split("search safari")[-1].strip()
                        search_on_safari(query)

                    elif "launch" in command:
                        app_name = command.split("launch")[-1].strip()
                        start_application(app_name)

                    elif "close" in command:
                        app_name = command.split("close")[-1].strip()
                        close_application(app_name)

                    elif "create folder" in command:
                        folder_name = command.split("create folder")[-1].strip()
                        create_folder(folder_name)
                        speak(f"Folder '{folder_name}' created boss.")

                    elif "delete file" in command:
                        file_name = command.split("delete file")[-1].strip()
                        delete_file(file_name)
                        speak(f"File '{file_name}' deleted boss.")

                    elif "send whatsapp message" in command:
                        contact = command.split("to")[-1].split("message")[0].strip()
                        message = command.split("message")[-1].strip()
                        send_whatsapp_message(contact, message)

                    elif "tell me about" in command:
                        topic = command.split("about")[-1].strip()
                        info = search_on_wikipedia(topic)
                        speak(info)

                    elif "exit darling" in command:
                        speak("Goodbye, have a nice day boss!")
                        break

                    else:
                        ai_response = get_ai_response(command)
                        speak(ai_response)

            except sr.UnknownValueError:
                logging.error("Speech Recognition could not understand audio.")
                speak("Sorry, I could not understand the audio boss.")
            except sr.RequestError as e:
                logging.error(f"Could not request results from the speech recognition service: {e}")
                speak("Could not request results from the speech recognition service boss.")
            except Exception as e:
                logging.error(f"An error occurred: {e}")
                speak("An error occurred. Please try again boss.")
       
        else:
            # If paused, wait for a command to resume
            try:
                with sr.Microphone() as source:
                    print("Listening for resume command...")
                    audio = recognizer.listen(source)
                    command = recognizer.recognize_google(audio).lower()
                    if "resume" in command:
                        is_paused = False
                        speak("Voice assistant resumed.")
            except sr.UnknownValueError:
                pass  # Ignore if nothing is heard
            except sr.RequestError as e:
                logging.error(f"Could not request results from the speech recognition service: {e}")
                speak("Could not request results from the speech recognition service boss.")
            except Exception as e:
                logging.error(f"An error occurred while waiting to resume: {e}")
                speak(f"An error occurred while waiting to resume: {e}")  # Print the error to the console

if __name__ == "__main__":
    main()
