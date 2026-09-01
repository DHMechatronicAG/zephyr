# Company Custom Kconfig Options — Upstream Deviations

Dieses Dokument listet alle `DH_*`-Kconfig-Optionen, die wir zusätzlich zu
Standard-Zephyr/NCS eingeführt haben. Jede Option hier lebt in der
`Kconfig.d&h mechatronic AG`-Datei.

**Zweck:** Nachvollziehbarkeit. Wer in einem Jahr fragt "warum verhält sich
unser Zephyr-Fork anders als Standard-Zephyr", findet hier die komplette
Liste mit Begründung, Risiko und Fundort — statt den Diff gegen Upstream
Zeile für Zeile durchsuchen zu müssen.

---

## Konvention

- Alle eigenen Optionen werden mit dem Präfix `DH_` versehen, damit sie
  beim Grep (`grep -r "CONFIG_DH_"`) sofort von Standard-Optionen
  unterscheidbar sind.
- Jede Option bekommt hier im Dokument einen eigenen Abschnitt nach dem
  Muster unten, **bevor** sie gemerged wird.

---

## Neue Option hinzufügen — Checkliste

1. `DH_`-Präfix verwenden.
2. Option in `Kconfig.d&h mechatronic AG` anlegen, `default n`.
3. Eigenen Abschnitt in diesem Dokument ergänzen (Problem, Lösung, Risiko,
   Merge-Hinweis).
4. Zeile in der [Optionsübersicht](#optionsübersicht)-Tabelle ergänzen.


## Optionsübersicht

| Option | Datei | Status | Default |
|---|---|---|---|
| [`DH_UART_NRFX_UARTE_ASYNC_CONTINUE_ON_ERROR`](#dh_uart_nrfx_uarte_async_continue_on_error) | `drivers/serial/uart_nrfx_uarte.c` | Aktiv | `n` |
| [`DH_GPIO_PCA953X_TCA9534`](#dh_gpio_pca953x_tca9534) | `drivers/gpio/gpio_pca953x.c` | Aktiv | `n` |

---

## `DH_UART_NRFX_UARTE_ASYNC_CONTINUE_ON_ERROR`

**Datei:** `drivers/serial/uart_nrfx_uarte.c`
**Eingeführt:** 2026-07-20
**Autor:** *Nikolai Köhler*

### Problem

Bei RS485/UART-Kommunikation mit dem PIC (9600 Baud) wurden intermittierend
Break- und Framing-Errors am **Ende** empfangener Frames beobachtet. Das
Standard-Verhalten der Zephyr Async-UART-API sieht bei einem
`UART_RX_STOPPED`-Event (ausgelöst durch `NRF_UARTE_EVENT_ERROR`) einen
vollständigen Teardown vor:

```
ERROR-Event → UART_RX_STOPPED (App-Callback)
            → STOPRX
            → UART_RX_BUF_RELEASED (pro Buffer, App-Callback)
            → UART_RX_DISABLED (App-Callback)
            → App muss uart_rx_enable() erneut aufrufen
```

Dieser Roundtrip durchläuft mehrere ISR-/Workqueue-/App-Callback-Hops. In der
Zeit, in der der Receiver dadurch effektiv "taub" ist, gehen alle auf der
Leitung ankommenden Bytes unwiederbringlich verloren — nicht verzögert,
nicht in einem späteren Event nachgeliefert, sondern komplett verschwunden.
Das erklärte, warum bei uns konsistent der **Tail** eines Frames fehlte:
der Error tritt gegen Ende der Übertragung auf, der Teardown frisst genau
die letzten Bytes.

### Lösung

Bei aktivierter Option wird das `ERROR`-Event im ISR-Handling nur noch
gecleart (inkl. `nrf_uarte_errorsrc_get_and_clear()`, um ein erneutes
Auslösen zu verhindern), **ohne** `STOPRX` zu triggern. Der Empfang läuft
über EasyDMA unterbrechungsfrei weiter. Das für den Error ursächliche Byte
kann dabei fehlerhaft im Buffer landen — der Rest des Frames bleibt jedoch
erhalten.

```c
#if IS_ENABLED(CONFIG_DH_UART_NRFX_UARTE_ASYNC_CONTINUE_ON_ERROR)
	nrf_uarte_event_clear(uarte, NRF_UARTE_EVENT_ERROR);
	nrf_uarte_errorsrc_get_and_clear(uarte);
#else
	/* ursprünglicher Teardown-Pfad (STOPRX etc.) */
#endif
```

### Voraussetzung auf Anwendungsebene

Diese Option verschiebt Fehlerkorrektur bewusst auf die Applikationsschicht:
das UART-Frame-Format **muss** über ein eigenes Prüfsummen-/Checksum-Byte
verfügen, damit ein durch den Error korrumpiertes Byte erkannt und der
betroffene Frame verworfen/erneut angefordert werden kann. Ohne
Frame-Checksumme sollte diese Option **nicht** aktiviert werden, da
korrupte Bytes sonst unbemerkt durchgereicht werden.

### Risiko / Caveats

- **Nicht gegen Nordic-Errata validiert.** Es wurde nicht abschließend
  geprüft, ob das Auslassen von `STOPRX` nach einem `ERROR`-Event für alle
  UARTE-Revisionen/Fehlerarten (Overrun, Parity, Framing, Break) sauber ist.
  Vor Einsatz auf neuer Hardware-Revision: nRF52833-Errata-Dokument auf
  Einträge zu UARTE + ERROR/STOPRX/ENDRX prüfen.
- Getestet primär im Kontext **Framing- und Break-Errors** bei 9600 Baud.
  Verhalten bei Overrun-Errors nicht gesondert verifiziert.
- Kein Soak-Test über sehr lange Laufzeit mit hoher Error-Rate durchgeführt.
  Bei Verdacht auf DMA-Pointer-Desync (Hänger, wiederholt falsch
  ausgerichtete Frames nach vielen Errors) diese Option als ersten
  Verdächtigen behandeln.
- `depends on UART_NRFX_UARTE_LEGACY_SHIM` — Option ist nur für den
  Legacy-UARTE-Shim-Treiber verfügbar; bei Wechsel auf einen anderen
  UARTE-Treiberpfad muss der Patch neu evaluiert/portiert werden.

### Root Cause (Kontext, nicht Teil des Fixes)

Die eigentliche Ursache der Break-/Framing-Errors ist vermutich elektrisch
-bedingt (keine 5V an der RX-Leitung), nicht
im UART-Treiber selbst. Diese Kconfig-Option behandelt die **Symptomatik**
(Datenverlust durch den Treiber-Teardown)

---

## `DH_GPIO_PCA953X_TCA9534`

**Dateien:** `drivers/gpio/gpio_pca953x.c`,
`drivers/gpio/Kconfig.pca953x`, `dts/bindings/gpio/ti,tca9534.yaml`

**Eingeführt:** 2026-09-01

**Autor:** *Nikolai Köhler*

### Problem

Der in diesem Fork vorhandene PCA953X-GPIO-Treiber unterstützt standardmäßig
nur Devicetree-Knoten mit `compatible = "ti,tca9538"`. Die D&H-Hardware
verwendet stattdessen vier TCA9534 mit den I2C-Adressen `0x20` bis `0x23`.
Ohne passende Bindings und Treiberinstanzen erzeugt Zephyr für diese Knoten
keine GPIO-Geräte.

### Lösung

Die Option aktiviert das zusätzliche Binding `ti,tca9534` und lässt den
PCA953X-Treiber für alle aktivierten TCA9534-Knoten Geräteinstanzen erzeugen.
Da TCA9534- und TCA9538-Instanzen gleichzeitig vorkommen können, verwendet
der gemeinsame Initialisierungscode node-basierte Devicetree-Makros statt
kompatibilitätslokaler Instanznummern.

```c
#if defined(CONFIG_DH_GPIO_PCA953X_TCA9534)
DT_FOREACH_STATUS_OKAY(ti_tca9534, GPIO_PCA953X_INIT)
#endif
DT_FOREACH_STATUS_OKAY(ti_tca9538, GPIO_PCA953X_INIT)
```

Die Anwendung muss die Erweiterung bewusst einschalten:

```conf
CONFIG_DH_GPIO_PCA953X_TCA9534=y
```

### Risiko / Caveats

- Der TCA9534 besitzt nicht alle erweiterten Register des TCA9538. Die
  Basisfunktionen für Richtung, Eingang und Ausgang verwenden die gemeinsamen
  Register `0x00`, `0x01` und `0x03`; TCA9538-spezifische Interrupt-Latch- und
  Interrupt-Mask-Funktionen dürfen für TCA9534-Knoten nicht vorausgesetzt
  werden.
- Der Interrupt-Ausgang des TCA9534 wurde in dieser Anwendung nicht angebunden
  oder validiert. Die vier Geräte werden ausschließlich per I2C abgefragt.
- Die Option ist absichtlich `default n`. Andere Anwendungen im Fork erhalten
  damit ohne ausdrückliche Aktivierung keine zusätzliche TCA9534-Unterstützung.

### Merge-Hinweis

Bei einem Rebase oder Versionswechsel zuerst prüfen, ob der Zielstand bereits
ein offizielles `ti,tca9534`-Binding und eine passende Treiberinstanziierung
enthält. In diesem Fall sollen Binding, Treiberänderung und diese Option
entfernt werden, statt parallele Implementierungen beizubehalten.
