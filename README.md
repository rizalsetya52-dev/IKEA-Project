# IKEA-Project
Scraping By Selenium-Python
import re
import time
from datetime import datetime, timedelta
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from webdriver_manager.chrome import ChromeDriverManager


# 1. Fungsi konversi teks relatif ke format DD-MM-YYYY
def konversi_ke_tanggal(teks_relatif):
    if not teks_relatif:
        return ""

    teks = teks_relatif.lower().strip()
    sekarang = datetime.now()

    angka_cocok = re.search(r"\d+", teks)
    jumlah = int(angka_cocok.group()) if angka_cocok else 1

    if "menit" in teks or "jam" in teks:
        hasil = sekarang
    elif "kemarin" in teks:
        hasil = sekarang - timedelta(days=1)
    elif "hari" in teks:
        hasil = sekarang - timedelta(days=jumlah)
    elif "minggu" in teks:
        hasil = sekarang - timedelta(weeks=jumlah)
    elif "bulan" in teks:
        hasil = sekarang - timedelta(days=jumlah * 30)
    elif "tahun" in teks:
        hasil = sekarang - timedelta(days=jumlah * 365)
    else:
        hasil = sekarang

    return hasil.strftime("%d-%m-%Y")


# 2. Setup Selenium (pakai folder profil KHUSUS, terpisah dari Chrome harian)
options = webdriver.ChromeOptions()
options.add_argument("--start-maximized")
options.add_argument(r"user-data-dir=C:\Users\pc\ChromeAutomationProfile")
driver = webdriver.Chrome(
    service=Service(ChromeDriverManager().install()), options=options
)

url = "https://www.google.com/maps/place/IKEA+Kota+Baru+Parahyangan/@-6.8662773,107.4648684,17z/data=!4m8!3m7!1s0x2e68fb9c9120e42f:0x28252fe17424dae6!8m2!3d-6.8671833!4d107.4667654!9m1!1b1!16s%2Fg%2F11nnz9l5bq?entry=ttu&g_ep=EgoyMDI2MDkwOS4wIKXMDSoASAFQAw%3D%3D"
driver.get(url)
time.sleep(5)

# Klik otomatis ke tab "Ulasan"
try:
    tab_ulasan = driver.find_element(
        By.XPATH, "//button[.//div[text()='Ulasan']]"
    )
    tab_ulasan.click()
    time.sleep(3)
    print(">>> Berhasil klik tab Ulasan otomatis.")
except Exception:
    print(">>> Gagal klik otomatis, silakan klik tab 'Ulasan' manual sekarang...")
    time.sleep(10)

reviews, seen = [], set()
target_reviews = 1000

processed_count = 0
prev_card_count = 0
stagnant_rounds = 0
max_stagnant = 5
checkpoint_every = 100

while len(reviews) < target_reviews:
    cards = driver.find_elements(By.CSS_SELECTOR, "div.jftiEf")
    new_cards = cards[processed_count:]

    for c in new_cards:
        try:
            more_btn = c.find_elements(By.CSS_SELECTOR, "button.w8nwRe")
            if more_btn:
                more_btn[0].click()
        except Exception:
            pass

        user_el = c.find_elements(By.CLASS_NAME, "d4r55")
        review_el = c.find_elements(By.CLASS_NAME, "wiI7pd")
        rating_el = c.find_elements(By.CLASS_NAME, "kvMYJc")
        date_el = c.find_elements(By.CLASS_NAME, "rsqaWe")

        if not review_el or not review_el[0].text.strip():
            continue

        user = user_el[0].text.strip() if user_el else "Unknown"
        review = review_el[0].text.strip()

        rating_raw = rating_el[0].get_attribute("aria-label") if rating_el else ""
        rating_match = re.search(r"\d+", rating_raw)
        rating = int(rating_match.group()) if rating_match else None

        raw_date = date_el[0].text.strip() if date_el else ""
        tanggal = konversi_ke_tanggal(raw_date)

        if (user, review) not in seen:
            seen.add((user, review))
            reviews.append({
                "username": user,
                "komentar": review,
                "rating": rating,
                "tanggal": tanggal,
            })

            if len(reviews) % checkpoint_every == 0:
                pd.DataFrame(reviews).to_csv(
                    "ulasan_ikea_checkpoint.csv", index=False, encoding="utf-8-sig"
                )
                print(f"Checkpoint: {len(reviews)} data tersimpan sementara.")

            if len(reviews) >= target_reviews:
                break

    processed_count = len(cards)

    if len(cards) == prev_card_count:
        stagnant_rounds += 1
        if stagnant_rounds >= max_stagnant:
            print("Ulasan sudah habis / tidak ada data baru lagi, berhenti.")
            break
    else:
        stagnant_rounds = 0
    prev_card_count = len(cards)

    try:
        scrollable_div = driver.find_element(By.CSS_SELECTOR, "div.m6QErb[aria-label]")
        driver.execute_script(
            "arguments[0].scrollTop = arguments[0].scrollHeight", scrollable_div
        )
    except Exception:
        break
    time.sleep(2)

driver.quit()

df = pd.DataFrame(reviews)
df.to_csv("ulasan_ikea_final.csv", index=False, encoding="utf-8-sig")
print(f"Beres! {len(df)} data berhasil disimpan ke file 'ulasan_ikea_final.csv'.")
