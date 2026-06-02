### In configuration.yaml
```yaml
shell_command:
  find_entity: python3 /config/scripts/parser.py "{{ query }}"
```

#### Create `parser.py` in the folder specified above and place this code in the file

```py
#!/usr/bin/env python3
import json
import sys
import os

ENTITY_REGISTRY_PATH = "/config/.storage/core.entity_registry"

def normalize_string(s):
    return str(s).lower().strip() if s else ""

def find_entity_id(entities_list, search_query):
    """Find entity"""
    normalized_query = normalize_string(search_query)
    if not normalized_query:
        return None

    for entity in entities_list:
        options = entity.get("options") or {}
        conversation = options.get("conversation") or {}
        
        if not conversation.get("should_expose"):
            continue

        if normalize_string(entity.get("name")) == normalized_query:
            return entity.get("entity_id")

        raw_aliases = (entity.get("aliases") or []) + (entity.get("aliases_v2") or [])
        
        if any(normalized_query == normalize_string(a) for a in raw_aliases):
            return entity.get("entity_id")

    return None

def load_entities(file_path):
    """Загружает сущности из файла core.entity_registry."""
    if not os.path.exists(file_path):
        return None
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        return data.get("data", {}).get("entities", [])
    except (json.JSONDecodeError, Exception):
        return None

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Error: No query provided")
        sys.exit(1)
    
    query = sys.argv[1]
    entities_list = load_entities(ENTITY_REGISTRY_PATH)
    if entities_list is None:
        print("Error: Failed to load entities")
        sys.exit(1)
    
    entity_id = find_entity_id(entities_list, query)
    print(entity_id if entity_id else "None")
```



#### Create a new automation via the interface, switch to text mode, and paste the code.
## `automation.voice_start_with_timer`
```yaml
alias: "Voice_start_with_timer"
triggers:
  - trigger: conversation
    command: Запусти {name} на {time}
conditions: []
actions:
  - data:
      query: "{{ trigger.slots.name }}"
    response_variable: response
    action: shell_command.find_entity
  - if:
      - condition: template
        value_template: "{{ response.stdout == 'None' }}"
    then:
      - set_conversation_response: Объект отсутствует
    else:
      - action: script.turn_on
        data:
          variables:
            id: "{{ response.stdout }}"
            time: "{{ trigger.slots.time }}"
        target:
          entity_id: script.light_with_delay
      - set_conversation_response: Ok
mode: single
```

#### Create a new script via the interface, switch to text mode, and paste the code. After saving, check the names so that the automation calls this script. If you are using a different language, replace the initial masks for minutes and seconds. "мин" -> "min" and so on.

## `script.light_with_delay`
```yaml
alias: light_with_delay
mode: parallel
max: 5
sequence:
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ id.split('.')[0] == 'light' }}"
        sequence:
          - action: light.turn_on
            target:
              entity_id: "{{ id }}"
          - delay: >
              {% set time_str = time | default('10 секунд') %} {% set number =
              (time_str | regex_findall('^\\d+') | first | default(1) | int) %}
              {% set unit = (time_str | regex_replace('^\\d+\\s*', '') | lower)
              %} {% if unit.startswith('сек') %}
                {{ '00:00:%02d' | format(number) }}
              {% elif unit.startswith('мин') %}
                {{ '00:%02d:00' | format(number) }}
              {% else %}
                {{ '00:00:10' }}
              {% endif %}
          - action: light.turn_off
            target:
              entity_id: "{{ id }}"
      - conditions:
          - condition: template
            value_template: "{{ id.split('.')[0] == 'switch' }}"
        sequence:
          - action: switch.turn_on
            target:
              entity_id: "{{ id }}"
          - delay: >
              {% set time_str = time | default('10 секунд') %} {% set number =
              (time_str | regex_findall('^\\d+') | first | default(1) | int) %}
              {% set unit = (time_str | regex_replace('^\\d+\\s*', '') | lower)
              %} {% if unit.startswith('сек') %}
                {{ '00:00:%02d' | format(number) }}
              {% elif unit.startswith('мин') %}
                {{ '00:%02d:00' | format(number) }}
              {% else %}
                {{ '00:00:10' }}
              {% endif %}
          - action: switch.turn_off
            target:
              entity_id: "{{ id }}"
```

#### Универсальный вариант для русского под любой вывод asr (цифры или слова)

```yaml
alias: light_with_delay
mode: parallel
max: 5
sequence:
  - variables:
      delay_time: >
        {% set time_str = time | default('10 секунд') | lower | trim %} {% if
        'полчаса' in time_str %}
          {{ '00:30:00' }}
        {% elif 'полторы' in time_str %}
          {{ '00:01:30' }}
        {% else %}
          {% set w2n = {
            'один': 1, 'одну': 1, 'два': 2, 'две': 2, 'три': 3, 'четыре': 4, 'пять': 5, 'шесть': 6,
            'семь': 7, 'восемь': 8, 'девять': 9, 'десять': 10, 'одиннадцать': 11,
            'двенадцать': 12, 'тринадцать': 13, 'четырнадцать': 14, 'пятнадцать': 15,
            'шестнадцать': 16, 'семнадцать': 17, 'восемнадцать': 18,
            'девятнадцать': 19, 'двадцать': 20, 'тридцать': 30, 'сорок': 40,
            'пятьдесят': 50, 'шестьдесят': 60
          } %}

          {% set num_digits = (time_str | regex_findall('^\d+') | first) %}

          {% if num_digits %}
            {% set number = num_digits | int %}
            {% set unit = time_str | regex_replace('^\d+\s*', '') %}
          {% else %}
            {% set ns = namespace(num=0, unit_start=false, unit_str='') %}
            {% for word in time_str.split() %}
              {% if not ns.unit_start and word in w2n %}
                {% set ns.num = ns.num + w2n[word] %}
              {% else %}
                {% set ns.unit_start = true %}
                {% set ns.unit_str = ns.unit_str + ' ' + word %}
              {% endif %}
            {% endfor %}

            {% set number = ns.num if ns.num > 0 else (1 if ns.unit_str | trim else 10) %}
            {% set unit = ns.unit_str | trim if ns.unit_str | trim else 'секунд' %}
          {% endif %}

          {% if unit.startswith('мин') %}
            {{ '00:%02d:00' | format(number) }}
          {% elif unit.startswith('час') %}
            {{ '%02d:00:00' | format(number) }}
          {% elif unit.startswith('сек') %}
            {{ '00:00:%02d' | format(number) }}
          {% else %}
            {{ '00:00:10' }}
          {% endif %}
        {% endif %}
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ id.split('.')[0] == 'light' }}"
        sequence:
          - action: light.turn_on
            target:
              entity_id: "{{ id }}"
          - delay: "{{ delay_time }}"
          - action: light.turn_off
            target:
              entity_id: "{{ id }}"
      - conditions:
          - condition: template
            value_template: "{{ id.split('.')[0] == 'switch' }}"
        sequence:
          - action: switch.turn_on
            target:
              entity_id: "{{ id }}"
          - delay: "{{ delay_time }}"
          - action: switch.turn_off
            target:
              entity_id: "{{ id }}"


```


