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