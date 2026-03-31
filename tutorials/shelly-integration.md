---
layout: default
title: Integracja BoneIO z Shelly
---

# Integracja BoneIO z Shelly

## Wstęp

Ten tutorial pokazuje jak zintegrować sterownik BoneIO z urządzeniami Shelly poprzez HTTP API. Integracja pozwala na sterowanie przełącznikami Shelly bezpośrednio z poziomu BoneIO.

## Architektura

```
BoneIO (ESP32) ──HTTP──► Shelly Device
     │                        │
     └─Przyciski              └─Przełączniki
```

## Przykład konfiguracji

### 1. Konfiguracja adresów IP Shelly

```yaml
globals:
  - id: shelly_ips
    type: std::array<std::string, 2>
    restore_value: no
    initial_value: '{"10.10.1.120","10.10.1.63"}'
  - id: shelly_state
    type: std::array<bool, 2>
    restore_value: no
    initial_value: '{false,false}'
```

### 2. Konfiguracja HTTP Request

```yaml
http_request:
  timeout: 4s
```

### 3. Skrypt sterowania Shelly

```yaml
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
                  if (response->status_code == 200) {
                    id(shelly_state)[index] = onOff;
                  }
```

### 4. Skrypt sprawdzania stanu

```yaml
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

## Użycie w praktyce

### Sterowanie przyciskiem

```yaml
binary_sensor:
  - platform: gpio
    pin: GPIO22
    name: "Przycisk Shelly 1"
    on_press:
      then:
        - script.execute:
            id: shelly_on_off
            index: 0
            onOff: true
```

### Synchronizacja przy starcie

```yaml
on_boot:
  then:
    - script.execute:
        id: shelly_check_state
        index: 0
    - script.execute:
        id: shelly_check_state
        index: 1
```

## API Shelly

| Operacja | URL |
|----------|-----|
| Włączenie | `http://IP/rpc/Switch.Set?id=0&on=true` |
| Wyłączenie | `http://IP/rpc/Switch.Set?id=0&on=false` |
| Stan | `http://IP/rpc/Switch.GetStatus?id=0` |

## Przykład z życia

W mojej instalacji w sypialni używam Shelly do sterowania oświetleniem LED:

- Shelly 1 (10.10.1.120) - steruje LED TV
- Shelly 2 (10.10.1.63) - steruje LED łóżka

Przyciski na BoneIO wysyłają komendy HTTP do odpowiednich Shelly, które włączają/wyłączają diody LED.

## Rozwiązywanie problemów

### Shelly nie odpowiada
- Sprawdź czy Shelly jest zasilany
- Upewnij się, że adres IP jest poprawny
- Sprawdź połączenie: `ping 10.10.1.120`

### HTTP timeout
```yaml
http_request:
  timeout: 6s  # Zwiększ timeout
```

### Błąd JSON
Sprawdź odpowiedź Shelly w logach ESPHome:
```bash
esphome logs shelly.yaml
```

---

*Więcej szczegółów znajdziesz w dokumentacji ESPHome i Shelly API.*
