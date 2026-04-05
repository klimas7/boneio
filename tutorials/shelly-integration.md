---
layout: default
title: Integracja BoneIO z Shelly
---

## Studium przypadku

Podczas mojej ostatniej realizacji klient zdecydował nie integrować podświetlenia mebli kuchennych (oświetlenie LED) z BoneIO,
podczas gdy cała reszta oświetlenia domowego była sterowana przez BoneIO. 
Sygnalizowałem, że będzie to uciążliwe, szczególnie że w domu przewidziany był przycisk master wyłączający całe oświetlenie.

I rzeczywiście, po wdrożeniu klient szybko stwierdził, że to błąd. 
Problem polegał na tym, że kiedy wyłączano całe światło w domu za pomocą mastera, oświetlenie LED w kuchni pozostawało włączone. 
Oznaczało to konieczność przejścia przez cały salon, aby ręcznie wyłączyć podświetlenie w kuchni.

Zaproponowałem zastosowanie modułu dopuszkowego **Shelly 1 Mini Gen4**, który byłby sterowany bezpośrednio z BoneIO poprzez HTTP API. 
W ten sposób przyciski w domu (w tym master) mogą kontrolować moduły Shelly.

Ten opis pokazuje, jak BoneIO może być rozszerzany o dodatkowe urządzenia, dostępne w sieci lokalnej, bez konieczności całkowitej przebudowy systemu.

## Architektura

```
BoneIO (ESP32) ── HTTP ──► Shelly Device
     │                        
     └─Przyciski              
```

## Organizacja konfiguracji

Aby integracja była łatwa w utrzymaniu i mogły być wielokrotnie używane w różnych wdrożeniach, podzieliłem konfigurację na dwa pliki:

1. **`shelly_common.yaml`** - Uniwersalna logika wspólna dla wszystkich wdrożeń
2. **`shelly.yaml`** - Dane specyficzne dla konkretnego wdrożenia (adresy IP, identyfikatory)

### `shelly_common.yaml` - Logika wspólna

Plik zawiera wszystkie skrypty i konfigurację HTTP potrzebne do komunikacji z modułami Shelly.
Jest to plik **uniwersalny**, niezmieniający się między wdrożeniami.

```yaml
http_request:
  timeout: 2s

script:
  - id: shelly_on_off
    mode: parallel
    parameters:
      index: int
      onOff: bool
    then:
      - lambda: |-
          auto ip = id(shelly_ips)[index];
          ESP_LOGI("shelly_on_off", "%s %d", ip.c_str(), onOff);
      - http_request.get:
          url: !lambda |-
            auto ip = id(shelly_ips)[index];
            std::string url = "http://" + ip + "/rpc/Switch.Set?id=0&on=";
            url += (onOff ? "true" : "false");
            ESP_LOGI("shelly_url", "%s", url.c_str());
            return url;
          on_response:
            then:
              - lambda: |-
                  auto ip = id(shelly_ips)[index];
                  if (response->status_code == 200) {
                    id(shelly_state)[index] = onOff;
                    ESP_LOGI("shelly_on_off", "State set for %s to %d", ip.c_str(), onOff); 
                  } else {
                    ESP_LOGE("shelly_on_off", "Failed to set state for %s, status code: %d", ip.c_str(), response->status_code);
                  }

  - id: shelly_check_state
    mode: parallel
    parameters:
      index: int
    then:
      - lambda: |-
          auto ip = id(shelly_ips)[index]; 
          ESP_LOGI("shelly_check_state", "%s", ip.c_str());
      - http_request.get:
          url: !lambda |-
            auto ip = id(shelly_ips)[index];
            std::string url = "http://" + ip + "/rpc/Switch.GetStatus?id=0";
            return url;
          capture_response: true
          on_response:
            then:
              - lambda: |-
                  if (response->status_code == 200) {
                    bool out = false;
                    json::parse_json(body, [&](JsonObject root) -> bool {
                      out = root["output"];
                      return true;
                    });
                    id(shelly_state)[index] = out;
                  }
```

**Co ten plik realizuje:**

#### Sekcja `http_request`
- **Timeout: 2s** - maksymalny czas czekania na odpowiedź z Shelly

#### Skrypt `shelly_on_off` - Sterowanie przełącznikiem
**Parametry:**
- `index: int` - indeks Shelly w tablicy `shelly_ips` (0, 1, 2...)
- `onOff: bool` - stan: `true` = włącz, `false` = wyłącz

**Co robi:**
1. Pobiera adres IP z tablicy `id(shelly_ips)[index]`
2. Loguje akcję: `"shelly_on_off" "192.168.0.165" "1"`
3. Buduje URL do Shelly RPC API: `http://192.168.0.165/rpc/Switch.Set?id=0&on=true`
4. Wysyła HTTP GET do Shelly
5. Jeśli odpowiedź ma status 200, aktualizuje `id(shelly_state)[index]` na nową wartość

#### Skrypt `shelly_check_state` - Odczyt stanu
**Parametry:**
- `index: int` - indeks Shelly w tablicy

**Co robi:**
1. Pobiera adres IP Shelly
2. Loguje: `"shelly_check_state" "192.168.0.165"`
3. Wysyła HTTP GET: `http://192.168.0.165/rpc/Switch.GetStatus?id=0`
4. `capture_response: true` - pobiera całą odpowiedź w zmiennej `body`
5. Parsuje JSON z odpowiedzi - wyciąga pole `output`
6. Aktualizuje `id(shelly_state)[index]` z rzeczywistym stanem

### `shelly.yaml` - Dane specyficzne dla wdrożenia

Ten plik zawiera **dane lokalne** dla konkretnego domu/wdrożenia:

```yaml
esphome:
  platformio_options:
    build_flags:
      - -DSHELLY_KUCHNIA=0
      - -DSHELLY_WYSPA=1

globals:
  - id: shelly_ips
    type: std::array<std::string, 2>
    restore_value: no
    initial_value: '{"192.168.0.165","192.168.0.101"}'
  - id: shelly_state
    type: std::array<bool, 2>
    restore_value: no
    initial_value: '{false,false}'

packages:
  shelly_common: !include shelly_common.yaml
```

**Co tu się zawiera:**

**Build flags:**
- `SHELLY_KUCHNIA=0` - identyfikator (Shelly na pozycji 0 w tablicy sterownie kuchnia)
- `SHELLY_WYSPA=1` - identyfikator (Shelly na pozycji 1 sterownie wyspa)
- Mogą być użyte w kodzie

**Tablice globalne:**
- `shelly_ips` - tablica IP adresów Shelly (w tym domu są 2 urządzenia)
  - `shelly_ips[0]` = `192.168.0.165` (Shelly kuchnia)
  - `shelly_ips[1]` = `192.168.0.101` (Shelly wyspa)

> ⚠️ **Ważne:** Adresy IP Shelly są wpisane na stałe w konfiguracji, dlatego każde urządzenie Shelly **musi mieć stały adres IP** — najlepiej przez rezerwację DHCP na routerze lub konfigurację statycznego IP bezpośrednio w aplikacji Shelly. W przeciwnym razie po restarcie routera adresy mogą się zmienić i integracja przestanie działać.

- `shelly_state` - tablica przechowująca ostatni znany stan (włączony/wyłączony)
  - `shelly_state[0]` = stan Shelly nr 0
  - `shelly_state[1]` = stan Shelly nr 1

> ℹ️ **Inicjalizacja stanu po starcie:** Tablica `shelly_state` inicjalizuje się wartościami `false` — stan jest nieznany do momentu pierwszego wywołania `shelly_check_state`. Aby pobrać rzeczywisty stan od razu po uruchomieniu sterownika, można dodać wywołanie w sekcji `on_boot`:
> ```yaml
> esphome:
>   on_boot:
>     priority: -100
>     then:
>       - script.execute:
>           id: shelly_check_state
>           index: SHELLY_KUCHNIA
>       - script.execute:
>           id: shelly_check_state
>           index: SHELLY_WYSPA
> ```

**Include:**
- `shelly_common: !include shelly_common.yaml` - ładuje uniwersalną logikę

### Include w głównym pliku (`boneIO.yaml`)

W głównym pliku konfiguracji wystarczy jeden include:

```yaml
packages:
  shelly: !include shelly.yaml
```

Ten include automatycznie ładuje również `shelly_common.yaml` poprzez pakiet wewnątrz `shelly.yaml`.

## Korzyści takiego podziału

1. **Ponowne użycie kodu** - `shelly_common.yaml` może być identyczny w każdym wdrożeniu
2. **Łatwa zmiana** - IP adresy Shelly zmieniają się tylko w `shelly.yaml`
3. **Czytelność** - główny plik `boneIO.yaml` zawiera tylko jeden include zamiast całej logiki
4. **Skalowalność** - łatwo dodać nowe Shelly, rozszerzając tablicę w `shelly_ips`
5. **Maintenance** - wspólna logika jest w jednym miejscu, łatwa do aktualizacji

## Praktyczne zastosowanie
W pliku konfiguracyjnym `boneIO.yaml` (główny plik) jest skrypt `master_off` realizujący funkcję wyłączania całego oświetlenia w domu, w tym modułów Shelly:
```yaml
  - id: master_off
    mode: single
    then:
      - lambda: |-
          ESP_LOGI("master_off", "start");
          
          // Logika wyłączania inne światła w domu
          
          ESP_LOGI("master_off", "shelly off");
          id(shelly_on_off)->execute(SHELLY_KUCHNIA, false);
          id(shelly_on_off)->execute(SHELLY_WYSPA, false);
          
          ESP_LOGI("master_off", "end");
```

W pliku konfiguracyjnym `boneIO_dl.yaml` (dimmer LED) jest skrypt `salon` realizujący **maszynę stanów** sterującą LED-ami oraz modułami Shelly:
```yaml
  - id: salon
    parameters:
      sn: string #switch name
      pt: string #push type
    then:
      - lambda: |-
          ESP_LOGI("salon", "%s %s", sn.c_str(), pt.c_str());
          
          // L01A, L01B -> L01 --> Blenda
          // L02        -> L02 --> Led TV
          // S01        -> S01 --> Shelly kuchnia
          
          // Maszyna stanów z przejściami SP/LP/VLP
          
          // VLP lub LP - wyłącz wszystko
          if (((sn == "S02B_P" || sn == "S03B_P") && pt == "LP") || pt == "VLP") {
            id(L01A).turn_off().perform();
            id(L01B).turn_off().perform();
            id(L02).turn_off().perform();
            
            ESP_LOGI("salon", "shelly off");
            id(shelly_on_off)->execute(SHELLY_KUCHNIA, false);
            id(shelly_on_off)->execute(SHELLY_WYSPA, false);
          
            return;
          }
          
          // SP - maszyna stanów z warunkami
          if ((sn == "S02B_P" || sn == "S03B_P") && pt == "SP") {
            auto isOnL01 = id(L01A).current_values.is_on() || id(L01B).current_values.is_on();
            auto isOnL02 = id(L02).current_values.is_on();

            // Stan 1: Wszystko wyłączone -> włącz wszystko
            if ((!isOnL01 && !isOnL02) || (!isOnL01 && isOnL02)) {
              id(L01A).turn_on().perform();
              id(L01B).turn_on().perform();
              id(L02).turn_on().perform();
              return;
            }
            
            // Sprawdź stan Shelly (LED kuchni) - ważne!
            id(shelly_check_state)->execute(SHELLY_KUCHNIA);
            auto isOnS01 = id(shelly_state)[SHELLY_KUCHNIA];
            
            // Jeśli Shelly wyłączone - włącz je (tylko jeśli jest wyłączone!)
            if (!isOnS01) {
              id(shelly_on_off)->execute(SHELLY_KUCHNIA, true);
              return;
            }

            // Stan 2: L01 i L02 włączone -> wyłącz L02
            if (isOnL01 && isOnL02) {
              id(L01A).turn_on().perform();
              id(L01B).turn_on().perform();
              id(L02).turn_off().perform();
              return;
            }

            // Stan 3: L01 włączone, L02 wyłączone -> wyłącz L01, włącz L02
            if (isOnL01 && !isOnL02) {
              id(L01A).turn_off().perform();
              id(L01B).turn_off().perform();
              id(L02).turn_on().perform();
              return;
            }
          }
```

**Jak ta logika pracuje w praktyce:**

- `LP` oraz `VLP` realizują prostą akcję „wyłącz wszystko” — gaszone są lokalne światła `L01A`, `L01B`, `L02` oraz oba moduły Shelly.
- `SP` steruje przede wszystkim lokalnym oświetleniem w salonie, czyli kombinacją `L01` (blenda) i `L02` (LED TV).
- Dodatkowo w jednym z kroków skrypt sprawdza stan `SHELLY_KUCHNIA` i włącza go tylko wtedy, gdy moduł jest aktualnie wyłączony.

Można więc spojrzeć na to tak, że `SP` składa się z dwóch części:

1. **Przełączanie lokalnych świateł** — zmiana układu `L01` i `L02` w zależności od bieżącego stanu.
2. **Dodatkowy warunek dla kuchni** — jeśli `SHELLY_KUCHNIA` jest wyłączone, skrypt może je włączyć jako jeden z elementów scenariusza.

**Kluczowe momenty:**

1. **Odczyt stanu Shelly:** `id(shelly_check_state)->execute(SHELLY_KUCHNIA)` - pobiera rzeczywisty stan modułu.
2. **Sprawdzenie warunku:** `auto isOnS01 = id(shelly_state)[SHELLY_KUCHNIA]` - korzysta z odczytanej wartości.
3. **Warunkowe włączenie:** `if (!isOnS01)` - włącza Shelly **tylko wtedy, gdy jest wyłączone**.

Ten przykład pokazuje, jak za pomocą jednego przycisku można:
- Sterować wieloma LED-ami lokalnie
- Odczytać stan modułów Shelly via HTTP
- Implementować warunkową logikę (włącz Shelly, tylko jeśli jest wyłączone)
- Tworzyć maszyny stanów z wieloma przejściami

## API Shelly
Shelly 1 Mini Gen4, podstawowe operacje sterowania przełącznikiem są realizowane poprzez HTTP GET do endpointów RPC:

| Operacja | URL                                          |
|----------|----------------------------------------------|
| Włączenie | `GET http://IP/rpc/Switch.Set?id=0&on=true`  |
| Wyłączenie | `GET http://IP/rpc/Switch.Set?id=0&on=false` |
| Stan | `GET http://IP/rpc/Switch.GetStatus?id=0`    |

---

*Więcej szczegółów znajdziesz w dokumentacji ESPHome i Shelly API.*
