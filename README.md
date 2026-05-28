# THZ504-HA-Dynamic-Cooling


Hier ist die Zusammenfassung der **Version 0.6.3**.

## Grundfunktion

Das Package steuert die Kühlung der THZ504 nicht direkt über Verdichter oder Vorlauf, sondern über die **Kühlfreigabe** und die **Tag-/Nacht-Zieltemperaturen** der THZ:

```yaml
switch.esp32_thz_controller_kuehlmode
climate.esp32_thz_controller_heating_day
climate.esp32_thz_controller_heating_night
```

Die THZ-interne Regelung bleibt führend für:

```text
Verdichter
Vorlauf-/Rücklaufregelung
Taupunktschutz
Pumpenlogik
interne Kühlhysterese
```

Home Assistant entscheidet also primär:

```text
Soll Kühlung jetzt freigegeben werden?
Soll sie weiterlaufen?
Soll sie sicherheitshalber gestoppt werden?
Welche Zieltemperatur soll an die THZ übergeben werden?
```

---

# Wie die Dynamik funktioniert

## 1. Tag-/Nachtlogik

Die Automation unterscheidet zwischen Tag und Nacht über UI-Helfer:

```yaml
input_datetime.thz504_cooling_day_start_time
input_datetime.thz504_cooling_day_end_time
```

Default:

```text
Tag: 06:00–21:00
Nacht: 21:00–06:00
```

Tagsüber gelten niedrigere Komfortgrenzen und PV-orientierte Freigaben. Nachts wird konservativer gearbeitet: höhere Starttemperatur, höherer Batterie-SOC, weniger erlaubter Netzbezug und strengere Feuchte-/Taupunktgrenzen.

---

## 2. Temperaturbedarf

Die Hauptgröße ist:

```yaml
sensor.temperatur_haus_kuhlung_gewichtung
```

Diese wird über den Mapping-Helper eingebunden:

```yaml
input_text.thz504_entity_indoor_temp_weighted
```

Die Automation verwendet zusätzlich den ungewichteten Mittelwert:

```yaml
sensor.temperatur_haus_mittel_alle
```

Der Kühlbedarf wird über `binary_sensor.thz504_cooling_demand` ermittelt.

Tagsüber gilt standardmäßig:

```text
Start: 23,2 °C
Stop: 22,0 °C
```

Nachts:

```text
Start: 23,8 °C
Stop: 22,8 °C
```

Wenn die Wetterprognose heiß ist, wird die dynamische Starttemperatur tagsüber abgesenkt:

```yaml
sensor.thz504_cooling_dynamic_start_temperature
```

Beispiel:

```text
normaler Tag-Start: 23,2 °C
Wetter-Vorkühlung aktiv
Offset: 0,4 K
dynamischer Start: 22,8 °C
```

Damit beginnt die Kühlung früher, wenn heiße Tage bevorstehen.

---

## 3. Wetter-Vorkühlung

Das Package ruft stündlich ab:

```yaml
weather.get_forecasts
```

für:

```yaml
weather.ieppel31
```

Daraus entstehen Forecast-Sensoren:

```yaml
sensor.thz504_weather_forecast_today_temperature
sensor.thz504_weather_forecast_tomorrow_temperature
sensor.thz504_weather_forecast_max_temperature_next_3_days
sensor.thz504_weather_forecast_max_precipitation_probability_next_3_days
sensor.thz504_weather_forecast_today_condition
sensor.thz504_weather_forecast_tomorrow_condition
```

Die Wetterlogik erzeugt:

```yaml
binary_sensor.thz504_weather_cooling_pressure
```

Dieser wird `on`, wenn z. B.:

```text
heute >= 27 °C
oder morgen >= 27 °C
oder Max. der nächsten 3 Tage >= 30 °C
und Niederschlagswahrscheinlichkeit nicht zu hoch
```

Daraus wird dann:

```yaml
binary_sensor.thz504_cooling_weather_preconditioning_ready
```

Dieser Sensor ist strenger. Er wird nur `on`, wenn zusätzlich:

```text
Tagzeit aktiv
PV-Vorkühlung aktiviert
Wetter-Vorkühlung aktiviert
Weather Cooling Pressure = on
Raumtemperatur mindestens leicht über Zieltemperatur liegt
```

Dadurch wird nicht bei 20 °C Raumtemperatur nur wegen heißer Prognose gekühlt.

---

## 4. Energie- und PV-Logik

Die Energieentscheidung läuft über:

```yaml
binary_sensor.thz504_cooling_energy_available
```

Dieser wird `on`, wenn mindestens eine Energiebedingung erfüllt ist:

```text
ausreichender PV-Überschuss
oder ausreichende Netzeinspeisung
oder gute PV-Prognose
oder Wetter-Vorkühlung mit ausreichender PV-Prognose
oder Batterie ausreichend voll
oder Batterie wird geladen
oder PV wird abgeregelt
oder Komfort-Override aktiv
```

Wichtige Quellen:

```yaml
sensor.pv_ueberschuss
sensor.senec_netz_einspeisung
sensor.senec_netz_bezug
sensor.senec_pv_erzeugung
sensor.senec_pv_begrenzung
sensor.senec_akku_ladestand
sensor.senec_akku_laden
sensor.senec_akku_entladen
sensor.solcast_pv_forecast_forecast_next_hour
sensor.solcast_pv_forecast_forecast_today
```

Die Logik bevorzugt PV, verhindert aber, dass der Komfort komplett von PV abhängig wird.

---

## 5. Komfort-Override

```yaml
binary_sensor.thz504_cooling_comfort_override
```

wird aktiv, wenn die Raumtemperatur zu hoch wird.

Default:

```text
24,8 °C
```

Dann darf die Kühlung auch starten oder weiterlaufen, wenn PV-/Batteriebedingungen nicht ideal sind.

Zweck: Komfort hat Vorrang, PV-Optimierung ist sekundär.

---

## 6. Taupunkt- und Feuchteschutz

```yaml
binary_sensor.thz504_cooling_dew_point_safe
```

prüft:

```text
mittlere Luftfeuchte
THZ-Taupunkt
Vorlauftemperatur
Verdichterstatus
Tag-/Nachtgrenzwerte
```

Tagsüber default:

```text
max. Taupunkt: 17,0 °C
max. Luftfeuchte: 65 %
```

Nachts default:

```text
max. Taupunkt: 16,5 °C
max. Luftfeuchte: 62 %
```

Zusätzlich gilt:

```text
Vorlauf muss mindestens 3 K über Taupunkt liegen, wenn der Verdichter läuft
```

Das ist eine zusätzliche HA-Sicherheitslogik zur THZ-internen Taupunktüberwachung.

---

## 7. Verdichterschutz

```yaml
binary_sensor.thz504_cooling_compressor_starts_safe
```

prüft die Starts pro Tag.

Default:

```text
max. 12 Starts/Tag
```

Zusätzlich gibt es:

```yaml
timer.thz504_cooling_min_runtime
timer.thz504_cooling_lockout
```

Default:

```text
Mindestlaufzeit: 90 Minuten
Sperrzeit nach Stopp: 90 Minuten
```

Dadurch werden Kurzzyklen reduziert.

---

## 8. Start Conditions, Start Allowed und Start Blocker

Ab v0.6.3 gibt es drei getrennte Diagnosewerte.

### `binary_sensor.thz504_cooling_start_conditions_met`

Bedeutung:

```text
Alle fachlichen Startbedingungen sind erfüllt.
```

Dabei ist egal, ob die Kühlung bereits läuft.

Dieser Sensor wird `on`, wenn:

```text
Automation aktiv
Sensordaten gültig
Kühlbedarf oder Wetter-Vorkühlung vorhanden
Energie verfügbar
Taupunkt sicher
Verdichterstarts ok
Zeitfenster erlaubt
Lockout idle
```

### `binary_sensor.thz504_cooling_start_allowed`

Bedeutung:

```text
Startbedingungen erfüllt und Kühlung aktuell aus.
```

Wenn die Kühlung bereits aktiv ist, bleibt dieser Sensor absichtlich `off`.

### `sensor.thz504_cooling_start_blocker`

Zeigt konkret, warum nicht gestartet werden darf.

Mögliche Werte:

```text
none
automation_disabled
invalid_sensor_data
no_cooling_demand_or_weather_preconditioning
energy_not_available
dew_point_not_safe
compressor_starts_not_safe
time_window_not_allowed
lockout_active
cooling_already_on
```

Das ist der wichtigste Debug-Sensor.

---

## 9. Stopplogik

```yaml
binary_sensor.thz504_cooling_stop_required
```

wird `on`, wenn die Kühlung aktiv ist und mindestens eine Stoppbedingung erfüllt ist:

```text
Zieltemperatur erreicht
Taupunkt/Feuchte nicht sicher
keine Energie mehr verfügbar und kein Komfort-Override
Verdichterstarts-Limit erreicht
Zeitfenster blockiert und kein Komfort-Override
Sensordaten ungültig
```

Die normale Stop-Automation wartet zusätzlich:

```text
Stop Required muss 20 Minuten stabil on sein
Mindestlaufzeit muss abgelaufen sein
```

Der Sicherheitsstopp umgeht diese Komfortlogik.

---

## 10. Sicherheitsstopp

Die Sicherheitsautomation stoppt sofort, wenn:

```text
Sensordaten länger ungültig
Taupunktsicherheit verletzt
Vorlauf zu nahe am Taupunkt
```

Sie startet danach die Lockout-Sperrzeit.

---

# Einstellmöglichkeiten

## Entity-Mapping über `input_text`

Diese Helfer machen das Package portabel. Andere Nutzer müssen hier ihre eigenen Entitäten eintragen.

| Helper                                                | Funktion                                      |
| ----------------------------------------------------- | --------------------------------------------- |
| `input_text.thz504_entity_indoor_temp_average`        | ungewichtete mittlere Raumtemperatur          |
| `input_text.thz504_entity_indoor_temp_weighted`       | gewichtete Führungsraumtemperatur für Kühlung |
| `input_text.thz504_entity_indoor_humidity_average`    | mittlere Luftfeuchte im Haus                  |
| `input_text.thz504_entity_outdoor_temp`               | Außentemperatur                               |
| `input_text.thz504_entity_weather`                    | Wetter-Entity für `weather.get_forecasts`     |
| `input_text.thz504_entity_pv_generation`              | aktuelle PV-Erzeugung                         |
| `input_text.thz504_entity_pv_direct_consumption`      | PV-Direktverbrauch                            |
| `input_text.thz504_entity_pv_surplus`                 | PV-Überschuss                                 |
| `input_text.thz504_entity_pv_curtailment`             | PV-Abregelung/Begrenzung                      |
| `input_text.thz504_entity_grid_import`                | Netzbezug                                     |
| `input_text.thz504_entity_grid_export`                | Netzeinspeisung                               |
| `input_text.thz504_entity_battery_soc`                | Akku-SOC                                      |
| `input_text.thz504_entity_battery_charge`             | Akku-Ladeleistung                             |
| `input_text.thz504_entity_battery_discharge`          | Akku-Entladeleistung                          |
| `input_text.thz504_entity_pv_forecast_next_hour`      | PV-Prognose nächste Stunde in Wh              |
| `input_text.thz504_entity_pv_forecast_today`          | PV-Prognose heute in Wh                       |
| `input_text.thz504_entity_pv_forecast_tomorrow`       | PV-Prognose morgen in Wh                      |
| `input_text.thz504_entity_cooling_mode_switch`        | THZ-Kühlmodus-Schalter                        |
| `input_text.thz504_entity_compressor`                 | Verdichterstatus                              |
| `input_text.thz504_entity_compressor_starts_today`    | Verdichterstarts heute                        |
| `input_text.thz504_entity_compressor_runtime_current` | aktuelle Verdichterlaufzeit                   |
| `input_text.thz504_entity_flow_temperature`           | Vorlauftemperatur                             |
| `input_text.thz504_entity_return_temperature`         | Rücklauftemperatur                            |
| `input_text.thz504_entity_dewpoint`                   | THZ-Taupunkt                                  |
| `input_text.thz504_entity_climate_day`                | THZ Tag-Solltemperatur-Entity                 |
| `input_text.thz504_entity_climate_night`              | THZ Nacht-Solltemperatur-Entity               |
| `input_text.thz504_entity_notify_service`             | Push-Service                                  |

---

## Schalter über `input_boolean`

| Helper                                                         | Funktion                                  |
| -------------------------------------------------------------- | ----------------------------------------- |
| `input_boolean.thz504_cooling_automation_enabled`              | Hauptfreigabe der gesamten Automation     |
| `input_boolean.thz504_cooling_allow_night`                     | erlaubt Kühlung außerhalb des Tagfensters |
| `input_boolean.thz504_cooling_pv_preconditioning_enabled`      | erlaubt PV-basierte Vorkühlung            |
| `input_boolean.thz504_cooling_weather_preconditioning_enabled` | erlaubt wetterbasierte Vorkühlung         |

---

## Zeitfenster über `input_datetime`

| Helper                                         | Default | Funktion        |
| ---------------------------------------------- | ------: | --------------- |
| `input_datetime.thz504_cooling_day_start_time` |   06:00 | Beginn Taglogik |
| `input_datetime.thz504_cooling_day_end_time`   |   21:00 | Ende Taglogik   |

Wenn Start- und Endzeit gleich sind, gilt rechnerisch immer Tagbetrieb.

---

## Temperatur- und Energieparameter über `input_number`

| Helper                                          |  Default | Funktion                                                       |
| ----------------------------------------------- | -------: | -------------------------------------------------------------- |
| `thz504_cooling_day_start_temp`                 |  23,2 °C | normale Tag-Starttemperatur                                    |
| `thz504_cooling_day_stop_temp`                  |  22,0 °C | Tag-Ziel-/Stopptemperatur                                      |
| `thz504_cooling_night_start_temp`               |  23,8 °C | normale Nacht-Starttemperatur                                  |
| `thz504_cooling_night_stop_temp`                |  22,8 °C | Nacht-Ziel-/Stopptemperatur                                    |
| `thz504_cooling_comfort_override_temp`          |  24,8 °C | Komfort hat Vorrang vor Energieoptimierung                     |
| `thz504_cooling_day_setpoint`                   |  22,0 °C | wird an THZ Tag-Climate-Entity gesendet                        |
| `thz504_cooling_night_setpoint`                 |  22,8 °C | wird an THZ Nacht-Climate-Entity gesendet                      |
| `thz504_cooling_min_pv_surplus_day`             |    700 W | Mindest-PV-Überschuss tagsüber                                 |
| `thz504_cooling_min_grid_export_day`            |    500 W | Mindest-Netzeinspeisung tagsüber                               |
| `thz504_cooling_max_grid_import_day`            |    500 W | maximal tolerierter Netzbezug tagsüber                         |
| `thz504_cooling_max_grid_import_night`          |    250 W | maximal tolerierter Netzbezug nachts                           |
| `thz504_cooling_min_battery_soc_day`            |     45 % | Mindest-Akku-SOC tagsüber                                      |
| `thz504_cooling_min_battery_soc_night`          |     70 % | Mindest-Akku-SOC nachts                                        |
| `thz504_cooling_max_battery_discharge_night`    |   1500 W | max. Akkuentladung nachts                                      |
| `thz504_cooling_min_pv_forecast_next_hour_wh`   |   800 Wh | Mindestprognose nächste Stunde                                 |
| `thz504_cooling_min_pv_forecast_today_wh`       | 12000 Wh | Mindestprognose heute                                          |
| `thz504_cooling_weather_hot_temp`               |    27 °C | Schwelle für heißes Wetter                                     |
| `thz504_cooling_weather_very_hot_temp`          |    30 °C | Schwelle für sehr heißes Wetter                                |
| `thz504_cooling_weather_max_precip_probability` |     60 % | max. Niederschlagswahrscheinlichkeit für Wetterdruck           |
| `thz504_cooling_weather_precooling_offset`      |    0,4 K | Absenkung der Starttemperatur bei Wetter-Vorkühlung            |
| `thz504_cooling_max_dewpoint_day`               |  17,0 °C | max. Taupunkt tagsüber                                         |
| `thz504_cooling_max_dewpoint_night`             |  16,5 °C | max. Taupunkt nachts                                           |
| `thz504_cooling_max_humidity_day`               |     65 % | max. Luftfeuchte tagsüber                                      |
| `thz504_cooling_max_humidity_night`             |     62 % | max. Luftfeuchte nachts                                        |
| `thz504_cooling_dewpoint_safety_margin`         |    3,0 K | Mindestabstand Vorlauf zu Taupunkt                             |
| `thz504_cooling_max_compressor_starts`          |       12 | maximale Verdichterstarts pro Tag                              |
| `thz504_cooling_min_effective_delta`            |    0,5 K | minimale Rücklauf-/Vorlauf-Spreizung für Kälteabnahme-Diagnose |

---

## Timer

| Timer                              | Default | Funktion                                      |
| ---------------------------------- | ------: | --------------------------------------------- |
| `timer.thz504_cooling_min_runtime` |  90 min | verhindert zu frühes Abschalten nach Start    |
| `timer.thz504_cooling_lockout`     |  90 min | verhindert erneutes Starten direkt nach Stopp |

---

# Vom Package erzeugte Sensoren

## Wetter-Sensoren

| Sensor                                                              | Funktion                                                    |
| ------------------------------------------------------------------- | ----------------------------------------------------------- |
| `sensor.thz504_weather_forecast_today_temperature`                  | Tageshöchsttemperatur heute                                 |
| `sensor.thz504_weather_forecast_tomorrow_temperature`               | Tageshöchsttemperatur morgen                                |
| `sensor.thz504_weather_forecast_max_temperature_next_3_days`        | höchste Temperatur der nächsten 3 Tage                      |
| `sensor.thz504_weather_forecast_max_precip_probability_next_3_days` | höchste Niederschlagswahrscheinlichkeit der nächsten 3 Tage |
| `sensor.thz504_weather_forecast_today_condition`                    | Wetterzustand heute                                         |
| `sensor.thz504_weather_forecast_tomorrow_condition`                 | Wetterzustand morgen                                        |
| `binary_sensor.thz504_weather_cooling_pressure`                     | Prognose spricht für Kühlung/Vorkühlung                     |
| `binary_sensor.thz504_cooling_weather_preconditioning_ready`        | Wetter-Vorkühlung ist tatsächlich als Startgrund zulässig   |

---

## Diagnose- und Regel-Sensoren

| Sensor                                                   | Funktion                                                  |
| -------------------------------------------------------- | --------------------------------------------------------- |
| `sensor.thz504_cooling_net_grid_surplus`                 | Netzeinspeisung minus Netzbezug                           |
| `sensor.thz504_cooling_effective_pv_surplus`             | effektiver PV-Überschuss aus PV-Überschuss und Netzbilanz |
| `sensor.thz504_cooling_return_flow_delta`                | Rücklauf minus Vorlauf; Diagnose der Kälteabnahme         |
| `sensor.thz504_cooling_active_setpoint`                  | aktuell verwendeter Tag-/Nacht-Zielwert                   |
| `sensor.thz504_cooling_dynamic_start_temperature`        | dynamische Starttemperatur, ggf. durch Wetter abgesenkt   |
| `sensor.thz504_cooling_control_reason`                   | aktueller Hauptstatus/Grund                               |
| `sensor.thz504_cooling_start_blocker`                    | konkrete Startblocker                                     |
| `sensor.thz504_cooling_score`                            | Diagnose-Score 0–100 % für Kühlpriorität                  |
| `binary_sensor.thz504_cooling_daytime`                   | aktuell Tagfenster aktiv                                  |
| `binary_sensor.thz504_cooling_sensor_data_valid`         | alle gemappten Pflichtsensoren gültig                     |
| `binary_sensor.thz504_cooling_demand`                    | normaler Kühlbedarf vorhanden                             |
| `binary_sensor.thz504_cooling_comfort_override`          | Komforttemperatur überschritten                           |
| `binary_sensor.thz504_cooling_energy_available`          | Energiebedingungen erfüllt                                |
| `binary_sensor.thz504_cooling_dew_point_safe`            | Taupunkt-/Feuchtebedingungen sicher                       |
| `binary_sensor.thz504_cooling_compressor_starts_safe`    | Verdichterstarts unter Tageslimit                         |
| `binary_sensor.thz504_cooling_time_window_allowed`       | Tagfenster aktiv oder Nachtkühlung erlaubt                |
| `binary_sensor.thz504_cooling_effective_heat_extraction` | Spreizung plausibel bzw. keine Prüfung nötig              |
| `binary_sensor.thz504_cooling_start_conditions_met`      | alle fachlichen Startbedingungen erfüllt                  |
| `binary_sensor.thz504_cooling_start_allowed`             | Startbedingungen erfüllt und Kühlung aktuell aus          |
| `binary_sensor.thz504_cooling_stop_required`             | Stoppbedingung liegt an                                   |

---

# Automationen

## `THZ504 - Dynamic Floor Cooling Start`

Startet die Kühlung, wenn:

```text
Start Allowed = on
für 15 Minuten stabil
```

Dann passiert:

```text
Tag-Zieltemperatur an THZ senden
Nacht-Zieltemperatur an THZ senden
Kühlmodus einschalten
Mindestlaufzeit-Timer starten
Pushnachricht senden
Logbook-Eintrag schreiben
```

---

## `THZ504 - Dynamic Floor Cooling Stop`

Stoppt die Kühlung, wenn:

```text
Stop Required = on
für 20 Minuten stabil
Mindestlaufzeit abgelaufen
Kühlmodus aktiv
```

Dann passiert:

```text
Kühlmodus ausschalten
Lockout-Timer starten
Pushnachricht senden
Logbook-Eintrag schreiben
```

---

## `THZ504 - Dynamic Floor Cooling Safety Stop`

Stoppt sofort bei sicherheitsrelevanten Zuständen:

```text
Sensordaten ungültig
Taupunkt nicht sicher
Vorlauf zu nahe am Taupunkt
```

Danach:

```text
Kühlmodus aus
Lockout aktiv
Pushnachricht
Logbook
```

---

## `THZ504 - Dynamic Floor Cooling Setpoint Sync`

Synchronisiert UI-Änderungen der Zieltemperaturen direkt an die THZ.

Trigger:

```yaml
input_number.thz504_cooling_day_setpoint
input_number.thz504_cooling_night_setpoint
Home Assistant Start
```

Damit werden geänderte Solltemperaturen nicht erst beim nächsten Kühlstart übertragen.

---

## `THZ504 - Dynamic Floor Cooling Status Notification`

Sendet Pushmeldungen bei relevanten Zustandsänderungen, z. B.:

```text
Startbedingungen erfüllt
Start erlaubt
Startblocker geändert
Stop erforderlich
Energie verfügbar/nicht verfügbar
Taupunkt sicher/unsicher
Wetterdruck aktiv
Vorkühlung bereit
Zieltemperatur geändert
Day/Night-Zeit geändert
```

---

# Praktische Lesart der wichtigsten Zustände

```yaml
binary_sensor.thz504_cooling_start_conditions_met: on
binary_sensor.thz504_cooling_start_allowed: off
sensor.thz504_cooling_start_blocker: cooling_already_on
```

Bedeutet:

```text
Alles passt, aber Kühlung läuft bereits.
```

```yaml
binary_sensor.thz504_cooling_start_conditions_met: off
sensor.thz504_cooling_start_blocker: energy_not_available
```

Bedeutet:

```text
Kühlung wäre thermisch sinnvoll, aber Energiebedingung blockiert.
```

```yaml
sensor.thz504_cooling_control_reason: cooling_active_conditions_met
```

Bedeutet:

```text
Kühlung läuft, und die Bedingungen sprechen weiterhin für Betrieb.
```

```yaml
sensor.thz504_cooling_control_reason: stop_required
```

Bedeutet:

```text
Kühlung soll beendet werden, sobald die Stop-Automation darf.
```

Die nützlichsten Dashboard-Entitäten sind:

```yaml
sensor.thz504_cooling_control_reason
sensor.thz504_cooling_start_blocker
binary_sensor.thz504_cooling_start_conditions_met
binary_sensor.thz504_cooling_start_allowed
binary_sensor.thz504_cooling_stop_required
sensor.thz504_cooling_dynamic_start_temperature
sensor.thz504_cooling_score
sensor.thz504_cooling_return_flow_delta
binary_sensor.thz504_cooling_dew_point_safe
binary_sensor.thz504_cooling_energy_available
```
