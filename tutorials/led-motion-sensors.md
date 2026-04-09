---
layout: default
title: Poradnik - LED w komunikacji i czujniki ruchu
---

## Wstęp

W komunikacji zastosowałem jedną, prostą logikę dla czterech obwodów LED: `L09A`, `L09B`, `L10A`, `L10B`.
Sterowanie działa zarówno z przycisków, jak i z czujników ruchu, ale z innymi poziomami jasności i innymi timerami.

## Architektura sygnałów

Przyciski (`S10`, `S11_L`, `S09_D`) oraz czujniki ruchu (`CO_1`, `CO_2`) są podłączone do sterownika głównego.
Ich stan jest przekazywany przez `packet_transport` do sterownika LED `boneIO_dl_01.yaml`, gdzie działa skrypt `przedpokoj`.

W skrypcie rozróżniam dwa źródła zdarzeń:

- `sn: "SW"` - zdarzenie z przycisku,
- `sn: "CO"` - zdarzenie z czujnika ruchu.

Przykład odbioru zdarzeń z `packet_transport` (`boneIO_dl_01.yaml`):

```yaml
- platform: packet_transport
  id: remote_S10
  remote_id: S10
  on_press:
    then:
      - script.execute: { id: przedpokoj, sn: "SW", pt: "OP" }

- platform: packet_transport
  id: remote_CO_1
  remote_id: CO_1
  on_press:
    then:
      - script.execute: { id: przedpokoj, sn: "CO", pt: "OP" }
```

## Trzy stany oświetlenia

Stan przechowuję tekstowo w `led_state_hall`:

- `OFF` - światła wyłączone,
- `ON_15` - światło orientacyjne (15%),
- `ON_100` - pełne światło użytkowe (100%).

## Jak działa logika

Rdzeń logiki (`boneIO_dl_01.yaml`):

```yaml
- id: przedpokoj
  parameters:
    sn: string
    pt: string
  then:
    - lambda: |-
        auto led_state = id(led_state_hall).state;
        if (sn == "CO" && led_state == "OFF") {
          id(turn_on_led_hall)->execute(0.15);
          id(led_state_hall).publish_state("ON_15");
          id(turn_off_led_hall_after)->execute(90);
        }
        if (sn == "SW" && (led_state == "OFF" || led_state == "ON_15")) {
          id(turn_on_led_hall)->execute(1.0);
          id(led_state_hall).publish_state("ON_100");
          id(turn_off_led_hall_after)->execute(900);
        }
```

### Ruch (`CO_1` / `CO_2`)

Jeżeli jest `OFF`, skrypt:

1. włącza LED-y na 15% (`turn_on_led_hall(0.15)`),
2. ustawia stan `ON_15`,
3. uruchamia wygaszanie po 90 sekundach (`turn_off_led_hall_after(90)`).

Jeżeli już jest `ON_15`, skrypt tylko odświeża timer 90 sekund.
Jeżeli jest `ON_100`, ruch nic nie zmienia.

### Przycisk (`S10`, `S11_L`, `S09_D`)

Jeżeli jest `OFF` lub `ON_15`, skrypt:

1. zatrzymuje aktualne wygaszanie,
2. przełącza LED-y na 100% (`turn_on_led_hall(1.0)`),
3. ustawia stan `ON_100`,
4. uruchamia wygaszanie po 15 minutach (`turn_off_led_hall_after(900)`).

Jeżeli jest `ON_100`, kolejne naciśnięcie wyłącza światło.

Timer wygaszania (`mode: restart`) zapewniający reset odliczania (`boneIO_dl_01.yaml`):

```yaml
- id: turn_off_led_hall_after
  mode: restart
  parameters:
    delay_s: int
  then:
    - delay: !lambda return delay_s * 1000;
    - lambda: |-
        light::LightState* lights[] = { id(L09A), id(L09B), id(L10A), id(L10B)};
        for (auto *l : lights) {
          l->turn_off().perform();
        }
        id(led_state_hall).publish_state("OFF");
```

## Dlaczego to działa dobrze

- czujnik ruchu daje delikatne podświetlenie bez oślepiania,
- przycisk zawsze daje pełną jasność i dłuższy czas świecenia,
- `mode: restart` w skrypcie wygaszania resetuje licznik i zapobiega nakładaniu timerów.

## Co dostosować u siebie

- poziom jasności dla trybu ruchu (`0.15`),
- czas wygaszania dla ruchu (`90s`) i ręcznego włączenia (`900s`),
- mapowanie przycisków/czujników przekazywanych przez `packet_transport`.
