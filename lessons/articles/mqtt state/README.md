### MQTT — хранение состояний режимов работы автоматизаций Home Assistant

:arrow_right: [Вернуться к оглавлению](https://alexkvazis.blogspot.com/p/blog-page.html#git)

В ходе настройки Home Assistant я, в своё время, столкнулся следующей задачей — некоторые автоматизации, например управления отоплением, увлажнением, кондиционированием и т.п. — носят явно сезонный характер. Очевидно что нужен удобный механизм по управлению их запуском, чтобы не приходилось вручную изменять в коде initial_state. В идеале — какой-то переключатель, который можно добавить в условия таких автоматизаций и которым можно управлять вручную — выключатель режимов.    
Конечно первое что приходит в голову — это Input Boolean — логический переключатель который можно создать в Home Assistant и он по своим возможностям вполне подходит для этой задачи. Но мне хотелось создать ещё более универсальное решение, не зависящее от работы home assistant и с возможностью использования одновременно несколькими серверами умного дома. Для этого я решил использовать для хранения таких состояний — топики MQTT.

Моё решение состоит из двух взаимосвязанных частей.    
Первая — это бинарный сенсор на платформе — который меняет своё состояние в зависимости от содержимого определённого топика. Приведу пример переключателя режима отопления. Выглядит это так:

```yaml
binary_sensor:
  - platform: mqtt
    name: radiator
    state_topic: "states/radiator"
    payload_on: "ON"
    payload_off: "OFF"
```
По строкам -    
:one: — это указание платформы — MQTT    
:two: — имя сенсора, этот будет называться binary_sensor.radiator и в таком виде его можно использовать либо в в разделе условий автоматизаций отопления, либо как часть других шаблонных сенсоров, отвечающих за работу таких автоматизаций.    
:three: — это имя топика MQTT, в который смотрит этот сенсор.    
:four: и :five: — это значения записанные в этот топик, при которых он будет включён — ON, или выключен — OFF.    

Вторая часть решения — это выключатель, который и пишет эти самые значение в топик. Построен он на платформе шаблонов.

```yaml
switch:
  - platform: template
    switches:
      radiator_mode:
        friendly_name: "Режим отопления"
        value_template: "{{  is_state('binary_sensor.radiator', 'on') }}"
        turn_on:
          service: mqtt.publish
          data_template:
            topic: "states/radiator"
            payload_template: 'ON'
            retain: true 
        turn_off:
          service: mqtt.publish
          data_template:
            topic: "states/radiator"
            payload_template: 'OFF'
            retain: true 
        icon_template: >-
          {% if is_state('switch.radiator_mode', 'on') %}
            mdi:radiator
          {% else %}
            mdi:radiator-off
          {% endif %}
```

Пройдёмся по строкам.    
:one: — это платформа шаблонов, на которой создан этот переключатель.    
:two: — открывает перечень выключателей, их может быть много.    
:three: — имя выключателя, в данном случае он будет называться switch.radiator_mode    
:four: — это имя выключателя которое будет отображаться в интерфейсе.    
Далее — шаблон который должен выполняться, чтобы выключатель имел статус включено. В данном случае он привязан к уже рассмотренному бинарному сенсору — их состояния синхронизированы.    
Далее идёт описание тех действий, которые будет производить система при его включении и выключении. В разделе turn_on — запускается сервис mqtt.publish, он пишет в тот самый топик, в который смотрит бинарник — значение ON, тем самым включая его.    
Последняя строка — retain: true сохраняет это значение в топике, она обязательна, иначе это значение будет теряться после каждой перезагрузке Home Assistant.    
Аналогично при действии turn_off — в топик пишется значение OFF, которое деактивирует сенсор.    
В конце — в разделе icon_template, указаны иконки, которые будет присвоены для отображения выключателя, в зависимости от его состояния.    
Таким образом — режим состояния вынесен из внутренней базы Home Assistant в MQTT брокер, где он будет хранится постоянно и может использоваться другими серверами — для этого можно либо подключать все интеграции MQTT к одному брокеру, либо синхронизировать брокеры в режиме моста.    

____
### Как поддержать развитие проекта?
* [Стать спонсором моего Youtube](http://kvazis.link/sponsorship)
* [Подписаться на Patreon](http://kvazis.link/patreon)
* [Перевод через Paypal](http://kvazis.link/paypal)
* Webmoney - Z243592584952
* BTC - 1Gzr7WQugfnPuWVawu47EiCMTDUBqCAshj
* ETH - 0xa0ce3E29Cf537013649Ae9cdbc08C4853fF91FAc
* LTC - ltc1qs493yk2wk9ywx5h6aruk4p9zm75hx42ekv4ym2
* TRX - TFTCLqvS1tMBwokRHBwz1TCDJ4oD1Z5zPk