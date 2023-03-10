

![image](https://user-images.githubusercontent.com/69485846/224211330-0c9cd10c-8ab0-42cd-baf8-bd22a5a70b8d.png)


keenetic.yaml
```
sensor:
  - platform: rest
    resource: http://192.168.0.1:81/rci/show/system
    name: Keenetic Processor Use
    value_template: "{{ value_json.cpuload }}"
    unit_of_measurement: '%'
    state_class: measurement
    icon: mdi:cpu-64-bit
    unique_id: Keenetic Processor Use
  - platform: rest
    resource: http://192.168.0.1:81/rci/show/system
    name: Keenetic Memory Use
    value_template: >
          {%- set mem = value_json.memory-%}
          {%- set memfree = mem.split('/')[0]|int(0)-%}
          {%- set memtotal = mem.split('/')[1]|int(1)-%}
          {{ (memfree*100/memtotal)| round(2) }}
    unit_of_measurement: '%'
    state_class: measurement
    icon: mdi:memory
    unique_id: Keenetic Memory Use
  - platform: rest
    resource: http://192.168.0.1:81/rci/show/system
    name: Keenetic Uptime
    device_class: duration
    unit_of_measurement: "s"
    unique_id: Keenetic Uptime
    value_template: "{{ (value_json.uptime) | int }}"
  - platform: rest
    resource: http://192.168.0.1:81/rci/show/interface/OpenVPN0
    name: Zaborona Uptime
    device_class: duration
    unit_of_measurement: "s"
    unique_id: Zaborona Uptime
    value_template: "{{ (value_json.uptime) | int }}"
binary_sensor:
  - platform: rest
    resource: http://192.168.0.1:81/rci/show/interface/OpenVPN0
    name: Zaborona
    value_template: >
          {%if value_json.state == "up" %}on{%else%}off{%endif%}
    unique_id: Zaborona
    device_class: connectivity
rest_command:
  keenetic_reboot:
    url: http://192.168.0.1:81/rci/system/reboot
    method: POST
    payload: "{}"
input_button:
  keenetic_reboot:
    name: Keenetic reboot
    icon: mdi:restart
automation:
  - alias: Keenetic reboot
    description: ''
    trigger:
      - platform: state
        entity_id: input_button.keenetic_reboot
    action:
      - service: rest_command.keenetic_reboot
        data: {}
```
http://docs.help.keenetic.com/cli/3.1/ru/cli_manual_kn-1711_ru.pdf
