🔌 Conectarea pinilor în proiect

🟦 ESP32 către MAX485 (Modul RS-485)

GPIO17 (TX2) → DI (Data In) — transmite date de la ESP32 către MAX485

GPIO16 (RX2) → RO (Receiver Out) — primește date de la MAX485

GPIO4 → DE și RE (legate împreună) — controlează direcția comunicației (transmitere/recepție)

GND → GND — masă comună

3.3V sau 5V → VCC — alimentarea modulului MAX485 (în funcție de versiune)

Notă: dacă modulul MAX485 nu suportă logică pe 3.3V, se recomandă alimentare cu 5V și folosirea unui convertor de nivel logic pentru pinii TX/RX.

🟧 MAX485 către Janitza UMG104 (prin interfață RS-485)
A (MAX485) → A (Janitza)

B (MAX485) → B (Janitza)

GND (opțional) → GND (Janitza) — pentru referință comună

⚡️ Alimentarea generală
ESP32 este alimentat de la un modul Hi-Link 5V, de la port USB sau de la o sursă stabilizată externă

