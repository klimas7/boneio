---
layout: default
title: "Poradnik - Oświetlenie zewnętrzne: zmierzch i harmonogram"
---

## Wstęp

W tej instalacji oświetlenie zewnętrzne (`O04`, `O11`) działa automatycznie na podstawie czujnika zmierzchu `CZ` oraz harmonogramu czasowego.
Nadal mogę sterować światłem z przycisków, ale logika pilnuje, aby nie psuć automatyki.

## Założenie działania

- po zmierzchu światła mogą zostać załączone,
- o 22:00 światła są wyłączane automatycznie,
- po restarcie sterownika wykonywana jest kontrola stanu, aby wrócić do poprawnego trybu.

## Realizacja w konfiguracji

W `boneIO_32_10.yaml` wykorzystuję trzy elementy:

1. `time.on_time` - codzienne wyłączenie o 22:00 (`light.turn_off: O04`, `light.turn_off: O11`),
2. `on_boot_time_check` - kontrola przy starcie (czas + stan `CZ`),
3. obsługa czujnika `CZ`:
   - `on_press`: uruchamia `on_boot_time_check`,
   - `on_release`: wyłącza `O04` i `O11`.

W praktyce to odpowiada typowemu podejściu "zmierzch + harmonogram" (jak cron), tylko zrealizowanemu natywnie w ESPHome przez `on_time`.

Przykład harmonogramu wyłączenia (`boneIO_32_10.yaml`):

```yaml
time:
  - id: !extend ds1307_time
    on_time:
      seconds: 0
      minutes: 0
      hours: 22
      then:
        - light.turn_off: O04
        - light.turn_off: O11
```

Przykład obsługi czujnika `CZ` (`boneIO_32_10.yaml`):

```yaml
- platform: gpio
  id: CZ
  on_press:
    then:
      - script.execute: on_boot_time_check
  on_release:
    then:
      - light.turn_off: O04
      - light.turn_off: O11
```

## Sterowanie ręczne i warunki

Przyciski `S01_L` oraz `S04_L` sterują oświetleniem zewnętrznym przez skrypt `zewnatrz`.
Skrypt sprawdza, czy światła powinny być aktywne (czas + zmierzch):

- jeśli są wyłączone - włącza je,
- jeśli są włączone - wyłączy je tylko wtedy, gdy instalacja jest poza oknem działania automatyki.

Dzięki temu ręczne sterowanie nie koliduje z główną logiką pracy po zmierzchu.

Przykład warunku dla sterowania ręcznego (`boneIO_32_10.yaml`, skrypt `zewnatrz`):

```yaml
if (sn == "S01_L" || sn == "S04_L") {
  auto isOn = id(O04).current_values.is_on() && id(O11).current_values.is_on();
  if (!isOn) {
    id(O04).turn_on().perform();
    id(O11).turn_on().perform();
  } else {
    auto time = id(ds1307_time).now();
    auto shouldBeOn = (time.hour >= 12 && time.hour < 22 && id(CZ).state);
    if (!shouldBeOn) {
      id(O04).turn_off().perform();
      id(O11).turn_off().perform();
    }
  }
}
```

## Dodatkowy obwód zewnętrzny

W tym samym skrypcie (`zewnatrz`) obsługuję też gniazda zewnętrzne `GZ01`, `GZ02`, `GZ03` z przycisków `S01_P` i `S04_P`.
Działają grupowo: jeżeli którekolwiek gniazdo jest włączone - wyłączam wszystkie, a jeśli wszystkie są wyłączone - włączam wszystkie.

## Co dostosować u siebie

- godzinę wyłączenia (`22:00`),
- warunek okna czasowego w skrypcie (`h >= 12 && h < 22`),
- zachowanie czujnika `CZ` i mapowanie wyjść (`O04`, `O11`).


