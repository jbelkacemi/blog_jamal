---
title: Integrar una Báscula Bluetooth Swisshome en Home Assistant con ESPHome
description: Aprende cómo integrar tu báscula Bluetooth Swisshome con Home Assistant usando un ESP32 y ESPHome, para hacer seguimiento de tu peso y métricas corporales sin depender de aplicaciones externas.
author: Jamal
date: 2025-03-23 11:33:00 +0800
categories: [Home Assistant, Automatización del Hogar, ESP32]
tags: [Bluetooth, Báscula, ESPHome, Home Assistant, Automatización, Tecnología]
pin: true

image:
  path: /assets/images/bascula-bluetooth-swisshome.png
  alt: Integración de una báscula Bluetooth Swisshome con ESP32 y Home Assistant.
---

# Cómo integrar una báscula Bluetooth Swisshome en Home Assistant con ESPHome

¡Hola a todos!

Hace unos meses, compré una báscula de baño marca Swisshome, una opción económica y genérica que me costó apenas 10 €. Aunque al principio solo la usaba para pesarnos ocasionalmente, un día revisé el manual (algo que admito no suelo hacer) y me di cuenta de que esta báscula tenía Bluetooth integrado.

Esto despertó mi curiosidad, ya que no me gusta instalar muchas aplicaciones en mi móvil, y pensé: ¿por qué no integrarla directamente con Home Assistant? De esta forma podría llevar un seguimiento del peso de toda mi familia sin depender de aplicaciones de terceros. Además, como entusiasta de la automatización y la tecnología, me pareció un desafío interesante.

En este artículo te mostraré cómo logré esta integración utilizando un ESP32 y ESPHome. También compartiré los pasos para procesar los datos emitidos por la báscula, que incluyen tanto el peso como la impedancia corporal. Si tienes una báscula Bluetooth similar, este tutorial puede ser útil para ti.

## ¿Por qué integrar una báscula Bluetooth con Home Assistant?

Si estás familiarizado con Home Assistant, sabes lo poderosa que es esta plataforma de automatización del hogar. Integrar dispositivos como una báscula Bluetooth tiene varias ventajas:

- **Seguimiento a largo plazo**: Puedes registrar el peso e incluso métricas corporales como la impedancia (útil para calcular la composición corporal).
- **Notificaciones**: Configura alertas para mantenerte al tanto de los cambios en tu peso o el de tu familia.
- **Independencia de aplicaciones externas**: Muchas básculas Bluetooth requieren apps específicas que recopilan tus datos. Con esta integración, tienes control total sobre tu información.
- **Automatizaciones personalizadas**: Por ejemplo, enviar un resumen semanal de tus progresos o activar recordatorios si detecta un cambio drástico en el peso.

## Paso 1: Descubriendo las capacidades de la báscula Swisshome

Al analizar las emisiones Bluetooth de la báscula, descubrí que funcionaba mediante broadcast, lo que significa que no es necesario establecer una conexión activa. En cambio, los datos se transmiten de forma abierta para que cualquier dispositivo pueda capturarlos, siempre que sepa cómo decodificarlos.

Esta báscula emite dos tipos de datos:

- **Peso**: Se transmite en paquetes identificados por el tipo 0xAD. Incluye el peso en gramos y un estado que confirma si la medición está lista.
- **Impedancia corporal**: Se transmite en un paquete diferente (0xA6) y aparece solo después de que el peso ha sido confirmado. La impedancia es clave para calcular métricas como el porcentaje de grasa corporal.

Para obtener esta información, utilicé herramientas como nRF Connect (en mi móvil) para capturar los paquetes Bluetooth y Wireshark para analizarlos en detalle. Fue fascinante ver cómo los datos crudos cobraban sentido tras un poco de decodificación.

## Paso 2: Decodificar los datos Bluetooth

Los datos transmitidos por la báscula están cifrados mediante un byte XOR basado en el ID del fabricante. Esto significa que para interpretarlos correctamente, primero debemos aplicar una operación XOR a los bytes relevantes y luego validar el checksum.

### Peso

El peso se encuentra codificado en los primeros cuatro bytes del paquete 0xAD. Después de decodificarlo, se convierte a kilogramos dividiendo el valor por 1000.

### Impedancia

La impedancia se transmite en los dos primeros bytes del paquete 0xA6. Este valor se procesa para obtener el dato en ohmios, que es esencial para estimar la composición corporal.

## Paso 3: Configuración del ESP32 con ESPHome

El ESP32 es una herramienta increíble para proyectos Bluetooth como este. Es barato, fácil de usar y perfectamente compatible con ESPHome, lo que simplifica su integración con Home Assistant.

A continuación, te comparto el código completo que utilicé para configurar el ESP32:

```yaml
esphome:
  name: bascula_bluetooth
  platform: ESP32
  board: esp32dev

esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms
    window: 1100ms
    active: true
  on_ble_advertise:
    - mac_address:
        - A0:91:63:4B:FB:C0  # Dirección MAC de la báscula
      then:
        - lambda: |-
            const uint16_t company_id = 0xA0AC;
            int xor_key = company_id >> 8;
            for (auto data : x.get_manufacturer_datas()) {
              if (data.data.size() >= 12) {
                std::vector<uint8_t> buf(6);
                for (int i = 0; i < 6; i++) {
                  buf[i] = data.data[i + 6] ^ xor_key;
                }
                int chk = 0;
                for (int i = 0; i < 5; i++) {
                  chk += buf[i];
                }
                if ((chk & 0x1F) != (buf[5] & 0x1F)) return;

                uint8_t packet_type = buf[4];
                if (packet_type == 0xAD) {  // Peso
                  int grams = buf[0] | (buf[1] << 8) | (buf[2] << 16) | (buf[3] << 24);
                  float weight_kg = grams / 1000.0;
                  id(weight_sensor).publish_state(weight_kg);
                } else if (packet_type == 0xA6) {  // Impedancia
                  int fat_raw = buf[0] | (buf[1] << 8);
                  float impedancia = fat_raw / 10.0;
                  id(body_impedancia_sensor).publish_state(impedancia);
                }
              }
            }

sensor:
  - platform: template
    name: "Peso Báscula"
    id: weight_sensor
    unit_of_measurement: "kg"
    accuracy_decimals: 2
    icon: "mdi:weight-kilogram"

  - platform: template
    name: "Impedancia (Z)"
    id: body_impedancia_sensor
    unit_of_measurement: "Ω"
    accuracy_decimals: 1
    icon: "mdi:scale-bathroom"
```
## Paso 4: Integración con Home Assistant
Una vez configurado el ESP32, Home Assistant detecta automáticamente los datos. Creé sensores personalizados para mostrar tanto el peso como la impedancia. También añadí automatizaciones, como notificaciones al registrar un nuevo peso o alertas cuando detecte valores fuera de lo esperado.

## Conclusión
Este proyecto me permitió aprovechar al máximo una báscula económica y genérica, transformándola en una herramienta útil y completamente integrada en mi sistema domótico. Además, me ayudó a entender mejor el funcionamiento de las emisiones Bluetooth y a trabajar con ESPHome para procesarlas.

Si tienes una báscula Bluetooth y te interesa un proyecto similar, ¡anímate a intentarlo! Y si tienes preguntas o ideas para mejorar este tutorial, puedes enviarmo un correo al info@belkacemi.es.