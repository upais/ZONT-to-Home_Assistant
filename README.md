# Управление устройствами ZONT для котлов

Для интерграции с Home Assistant нужно выполнить следующие действия:

### 1) Через любой сервис закодировать ваши логин и пароль к ZONT в Base64.
[На примере сервиса](https://base64encode.net)
+    Выбираем "Base64 Encode"
+    В первое поле вводим ваше логин и пароль в формате "login:passoword" (без ковычек)
+    Жмем "Encode" и получаем закодированное значение во втором поле. Сохраняем его в надежном месте.
<details>
<summary>Скриншот</summary>

![Screenshot_1](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/bff5563c-dbd7-4e5c-baf9-10211246ac92)
</details>

### 2) Через утилиту "Advanced REST Client" получаем токет аутентификации к API ZONT.
[Ссылка](https://github.com/advanced-rest-client/arc-electron/releases)
+    Выбираем метод "POST"
+    В адрес вставляем URL: https://lk.zont-online.ru/api/get_authtoken
+    Во вкладке "Headers" выбираем "Text editor"
+    Вставляем набор Header'ов:
````
content-type: application/json
authorization: Basic XXXXXXXX
x-zont-client: your@email
client_name: "Home Assistant"
````
где вместо ***XXXXXXXX*** закодированное значение логина и пароля из пункта №1.
   
+   Жмем кнопку отправки запроса. Если все верно, внизу будет код ответ "200", а в теле ответа нас интересует значение "token". Сохраняем его в надежном месте.
<details>
<summary>Скриншот</summary>

![Screenshot_2](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/5c288f11-e75d-47cb-b172-35ee50e75228)
</details>

### 3) Через утилиту "Advanced REST Client" получаем содержимое параметров и команд вашего устройства.
+    Выбираем метод "POST"
+    В адрес вставляем URL: https://lk.zont-online.ru/api/devices
+    Во вкладке "Headers" выбираем "Text editor"
+    Вставляем набор Header'ов:   
````
content-type: application/json
x-zont-client: your@email
x-zont-token: YYYYYYYY
````
где вместо ***YYYYYYYY*** значение токена из пункта №2.
+   Во вкладке "Body" заполняем значение: {"load_io":true}
+   Жмем кнопку отправки запроса. Если все верно, внизу будет код ответ "200", а в ответе будет большой JSON с параметрами вашего устройства, с которым далее и будем работать.
   Рекомендую его вставить в какой-нибудь обработчик JSON ([можно веб версии](https://jsoneditoronline.org), можно через notepad++), так удобнее работать.
<details>
<summary>Скриншоты</summary>
    
![Screenshot_3](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/2bd2d646-f8aa-48f0-bfb0-facdf171b539)
![Screenshot_4](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/8743658e-1ce7-41a8-91c3-e745e70f4262)
</details>

### 4) Получаем device_id и пути до нужных атрибутов
Далее все сильно зависит от модели вашего устройства и ваших потребностей, как выяснилось, у разных устройств разные структуры ответов и команд. Например, то что описано в статье на sprut.ai не подошло для моего устройства и я обращался в тех. поддержку ZONT, которая мне предоставила команды для моего устройства. А офф. документация к API была устаревшая.
По этому, если мои команды не работают, пробуйте из статьи на sprut.ai, а если и они не работают, то пишите в поддержку ZONT.

Итак, теперь вам из ответа в пункте №3 необходимо получать id компонентов, которие вы ходите читать или управлять, а точнее не просто id, а JSON-путь до атрибутов.
Основное, что тут требуется это значение для device_id. А разные значения - температура, статусы и т.д. ищутся ниже по дереву.
Идентификатор прибора (device_id) можно увидеть в настройках. Идентификатор отопительного контура (object_id): нужно отправить запрос на https://zont-online.ru/api/devices, найти нужный контур в ответе в массиве **devices[].z3k_config.heating_circuits**, и у нужного контура взять значение поля id.

<details>
<summary>Скриншоты</summary>
    
![Screenshot_5](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/d5d698a1-3bb7-4f54-87a7-a5e381cf7ed0)
![Screenshot_6](https://github.com/upais/ZONT-to-Home_Assistant/assets/86227709/e3b6cce0-38a0-4d15-a9d2-2afcc7892536)
</details>

+ Запрос для смены целевой температуры выглядит так:
```
https://zont-online.ru/api/send_z3k_command
{
  "device_id": ...,  // идентификатор прибора
  "object_id": ...,  // идентификатор контура
  "command_name": "TargetTemperature",
  "command_args": {"value": 56}, // желаемая целевая
  "is_guaranteed": false/true
}
```

+ Запрос для смены режима:
```
https://zont-online.ru/api/send_z3k_command
{
  "device_id": ..., // идентификатор прибора
  "object_id": ..., // идентификатор режима
  "command_name": "SelectHeatingMode",
  "command_args": null,
  "is_guaranteed": false/true
}
```
Идентификатор режима можно найти также в ответе на запрос devices в  **devices[].z3k_config.heating_modes [].id.**

**is_guaranteed** — признак гарантированной доставки команды. Если команду не удастся доставить сразу же, сервер запомнит её и будет пытаться доставить снова при выходе прибора на связь, пока прибор не подтвердит получение. Если **false**, сервер сделает однократную попытку доставить команду.

### 5) Формируете нужные вам запросы в сенсоры для Home Assistant.
Все зависит от ваших желаний и потребностей, ниже два варианта сделанных через Packages. Плюс приложу мой конфиг, который работает уже пару лет. По примерам, думаю, каждый сможет разобраться.


### Конфигурация №1: 
Получает только используемые атрибуты. Более правильный метод

```
climate_heating:
  rest:
    - resource: https://zont-online.ru/api/devices
      method: POST
      scan_interval: 60
      timeout: 20
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"load_io": true}'
      sensor:
        - name: "zont_rest_first_floor_temperature"
          json_attributes_path: "$.devices[0].io.z3k-state.4099"
          value_template: "OK"
          json_attributes:
            - "temperature"
        - name: "zont_rest_moduletion"
          json_attributes_path: "$.devices[0].io.z3k-state.20600.ot"
          value_template: "OK"
          json_attributes:
            - "rml"
        - name: "zont_rest_status_heating"
          json_attributes_path: "$.devices[0].io.z3k-state.20496"
          value_template: "OK"
          json_attributes:
            - "target_temp"
            - "setpoint_temp"
            - "status"
        - name: "zont_rest_burner_gvs"
          json_attributes_path: "$.devices[0].io.z3k-state.20603"
          value_template: "OK"
          json_attributes:
            - "status"
        - name: "zont_rest_temp_coolant"
          json_attributes_path: "$.devices[0].io.z3k-state.20600.ot"
          value_template: "OK"
          json_attributes:
            - "bt"
#
  template:
    sensor:
      - name: 'temperature_1st_floor'
        state: "{{ state_attr('sensor.zont_rest_first_floor_temperature', 'temperature') }}"
        availability: "{{is_state('sensor.zont_rest_first_floor_temperature', 'OK') }}"
        unit_of_measurement: '°C'
        icon: mdi:home-thermometer
#
      - name: 'target_temperature_heating'
        state: "{{ state_attr('sensor.zont_rest_status_heating', 'target_temp') }}"
        availability: "{{is_state('sensor.zont_rest_status_heating', 'OK') }}"
        unit_of_measurement: '°C'
        icon: mdi:temperature-celsius
#
      - name: 'moduletion_burner'
        state: "{{ state_attr('sensor.zont_rest_moduletion', 'rml')|round(0) }}"
        availability: "{{is_state('sensor.zont_rest_moduletion', 'OK') }}"
        unit_of_measurement: '%'
        icon: mdi:gas-burner
#
      - name: 'temperature_coolant'
        state: "{{ state_attr('sensor.zont_rest_burner_gvs', 'status')|round(0) }}"
        availability: "{{is_state('sensor.zont_rest_burner_gvs', 'OK') }}"
        unit_of_measurement: '°C'
        icon: mdi:coolant-temperature
#
      - name: 'calculation_temperature_coolant'
        state: "{{ state_attr('sensor.zont_rest_status_heating', 'setpoint_temp')|round(0) }}"
        availability: "{{is_state('sensor.zont_rest_status_heating', 'OK') }}"
        unit_of_measurement: '°C'
        icon: mdi:thermometer
#
      - name: 'burner_heating'
        state: "{{ state_attr('sensor.zont_rest_status_heating', 'status') }}"
        availability: "{{is_state('sensor.zont_rest_status_heating', 'OK') }}"
        icon: mdi:fire
#
      - name: 'burner_gvs'
        state: "{{ state_attr('sensor.zont_rest_burner_gvs', 'status') }}"
        availability: "{{is_state('sensor.zont_rest_burner_gvs', 'OK') }}"
        icon: mdi:fire
#
      - name: 'zont_status_heating'
        state: >
            {% if (state_attr('sensor.zont_rest_burner_gvs', 'status') == 1 ) and (state_attr('sensor.zont_rest_status_heating', 'status') == 0 ) %}
                ГВС 
            {% elif (state_attr('sensor.zont_rest_burner_gvs', 'status') == 0 ) and (state_attr('sensor.zont_rest_status_heating', 'status') == 1 ) %} 
                Отопление 
            {% elif (state_attr('sensor.zont_rest_burner_gvs', 'status') == 1 ) and (state_attr('sensor.zont_rest_status_heating', 'status') == 1 ) %} 
                Отопление | ГВС
            {% else %} Не активно {% endif %}
        icon: mdi:radiator
```

### Конфигурация №2: 
Получает весь JSON от ZONT и уже из него получаем желаемые атрибуты. Не очень правильный метод, тянет много, историю сохраняет не совсем корректно.
```
climate_heating:
  sensor:
  - platform: rest
    resource: https://zont-online.ru/api/devices
    name: zont
    method: POST
    timeout: 30
    scan_interval: 10
    force_update: true
    headers:
      X-ZONT-Client: !secret zont_client
      X-ZONT-Token: !secret zont_token
      Content-Type: application/json
    payload: '{"load_io": true}'
    value_template: '{{ value_json.ok }}'
    json_attributes:
      - devices
  - platform: template
    sensors:
      zont_first_floor_temperature:
        value_template: '{{ states.sensor.zont.attributes.devices[0].io["z3k-state"]["4099"].temperature }}'
        friendly_name: 'Температура 1-й этаж'
        device_class: temperature
        unit_of_measurement: '°C'
  - platform: template
    sensors:
      zont_moduletion:
        value_template: '{{ states.sensor.zont.attributes.devices[0].io["z3k-state"]["20600"]["ot"].rml |round(0) }}'
        friendly_name: 'Модуляция горелки'
        unit_of_measurement: '%'
  - platform: template
    sensors:
      zont_target_temperature:
        value_template: '{{ states.sensor.zont.attributes.devices[0].io["z3k-state"]["20496"].target_temp }}'
        friendly_name: 'Целевая температура'
        device_class: temperature
        unit_of_measurement: '°C'
  - platform: template
    sensors:
      zont_burner_heat:
        value_template: '{{ states.sensor.zont.attributes.devices[0].io["z3k-state"]["20496"].status }}'
        friendly_name: 'Горелка Отопление'
  - platform: template
    sensors:
      zont_burner_gvs:
        value_template: '{{ states.sensor.zont.attributes.devices[0].io["z3k-state"]["20603"].status }}'
        friendly_name: 'Горелка ГВС'

# Установка режима "Комфорт"
  rest_command:
    zont_set_comfort:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 20501, "command_name": "SelectHeatingMode", "command_args": null, "is_guaranteed": true}'

# Установка режима "Расписание"
    zont_set_scheduled:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 20503, "command_name": "SelectHeatingMode", "command_args": null, "is_guaranteed": true}'

# Установка режима "Выключен"
    zont_set_off:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 20504, "command_name": "SelectHeatingMode", "command_args": null, "is_guaranteed": true}'

# Установить +1 к текущей целевой температуре
    zont_up_target_temp:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 20496, "command_name": "TargetTemperature", "command_args": {"value": {{ (states.sensor.zont_target_temperature.state) | round(0) +1 }}}, "is_guaranteed": true}'

# Установить -1 к текущей целевой температуре
    zont_down_target_temp:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 20496, "command_name": "TargetTemperature", "command_args": {"value": {{ (states.sensor.zont_target_temperature.state) | round(0) -1 }}}, "is_guaranteed": true}'

# Установить целевую температуру из input_number (можно задавать дробные от 0.1)
    zont_set_target_temp:
      url: https://zont-online.ru/api/send_z3k_command
      method: POST
      headers:
        X-ZONT-Client: !secret zont_client
        X-ZONT-Token: !secret zont_token
        Content-Type: application/json
      payload: '{"device_id": 123456, "object_id": 33333, "command_name": "TargetTemperature", "command_args": {"value": {{ (states.input_number.target_heating_temperature.state) | round(1) }}}, "is_guaranteed": true}'
```

### Основано на:
1) https://sprut.ai/client/article/2292
2) Информации от технической поддержки ZONT и точно актуально для Zont H2000+, Zont Connect+. Причина обращения в том, что частично официальная документация описывающая API не актуальна.
