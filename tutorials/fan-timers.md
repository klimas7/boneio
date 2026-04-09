---
layout: default
title: Poradnik - Wentylator i timery
---

## Wstęp

W moim wdrożeniu wentylator łazienkowy (`W01`) nie jest uruchamiany "na sztywno" z jednego przycisku.
Sterowanie opieram o stan `LED_STATE`, który jest publikowany z dimmera łazienkowego i odbierany przez sterownik główny.
W trybie automatycznym wentylator włącza się po `1min`, a wyłącza po `2min` od zaniku sygnału.
Niezależnie od timerów mogę też sterować wentylatorem ręcznie z przycisku (bez czekania na opóźnienia).

## Skąd bierze się sygnał `LED_STATE`

W `boneIO_dl_02.yaml` skrypt `lazienka` publikuje stan logiczny:

- `id(LED_STATE).publish_state(true)` gdy oświetlenie strefy wejściowej jest aktywne,
- `id(LED_STATE).publish_state(false)` gdy ta strefa zostaje wyłączona.

Na sterowniku głównym (`boneIO_32_10.yaml`) stan ten jest odbierany przez `packet_transport` jako `remote_LED_STATE`.

Przykład publikacji stanu po stronie dimmera (`boneIO_dl_02.yaml`):

```yaml
- id: lazienka
  then:
	- lambda: |-
		if (sn == "S09_G" && pt == "SP") {
		  auto isLedOn = (id(L05).current_values.is_on()
					   || id(L07).current_values.is_on()
					   || id(L08).current_values.is_on());
		  id(LED_STATE).publish_state(!isLedOn);
		}
```

Pełniejszy przykład odbioru i reakcji na stan po stronie sterownika głównego (`boneIO_32_10.yaml`):

```yaml
- platform: packet_transport
  transport_id: transport
  provider: boneio-dr-8ch-03-7c8500
  id: remote_LED_STATE
  remote_id: LED_STATE
  type: data
  on_state_change:
	then:
	  - lambda: |-
		  auto ledState = x.has_value() ? ONOFF(x.value()) : "Unknown";
		  ESP_LOGI("remote_LED_STATE", "state=%s", ledState.c_str());

		  if (ledState == "ON") {
			id(wentylator_off)->stop();
			if (!id(fan_cycle_active)) {
			  id(fan_cycle_active) = true;
			  id(wentylator_on)->execute();
			}
		  } else if (ledState == "OFF") {
			if (id(fan_cycle_active)) {
			  id(wentylator_off)->execute();
			}
		  }
```

Jak działa ten fragment skryptu:

- najpierw odczytuję stan i loguję go, żeby było wiadomo, co faktycznie przyszło z dimmera,
- gdy przychodzi `ON`, zatrzymuję ewentualne wyłączanie (`wentylator_off`),
- dopiero potem sprawdzam `fan_cycle_active`; jeśli cykl nie jest aktywny, ustawiam flagę na `true` i uruchamiam `wentylator_on`,
- gdy przychodzi `OFF`, nie wyłączam wentylatora od razu, tylko startuję `wentylator_off` (opóźnione wyłączenie), ale tylko gdy cykl był aktywny.

Najważniejsza jest tutaj flaga `fan_cycle_active`.
To blokada przed wielokrotnym uruchamianiem tego samego cyklu przy kolejnych pakietach `ON`.
Dzięki temu nie nakładają się równoległe wywołania i logika pozostaje przewidywalna: jeden aktywny cykl start/stop wentylatora na raz.

## Sterowanie ręczne z przycisku

Timery odpowiadają tylko za tryb automatyczny oparty o `LED_STATE`.
Równolegle wentylator `W01` mogę w każdej chwili przełączyć ręcznie z przycisku - to sterowanie działa niezależnie od opóźnień `1min` i `2min`.

Fragment skryptu (`boneIO_32_10.yaml`, `script: lazienka`):

```yaml
if (sn == "S08_L" && pt == "LP") {
  id(W01).toggle().perform();
}
```

W praktyce `LP` na `S08_L` natychmiast przełącza `W01`, bez czekania na logikę timerów.

## Logika timerów

W `boneIO_32_10.yaml` działają dwa skrypty:

- `wentylator_on` - opóźnienie `1min`, potem `id(W01).turn_on()`,
- `wentylator_off` - opóźnienie `2min`, potem `id(W01).turn_off()`.

Dodatkowo:

- `wentylator_off` ma `mode: restart`, więc każde kolejne zdarzenie OFF resetuje odliczanie,
- flaga `fan_cycle_active` pilnuje, żeby nie uruchamiać równolegle wielu cykli,
- przejście `LED_STATE` na ON zatrzymuje trwające wyłączanie (`id(wentylator_off).stop()`).
- poza automatyką timerową wentylator może być w każdej chwili przełączony ręcznie z przycisku (niezależnie od aktywnego odliczania).

Pełny fragment skryptów timerów (`boneIO_32_10.yaml`):

```yaml
- id: wentylator_on
  mode: single
  then:
	- lambda: |-
		ESP_LOGI("wentylator", "on before delay");
	- delay: 1min
	- lambda: |-
		ESP_LOGI("wentylator", "on after delay");
		id(W01).turn_on().perform();

- id: wentylator_off
  mode: restart
  then:
	- lambda: |-
		ESP_LOGI("wentylator", "off before delay");
	- delay: 2min
	- lambda: |-
		ESP_LOGI("wentylator", "off after delay");
		id(W01).turn_off().perform();
		id(fan_cycle_active) = false;
```

## Efekt użytkowy

W praktyce:

- wentylator startuje po minucie od zapalenia światła wejściowego,
- wentylator wyłącza się 2 minuty po zgaszeniu,
- ręczne sterowanie z przycisku działa niezależnie od logiki timerów i jest dostępne od razu,
- krótkie zmiany stanu światła nie powodują chaotycznego włącz/wyłącz.

To daje spokojniejszą pracę instalacji i bardziej przewidywalne zachowanie w codziennym użyciu.

## Co dostosować u siebie

- opóźnienie startu (`1min`) i wyłączenia (`2min`),
- warunek publikacji `LED_STATE` po stronie dimmera,
- nazwy `provider` i `remote_id` w `packet_transport`.

