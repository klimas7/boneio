---
layout: default
title: Poradnik - Szablony w ESPHome
---

# Poradnik - Szablony w ESPHome

## Architektura obsługi przycisków

W moim przypadku przyciski podłączone do BoneIO mają dokładnie takie samo zachowanie.
To znaczy, każdy z nich reaguje na trzy zdarzenia: krótkie naciśnięcie SP (< 1s), długie naciśnięcie LP (< 3s) oraz bardzo długie naciśnięcie VLP (> 3s).
W reakcji na takie zdarzenie wywoływany jest skrypt, który otrzymuje informacje o tym, który przycisk został naciśnięty oraz jaki typ zdarzenia miał miejsce.

- `sn` - nazwa/identyfikator przycisku
- `pt` - typ zdarzenia (SP/LP/VLP)

Okazało się, że definicja każdego przycisku jest niemal identyczna, różni się tylko nazwą przycisku i skryptem,
który jest wywoływany. Bardzo mocno zaciemniło to konfigurację, ponieważ mamy wiele powtórzeń tej samej logiki detekcji SP/LP/VLP dla każdych przycisków.

## Użycie template'ów w detekcji wielokrotnych kliknięć

Rozwiązaniem tego problemu mogą być **szablony** (templates) w ESPHome.
Zamiast powtarzać logikę detekcji (Short Push, Long Push, Very Long Push) dla każdego przycisku, zdefiniowałem sobie **szablon** w pliku `on-multi-click.yaml`:
(W tym samym katalogu co główny plik konfiguracyjny `boneio.yaml`)

```yaml
# on-multi-click.yaml - szablon do wielokrotnych kliknięć
- timing:
    - ON for at least 3s
  then:
    - logger.log: "${switch_name} VLP"
    - script.execute:
        id: ${script}
        sn: "${switch_name}"
        pt: "VLP"
- timing:
    - ON for at least 1s
  then:
    - logger.log: "${switch_name} LP"
    - script.execute:
        id: ${script}
        sn: "${switch_name}"
        pt: "LP"
- timing:
    - ON for at most 1s
  then:
    - logger.log: "${switch_name} SP"
    - script.execute:
        id: ${script}
        sn: "${switch_name}"
        pt: "SP"
```

## Jak używać template'u

W definicji czujnika binarnego (przycisku) wystarczy odwołać się do szablonu z odpowiednimi parametrami:

```yaml
binary_sensor:
  - platform: gpio
    name: 'Salon, przy oknie tarasowym: Prawa'
    id: TS1_A
    pin:
      pcf8574: pcf_inputs_1to14
      number: 0
      inverted: true
    on_multi_click: !include { file: on-multi-click.yaml, vars: { switch_name: "TS1_A", script: "salon" } }
```

**Zmienne szablonu:**
- `${switch_name}` - nazwa przycisku (ID czujnika) używana w logowaniu i jako parametr skryptu
- `${script}` - ID skryptu, który ma być wykonany (np. `test`, `salon`, `sypialnia`)

## Korzyści

1. **Brak powtórzeń** - jedna definicja dla wszystkich przycisków
2. **Jedność** - wszystkie przyciski mają jednolite zachowanie
3. **Łatwa zmiana** - modyfikacja logiki multi-click w jednym miejscu

## Typy zdarzenia

- **SP (Short Push)** - ≤ 1 sekunda
- **LP (Long Push)** - 1-3 sekundy  
- **VLP (Very Long Push)** - ≥ 3 sekundy

Każde zdarzenie trafia do skryptu jako parametr `pt`, dzięki czemu jeden skrypt może obsługiwać różne akcje w zależności od sposobu naciśnięcia przycisku.

## Definicja skryptu

Skrypt przyjmuje parametry `sn` (nazwa przycisku) i `pt` (typ naciśnięcia/zdarzenia). Każdy skrypt definiujemy w sekcji `script:`.
Oto przykłady takich skryptów:

```yaml
script:
  - id: salon
    parameters:
      sn: string  # switch name - nazwa przycisku
      pt: string  # push type - typ naciśnięcia (SP/LP/VLP)
    then:
      - lambda: |-
          ESP_LOGI("salon", "%s %s", sn.c_str(), pt.c_str());
          id(O01_A).toggle().perform();
          id(O02_A).toggle().perform();
          id(O03_A).toggle().perform();

  - id: lazienka
    parameters:
      sn: string
      pt: string
    then:
      - lambda: |-
          ESP_LOGI("lazienka", "%s %s", sn.c_str(), pt.c_str());
          if ((sn == "S09_G" || sn == "S08_P") && pt == "LP") {
            id(O09).turn_off().perform();
          }
          if (sn == "S08_P" && pt == "SP") {
            id(O09).toggle().perform();  
          }
          if (sn == "S08_L" && pt == "LP") {
            id(W01).toggle().perform();
          }
```

### Struktura skryptu

- `id` - identyfikator skryptu (musi być zgodny z `script:` w szablonie `on_multi_click`)
- `parameters` - definiuje parametry przyjmowane przez skrypt (zawsze `sn` i `pt`)
- `then` - akcje wykonywane w lambda, mają dostęp do zmiennych `sn` (C++ string) i `pt`
- `ESP_LOGI()` - logowanie dla celów debugowania
- `id(O01_A).toggle()` - sterowanie światłami (ID światła)

Przeniesienie logiki do skryptu pozwala pisać logikę (przynajmniej mi) bardziej czytelnie.
Użycie C++ w `lambda` daje pełną swobodę w pisaniu warunków i logiki sterowania,
a parametry `sn` i `pt` pozwalają na uniwersalność skryptu dla różnych przycisków i typów naciśnięć.

---

Więcej szczegółów znajdziesz w dokumentacji ESPHome