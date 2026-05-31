# NetIP32

**NetIP32** to w pełni autonomiczny, przenośny gadżet do bezprzewodowego audytu sieci Wi-Fi, który zbudowałem od zera na bazie mikrokontrolera ESP32. 

Zamiast kupować gotowe zabawki typu Flipper Zero, postanowiłem zakodzić własne narzędzie do network reconu. Efekt? Urządzenie mieści się w dłoni, automatycznie mapuje architekturę sieci, wykrywa aktywne maszyny i prześwietla je pod kątem otwartych portów. Całość obsługujesz za pomocą fizycznych przycisków, a wyniki śledzisz na żywo na małym ekranie OLED.

### Hardware

<img width="1840" height="1405" alt="netip32_build2" src="https://github.com/user-attachments/assets/51d783e2-91e6-4b5f-b573-59de789f62b5" />

* **Mózg operacji:** **ESP32** jako mikrokontroler, który w tym projekcie bierze całe przetwarzanie danych, obsługę stosu sieciowego TCP/IP i bezprzewodową łączność Wi-Fi.
* **Interfejs graficzny:** Monochromatyczny ekran **OLED 0.96" (128x64 px)** na sterowniku SSD1306. Wyświetla menu i wyniki skanowania. Spięty z ESP32 przez I2C na niestandardowych, ale stabilnych pinach: `SDA (GPIO 17)` oraz `SCL (GPIO 5)`.
* **Sterowanie:** 4 fizyczne przyciski wrzucone w tryb `INPUT_PULLUP`. Wykorzystałem wewnętrzne rezystory podciągające w ESP32, dzięki czemu nie potrzebowałem żadnych zewnętrznych oporników. 
  * `BTN_UP` (GPIO 23) – Nawigacja w górę.
  * `BTN_DOWN` (GPIO 22) – Nawigacja w dół.
  * `BTN_OK` (GPIO 4) – Odpalenie akcji / Zatwierdzenie opcji.
  * `BTN_BACK` (GPIO 16) – Powrót do menu lub przycisk przerywania, który natychmiast zrywa trwające skanowanie.
    
<img width="1840" height="1946" alt="netip32_build" src="https://github.com/user-attachments/assets/500fc0a3-cd58-4fb1-89e3-9a485886e618" />

### Software

Kod napisałem w C++ (środowisko Arduino IDE). Połączyłem niskopoziomowe biblioteki sieciowe ESP32 z frameworkiem graficznym od Adafruit (`Adafruit_GFX` i `Adafruit_SSD1306`). Żeby program nie zwalniał przy setkach danych, zarządzanie listami hostów postawiłem na dynamicznych kontenerach z biblioteki standardowej C++ (`std::vector` + szybkie przeszukiwanie przez `std::find`). 

Dodatkowo zaimplementowałem programowy *debouncing* przycisków – kod sam odfiltrowuje drgania mechaniczne styków, więc interfejs klika się idealnie płynnie.

### Handshake i Inicjalizacja Sieci

Zaraz po resecie ESP32 budzi ekran, po czym odpala tryb stacji Wi-Fi (`WiFi.begin`) i podłącza się do sieci lokalnej. Na ekranie leci animacja ładowania, a gdy dostanie się do sieci, skaner wyciąga z DHCP nasz adres IP oraz maskę podsieci, przygotowując grunt pod atak. 

<img width="1840" height="1346" alt="connecting" src="https://github.com/user-attachments/assets/7b53da16-ab78-4dc2-b1f9-0b592e987c89" />
<img width="1840" height="1630" alt="connected" src="https://github.com/user-attachments/assets/8490d30e-6752-479d-997f-f3b11c1b5d48" />

Pokaże się ekran powitalny z nazwą urządzenia. Po wybudzeniu ekranu dowolnym przyciskiem, przenoszę się do menu głównego.

<img width="1840" height="1459" alt="welcome" src="https://github.com/user-attachments/assets/2513843e-895f-425f-a986-c50172cfeafc" />
<img width="1840" height="1405" alt="main_menu" src="https://github.com/user-attachments/assets/6d588baf-9a1a-4f8b-badb-6bcf1b449529" />


### Szybki Rekonesans

Po kliknięciu **"Scan Network"** funkcja `performGeneralScan()` zaczyna mapować całą podsieć klasy C (od końcówki `.1` do `.254`).

<img width="1840" height="1460" alt="fast_scan" src="https://github.com/user-attachments/assets/aa6f67ed-ebb0-4b73-99a2-aa1d37c169a6" />

* **Algorytm Quick Alive Check:** Skanowanie wszystkich portów na każdym IP zajęłoby wieki, więc podszedłem do tego strategicznie: Skaner wysyła szybkie zapytanie `client.connect()` tylko na **12 najbardziej popularnych portach** (SSH, HTTP, HTTPS, RDP, FTP itp.). Jeśli maszyna odpowie na choćby jeden z nich, wiemy, że żyje. Skaner natychmiast przerywa skanowanie tego IP, oznacza hosta jako `[ALIVE]` i zapisuje go na listę celów.

### Skanowanie Celowane

Kiedy mamy już listę "żywych" maszyn, możemy dobrać się do nich bardziej szczegółowo.

<img width="1840" height="1665" alt="vectors" src="https://github.com/user-attachments/assets/1e80eb44-68c0-4006-8543-3de735f19072" />

Wybieram z menu konkretny **Wektor Skanowania**, a ESP32 bierze na celownik *tylko i wyłącznie* aktywne urządzenia, bombardując je zapytaniami o konkretne, specjalistyczne usługi:

| Wektor Skanowania | Przykładowe Porty w Bazie | Cel Audytu |

| **Web Services** | 80, 443, 3000, 5000, 8080, 9200 | Szukanie ukrytych serwerów HTTP, API i paneli admina. |

| **Remote Access** | 22, 23, 2222, 3389, 5900, 5985 | Wykrywanie otwartych furtek: SSH, Telnet, RDP czy VNC. |

| **File Shares** | 20, 21, 139, 445, 2049 | Lokalizowanie serwerów FTP i podatnych udziałów Samba/SMB. |

| **Databases** | 1433, 1521, 3306, 5432, 6379, 27017 | Namierzanie baz danych (MySQL, Postgres, MongoDB, Redis). |

| **Infrastructure** | 53, 67, 88, 161, 389, 5060 | Podglądanie usług sieciowych: DNS, LDAP, SNMP czy VoIP. |

| **DevOps / CI-CD** | 2375, 2376, 6443, 9092, 15672 | Prześwietlanie środowisk pod Dockerem, Kubernetesem czy RabbitMQ. |

| **Other / Custom** | Ponad 100 niestandardowych portów | Głęboka orka w poszukiwaniu nietypowych aplikacji i backdoorów. |

### Zarządzanie Prędkością

W menu **Settings** umożliwiłem ustawienie czasu oczekiwania na odpowiedź portu (`generalScanTimeout`) w przedziale od **10 ms do 100 ms**. Możemy tym sterować:

* **Tryb Agresywny (10-20 ms):** Skanuję sieć szybciej. Preferowane przy bardzo dobrym zasięgu WiFi.
* **Tryb Głęboki (50-100 ms):** Wolniejszy, ale niezawodny. Przydaje się w zaszumionym środowisku lub gdy sygnał jest słaby, dzięki temu nie pominiemy żadnego urządzenia (eliminujemy tzw. *False Negatives*).

<img width="1840" height="1236" alt="scan_speed" src="https://github.com/user-attachments/assets/5bcbfcc1-4989-46f4-83da-4b914d4d06dc" />

### Info

To sekcja z informacjami na temat NetIP32. W przyszłości będzie rozbudowana o więcej informacji, gdy dodam więcej funkcjonalności.

<img width="1840" height="1537" alt="info" src="https://github.com/user-attachments/assets/7e3719b5-b6df-4a70-abe3-544e89517b55" />

## Podsumowanie
Projekt dał mi masę frajdy i pozwolił genialnie połączyć zabawę elektroniką (I2C, GPIO, pull-up) z czystym network security i niskopoziomowym kodowaniem w C++. Idealny dowód na to, że potężne narzędzia pentesterskie można zmieścić w kieszeni spodni! Mam w planach rozbudować NetIP32 o dodatkowe funkcjonalności takie jak: wyszukiwanie publicznych sieci, banner grabbing, fingerprinting OS i panel webowy dla lepszej czytelności wyników.
