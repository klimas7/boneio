## Założenia
Sterownik [BoneIO](https://boneio.eu/) został zaprojektowany jako autonomiczne urządzenie, które zgodnie z mottem projektu ma być "sercem" automatyki domowej. 
W założeniu cała logika sterowania powinna być realizowana lokalnie na sterowniku, 
bez konieczności przenoszenia jej do zewnętrznego systemu np. Home Assistant (HA). 

Jeśli HA będzie używany, jego rola ogranicza się wyłącznie do funkcji interfejsu użytkownika (UI) - pełni on funkcję panelu sterowania, 
ale nie przejmuje logiki automatyki.

I dokładnie taka filozofia przyświecała mojej ostatniej inwestycji, 
gdzie wszystkie automatyzacje zostały zrealizowane bezpośrednio na sterownikach BoneIO.
W związku z tym, w tym poradniku przedstawię kilka ciekawych przypadków wdrożeń automatyzacji bezpośrednio na sterownikach.

## Poradniki

Kilka porad, które mogą pomóc przy rzeczywistym wdrożeniu BoneIO:

- **[Zestaw doświadczalny](tutorials/test-setup.md)** - Tablica testowa do weryfikacji automatyzacji przed wdrożeniem
- **[Szablony w ESPHome](tutorials/template-guide.md)** - Użycie szablonów na przykładzie on-multi-click
- **[Integracja BoneIO z Shelly](tutorials/shelly-integration.md)** - Integracja BoneIO z Shelly poprzez HTTP API
- **[Master Off](tutorials/master-off.md)** - Implementacja funkcji Master Off dla całego oświetlenia
- **[Wentylator i timery](tutorials/fan-timers.md)** - Opóźnione załączanie i wyłączanie wentylatora
- **[LED w komunikacji i czujniki ruchu](tutorials/led-motion-sensors.md)** - Dwa stany jasności i różne timery dla ruchu oraz przycisków
- **[Oświetlenie zewnętrzne: zmierzch i harmonogram](tutorials/outdoor-lighting-dusk-cron.md)** - Sterowanie oparte o czujnik zmierzchu i harmonogram `on_time`

## Dokumentacja
- [BoneIO](https://boneio.eu/)
- [ESPHome Documentation](https://esphome.io/)
