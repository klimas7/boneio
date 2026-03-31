## Założenia
Sterownik BoneIO jest autonomicznym urządzeniem, które zgodnie z mottem projektu ma być "sercem" automatyki domowej. 
W założeniu cała logika sterowania powinna być realizowana lokalnie na sterowniku, 
bez konieczności przenoszenia jej do zewnętrznego systemu np. Home Assistant (HA). 

Jeśli HA będzie używany, jego rola ogranicza się wyłącznie do funkcji interfejsu użytkownika (UI) - pełni on funkcję panelu sterowania, 
ale nie przejmuje logiki automatyki.

## Poradniki

Kilka porad, które mogą pomóc rzeczywistym wdrożeniu BoneIO:

- **[Poradnik - Szablony w ESPHome](tutorials/template-guide.md)** - Jak używać szablonów na przykładzie on-multi-click
- **[Poradnik - Integracja BoneIO z Shelly](tutorials/shelly-integration.md)** - Jak zintegrować BoneIO z urządzeniami Shelly poprzez HTTP API

## Dokumentacja
- [BoneIO](https://boneio.io/)
- [ESPHome Documentation](https://esphome.eu/)
