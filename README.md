# 🚌 KVV Abfahrten für Home Assistant

Dieses Projekt zeigt Abfahrten vom KVV (Karlsruher Verkehrsverbund) im Home Assistant Dashboard an.

Inspiriert wurde es durch den Vortrag von Till auf YouTube:  
📺 [KVV & Home Assistant Vortrag von Till](https://www.youtube.com/watch?v=_qhGuTVMc5A)  
Technische Grundlage: [Original-Repository von Till](https://github.com/harbaum/kvv)

Ich habe mit Hilfe von ChatGPT und ohne große Vorkenntnisse im Programmieren oder Scripten ein funktionierendes Dashboard gebaut.  

Fühlt euch frei, es zu verbessern oder anzupassen!

## Beispiel:

![grafik](https://github.com/user-attachments/assets/ad8a8f84-334a-4c42-9d49-b25e089fbc1d)

## Haltestellen ID finden

Die Haltestelle im Beispiel ist Karlsruhe Entenfang mit der Kataster ID 7000051.

Um deine richtige Haltestelle zu suchen passe die URL aus der kvv.yaml an und rufe diese im Browser auf:

## Vorher:

https://www.kvv.de/tunnelEfaDirect.php?outputFormat=JSON&language=de&coordOutputFormat=WGS84[DD.DDDDD]&type_dm=stop&name_dm=7000051&lookahead=60&limit=10&mode=direct&useRealtime=1&action=XSLT_DM_REQUEST

## Nachher:
https://www.kvv.de/tunnelEfaDirect.php?outputFormat=JSON&language=de&coordOutputFormat=WGS84[DD.DDDDD]&type_dm=stop&name_dm=Karlsruhe%20Europaplatz&lookahead=60&limit=10&mode=direct&useRealtime=1&action=XSLT_DM_REQUEST

* dm=7000051 wird zu dm=Karlsruhe Europaplatz
Karlsruhe ist klar der Ort, gefolgt vom (Teil)Name der Haltestelle. Es geht bspw. auch dm=Karlsruhe Europaplatz (U) für die Ustrab. Im Firefox funktioniert es mit Leerstellen und den () - Alternativ kann für die Leerstelle %20 genutzt werden.

In der angezeigten JSON ist die ```STOP ID 7000037``` zu finden.

---

## 🚀 Einrichtung

### 1. `configuration.yaml` anpassen

Füge folgende Zeile ein:

```yaml
rest: !include sensors/kvv.yaml
```

Alternativ kannst du den Inhalt direkt in deine `configuration.yaml` kopieren.

---

### 2. Neue Datei `sensors/kvv.yaml` erstellen

```yaml
- resource: "https://www.kvv.de/tunnelEfaDirect.php?outputFormat=JSON&language=de&coordOutputFormat=WGS84[DD.DDDDD]&type_dm=stop&name_dm=7000051&lookahead=60&limit=10&mode=direct&useRealtime=1&action=XSLT_DM_REQUEST"
  scan_interval: 60
  sensor:
    - name: "KVV Entenfang"
      value_template: >
        {% if value_json.departureList %}
          {{ value_json.departureList[0].servingLine.number }} → {{ value_json.departureList[0].servingLine.direction }} in {{ value_json.departureList[0].countdown }} min
        {% else %}
          Keine Abfahrten
        {% endif %}
      # Hier: json_attributes_path oder json_attributes je nachdem was deine HA-Version erwartet
      json_attributes:
        - departureList
```

---

### 3. Markdown-Karte im Dashboard erstellen

Erstelle im Dashboard eine neue **Markdown-Karte** mit folgendem Inhalt:

```yaml
type: markdown
title: 🚉 Abfahrten – Karlsruhe Entenfang
content: >
  {% set icons = {
    'Straßenbahn': '🚋',
    'S-Bahn': '🚆',
    'Stadtbus': '🚌',
    'Regionalbus': '🚌',
    'Einsatzwagen': '🚍'
  } %}

  {% set departures_raw = state_attr('sensor.kvv_entenfang', 'departureList') %}
  {% if departures_raw %}
    {% set ns = namespace(departures=[]) %}
    {% for dep in departures_raw %}
      {% set dep = dep | combine({'_countdown': dep.countdown | int }) %}
      {% set ns.departures = ns.departures + [dep] %}
    {% endfor %}
    {% set departures = ns.departures | sort(attribute='_countdown') %}
    {% set first = departures[0] %}
    {% set product = first.servingLine.name if first.servingLine.name is defined else '–' %}
    {% set planned = first.dateTime if first.dateTime is defined else none %}
    {% set real = first.realDateTime if first.realDateTime is defined else planned %}
    {% set p_hour = planned.hour | int if planned is not none else 0 %}
    {% set p_min = planned.minute | int if planned is not none else 0 %}
    {% set r_hour = real.hour | int if real is not none else p_hour %}
    {% set r_min = real.minute | int if real is not none else p_min %}
    {% set delay = (r_hour * 60 + r_min) - (p_hour * 60 + p_min) %}
    {% set delay_str = '+' ~ delay ~ ' Min' if delay > 0 else (delay ~ ' Min' if delay < 0 else '±0 Min') %}
    {% set icon = icons.get(product, '') %}

    **Nächste Abfahrt:**  
    {{ icon }} {{ first.servingLine.number }} ({{ product }}) → {{ first.servingLine.direction }}  
    🕒 {{ first.countdown }} min — {{ first.platformName if first.platformName is defined else '–' }}  
    📅 geplant {{ "%02d:%02d"|format(p_hour, p_min) if planned is not none else '–' }} — {{ delay_str }}

    {%- for dep in departures[1:] %}
      {%- set product = dep.servingLine.name if dep.servingLine.name is defined else '–' %}
      {%- set planned = dep.dateTime if dep.dateTime is defined else none %}
      {%- set real = dep.realDateTime if dep.realDateTime is defined else planned %}
      {%- set p_hour = planned.hour | int if planned is not none else 0 %}
      {%- set p_min = planned.minute | int if planned is not none else 0 %}
      {%- set r_hour = real.hour | int if real is not none else p_hour %}
      {%- set r_min = real.minute | int if real is not none else p_min %}
      {%- set delay = (r_hour * 60 + r_min) - (p_hour * 60 + p_min) %}
      {%- set delay_str = '+' ~ delay ~ ' Min' if delay > 0 else (delay ~ ' Min' if delay < 0 else '±0 Min') %}
      {%- set icon = icons.get(product, '') %}

      {{ icon }} {{ dep.servingLine.number }} ({{ product }}) → {{ dep.servingLine.direction }}  
      🕒 {{ dep.countdown }} min — {{ dep.platformName if dep.platformName is defined else '–' }}  
      📅 geplant {{ "%02d:%02d"|format(p_hour, p_min) if planned is not none else '–' }} — {{ delay_str }}
    {%- endfor %}
  {% else %}
    *(Keine Daten verfügbar)*
  {% endif %}
```

---
