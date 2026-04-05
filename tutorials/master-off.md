---
layout: default
title: Poradnik - Master Off
---

## Wstęp

Przycisk `Master Off` daje użytkownikowi jeden, szybki sposób na wygaszenie oświetlenia przy wyjściu z domu (ID `S11_L`).
W mojej ostatniej realizacji taki przycisk jest fizycznie przy drzwiach wyjściowych i uruchamia scenariusz wyłączenia całego oświetlenia.

## Kontekst tej inwestycji

Na tej instalacji pracują 3 sterowniki:

- `boneIO.yaml` - sterownik główny, odpowiedzialny za przyciski, czujniki i klasyczne obwody oświetleniowe
- `boneIO_dl_01.yaml` - sterownik LED
- `boneIO_dl_02.yaml` - sterownik LED

Przyciski są podłączone do sterownika głównego, a ich stan jest dystrybuowany do sterowników LED z wykorzystaniem `packet_transport` (UDP).

## Implementacja

### Dystrybucja stanu przycisku przez UDP

W sterowniku głównym publikuję wybrane wejścia (m.in. `S11_L`) do dwóch urządzeń:

```yaml
udp:

packet_transport:
  - id: transport
    platform: udp
    update_interval: 30s
    encryption: 'SUPER_SECRET_KEY'
    rolling_code_enable: false
    binary_sensors:
      - S11_L
      # ...inne wejścia...
```

Po stronie sterowników LED odbieram zdarzenie `S11_L` i wykorzystuję ten sam gest (`VLP`) do uruchomienia lokalnego `master_off`:

```yaml
- platform: packet_transport
  transport_id: transport
  provider: boneio-32-l-07-39d104
  id: remote_S11_L
  remote_id: S11_L
  type: data
  on_multi_click:
    - timing:
        - ON for at least 3s
      then:
        - script.execute:
            id: master_off
```

To daje spójne zachowanie: jedno długie przytrzymanie przy wyjściu i każdy sterownik gasi swoje obwody.

### Skrypt `master_off` w `boneIO.yaml`

```yaml
  - id: master_off
    mode: single
    then:
      - lambda: |-
          ESP_LOGI("master_off", "start");

          light::LightState* lights[] =
            { id(O01_A), id(O01_B), id(O02_A), id(O02_B), id(O03_A), id(O03_B),
              id(O06), id(O07), id(O08),
              id(O09), id(O10), id(O12)
            };

          for (auto *l : lights) {
            auto call = l->turn_off();
            call.perform();
          }

          ESP_LOGI("master_off", "shelly off");
          id(shelly_on_off)->execute(SHELLY_KUCHNIA, false);
          id(shelly_on_off)->execute(SHELLY_WYSPA, false);

          ESP_LOGI("master_off", "end");
```

Zastosowanie `lambda` i listy świateł/urządzeń `lights[]` daje bardzo zwarty i czytelny kod - jedna pętla obsługuje wszystkie obwody bez powielania tych samych akcji.

### Skrypt `master_off` w `boneIO_dl.yaml`

```yaml
  - id: master_off
    mode: single
    then:
      - lambda: |-
          ESP_LOGI("master_off", "start");

          light::LightState* lights[] =
            { id(L05), id(L06), id(L07), id(L08),
              id(L01A), id(L01B), id(L02)};

          for (auto *l : lights) {
            auto call = l->turn_off();
            call.perform();
          }

          ESP_LOGI("master_off", "end");
```

### Wywołanie z przycisku na sterowniku głównym (`S11_L`, VLP)

```yaml
binary_sensor:
  - platform: gpio
    name: 'Przedpokoj wejscie do domu - Lewa'
    id: S11_L
    pin:
      pcf8574: pcf_inputs_28to35_menu
      number: 0
      inverted: true
    on_multi_click:
      - timing:
          - ON for at least 3s
        then:
          - logger.log: "S11_L VLP -> przedpokoj wejscie lewa"
          - script.execute:
              id: master_off
```

### Co dostosować u siebie

- listę świateł w tablicy `lights[]`
- wywołania Shelly (`SHELLY_KUCHNIA`, `SHELLY_WYSPA`)
- przycisk i czas VLP (`ON for at least 3s`)
- listę wejść publikowanych przez `packet_transport` 

---

Po pełniejszy opis integracji Shelly zajrzyj do [Integracja BoneIO z Shelly](shelly-integration.md).<br />
Opis szablonów znajdziesz w [Szablony w ESPHome](template-guide.md).
