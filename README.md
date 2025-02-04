import time
import requests
import csv
import sqlite3
import threading
import random
import tkinter as tk
from tkinter import messagebox, ttk
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from fake_useragent import UserAgent
from stem.control import Controller
from stem import Signal
import telebot

# Configuration du bot Telegram pour recevoir des notifications
TELEGRAM_BOT_TOKEN = "8054927329:AAFDcwA-REasSRwb_6sMIb0FSU"
TELEGRAM_CHAT_ID = "5394117357"
bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)

# Initialisation de la base de données pour stocker les annonces
def init_db():
    conn = sqlite3.connect("vinted_bot.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS annonces (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            titre TEXT,
            prix TEXT,
            lien TEXT UNIQUE
        )
    """)
    conn.commit()
    conn.close()

# Fonction pour envoyer une notification Telegram
def send_telegram_message(message):
    bot.send_message(TELEGRAM_CHAT_ID, message)

# Fonction pour relancer TOR et changer d'IP pour masquer l'activité
def change_ip():
    with Controller.from_port(port=9051) as controller:
        controller.authenticate(password="your_password")
        controller.signal(Signal.NEWNYM)

# Génération d'un User-Agent aléatoire pour simuler un navigateur différent à chaque requête
def get_random_user_agent():
    ua = UserAgent()
    return ua.random

# Fonction pour récupérer les annonces en fonction d'un mot-clé de recherche avec filtres
def scrape_vinted(search_query, min_price=None, max_price=None, min_quantity=1):
    options = webdriver.ChromeOptions()
    options.add_argument("--headless")
    options.add_argument(f"user-agent={get_random_user_agent()}")
    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
    
    driver.get("https://www.vinted.fr")
    time.sleep(random.uniform(2, 5))
    
    search_box = driver.find_element(By.NAME, "search_text")
    search_box.send_keys(search_query)
    search_box.send_keys(Keys.RETURN)
    time.sleep(random.uniform(3, 6))
    
    items = driver.find_elements(By.CLASS_NAME, "feed-grid__item")
    results = []
    
    for item in items[:10]:
        try:
            title = item.find_element(By.CLASS_NAME, "web_ui__Text").text
            price = item.find_element(By.CLASS_NAME, "web_ui__Text--bold").text.replace("€", "").strip()
            link = item.find_element(By.TAG_NAME, "a").get_attribute("href")
            quantity = title.lower().count(search_query.lower())
            
            if (min_price is None or float(price) >= min_price) and (max_price is None or float(price) <= max_price) and quantity >= min_quantity:
                results.append((title, price, link))
                send_telegram_message(f"🔔 Nouvelle annonce trouvée : {title} - {price}€\n{link}")
                if quantity >= min_quantity:
                    auto_buy(driver, link)
        except:
            continue
    
    driver.quit()
    save_to_db(results)
    return results

# Fonction d'auto-achat
def auto_buy(driver, item_link):
    driver.get(item_link)
    time.sleep(random.uniform(2, 5))
    try:
        buy_button = driver.find_element(By.XPATH, "//button[contains(text(), 'Acheter')]")
        buy_button.click()
        time.sleep(random.uniform(3, 6))
        confirm_button = driver.find_element(By.XPATH, "//button[contains(text(), 'Confirmer')]")
        confirm_button.click()
        send_telegram_message(f"✅ Article acheté automatiquement : {item_link}")
    except:
        send_telegram_message(f"⚠️ Impossible d'acheter automatiquement : {item_link}")

# Fonction principale qui lance les différentes fonctionnalités
def run_bot():
    search_query = entry_search.get()
    email = entry_email.get()
    password = entry_password.get()
    min_price = float(entry_min_price.get()) if entry_min_price.get() else None
    max_price = float(entry_max_price.get()) if entry_max_price.get() else None
    min_quantity = int(entry_min_quantity.get()) if entry_min_quantity.get() else 1
    
    if not search_query or not email or not password:
        messagebox.showerror("Erreur", "Veuillez remplir tous les champs")
        return
    
    threading.Thread(target=scrape_vinted, args=(search_query, min_price, max_price, min_quantity)).start()
    threading.Thread(target=refresh_ads, args=(email, password)).start()
    threading.Thread(target=alert_new_items, args=(f"https://www.vinted.fr/vetements?search_text={search_query}",)).start()

# Initialisation de la base de données
init_db()

# Interface Graphique améliorée pour ajouter les filtres
top = tk.Tk()
top.title("Bot Vinted Ultra Sécurisé")

tk.Label(top, text="Mot-clé de recherche :").pack()
entry_search = tk.Entry(top)
entry_search.pack()

tk.Label(top, text="Email Vinted :").pack()
entry_email = tk.Entry(top)
entry_email.pack()

tk.Label(top, text="Mot de passe Vinted :").pack()
entry_password = tk.Entry(top, show="*")
entry_password.pack()

tk.Label(top, text="Prix minimum (€) :").pack()
entry_min_price = tk.Entry(top)
entry_min_price.pack()

tk.Label(top, text="Prix maximum (€) :").pack()
entry_max_price = tk.Entry(top)
entry_max_price.pack()

tk.Label(top, text="Quantité minimale dans l'annonce :").pack()
entry_min_quantity = tk.Entry(top)
entry_min_quantity.pack()

tk.Button(top, text="Lancer le Bot", command=run_bot).pack()

top.mainloop()
