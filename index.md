## Założenia
Sterownik BoneIO jest autonomicznym urządzeniem, które zgodnie z mottem projektu ma być "sercem" automatyki domowej. 
W założeniu cała logika sterowania powinna być realizowana lokalnie na sterowniku, 
bez konieczności przenoszenia jej do zewnętrznego systemu np. Home Assistant (HA). 

Jeśli HA będzie używany, jego rola ogranicza się wyłącznie do funkcji interfejsu użytkownika (UI) - pełni on funkcję panelu sterowania, 
ale nie przejmuje logiki automatyki.

## Poradniki

Kilka porad, które mogą pomóc przy rzeczywistym wdrożeniu BoneIO:

- **[Zestaw doświadczalny](tutorials/test-setup.md)** - Tablica testowa do weryfikacji automatyzacji przed wdrożeniem
- **[Szablony w ESPHome](tutorials/template-guide.md)** - Użycie szablonów na przykładzie on-multi-click
- **[Integracja BoneIO z Shelly](tutorials/shelly-integration.md)** - Integracja BoneIO z Shelly poprzez HTTP API
- **[Master Off](tutorials/master-off.md)** - Implementacja funkcji Master Off dla całego oświetlenia

## Dokumentacja
- [BoneIO](https://boneio.eu/)
- [ESPHome Documentation](https://esphome.io/)
