# 🐠 Aquarium Cooler Controller

## 🌟 Overview
This project implements an aquarium cooler controller using an ESP32 with ESPHome. It manages temperature and humidity, water flow, and CO2 pressure, integrating seamlessly with Home Assistant. This setup ensures your aquarium stays at optimal conditions, providing a healthy environment for aquatic life.

## 🔥 Features
- **🌡️ Temperature and Humidity Monitoring**: Uses a DHT22 sensor to monitor and control the environment.
- **💧 Water Flow Monitoring**: Pulse counter sensors measure water flow in liters per hour, gallons per minute, and total daily water consumption.
- **🧪 CO2 Pressure Monitoring**: ADC sensor measures CO2 pressure and percentage.
- **💨 Fan Speed Control**: PID controlled thermostat manages the fan speed for cooling.
- **📶 WiFi Connectivity**: Connects to your home network and integrates with Home Assistant.
- **🛡️ Fallback Hotspot**: Provides a fallback WiFi hotspot in case of connection issues.
- **⏰ Time Synchronization**: Syncs time using SNTP servers.
- **🔄 Remote Updates**: Supports over-the-air (OTA) updates.

## 🛠️ Hardware Requirements
- ESP32 Development Board
- DHT22 Sensor
- Pulse Counter Sensors
- CO2 Pressure Sensor
- 12V Fan
- Power Supply
- Connecting Wires and Breadboard

## 📐 Wiring Diagram
- **DHT22 Sensor**: 
  - Data pin to GPIO3
  - VCC to 3.3V
  - GND to GND

- **Pulse Counter Sensor**: 
  - Data pin to GPIO12
  - VCC to 5V (or as required by your sensor)
  - GND to GND

- **CO2 Pressure Sensor**: 
  - Data pin to GPIO32
  - VCC to 3.3V
  - GND to GND

- **Fan**:
  - PWM pin to GPIO16
  - VCC to 12V
  - GND to GND

## 🚀 Getting Started
1. **Add the ESPHome Configuration**: Add the following configuration to your ESPHome setup:

```yaml
esphome:
  name: "aquariumcooler"

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable Home Assistant API
api:
  encryption:
    key: "redacted"

ota:
  password: "redacted"
  
time:
  - platform: sntp
    id: sntp_time
    servers:
      - '0.pool.ntp.org'
      - '1.pool.ntp.org'
      - '2.pool.ntp.org'

wifi:
  ssid: "redacted"
  password: "redacted"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Console-Fan Fallback Hotspot"
    password: "PMVXQ3dneK4n"

captive_portal:

substitutions:
  friendly_name: Console Fan

globals:
  - id: dhttemp
    type: float
    restore_value: yes
    initial_value: '0'

logger:
  level: DEBUG
  logs: 
    climate: WARN
    dht: WARN

number:
  # KP
  - platform: template
    name: $friendly_name kp
    icon: mdi:chart-bell-curve
    restore_value: true
    min_value: 0
    max_value: 50
    step: 0.001
    set_action: 
      lambda: |- 
        id(console_thermostat).set_kp(x);

  # KI
  - platform: template
    name: $friendly_name ki
    icon: mdi:chart-bell-curve
    restore_value: true
    min_value: 0
    max_value: 50
    step: 0.0001
    set_action: 
      lambda: id(console_thermostat).set_ki(x);

  # KD
  - platform: template
    name: $friendly_name kd
    icon: mdi:chart-bell-curve
    restore_value: true
    min_value: -50
    max_value: 50
    step: 0.001
    set_action: 
      lambda: id(console_thermostat).set_kd(x);

text_sensor:
  # Send IP Address
  - platform: wifi_info
    ip_address:
      name: $friendly_name IP Address

  # Send Uptime in raw seconds
  - platform: template
    name: $friendly_name Uptime
    id: uptime_human
    icon: mdi:clock-start

sensor:
  # Send WiFi signal strength & uptime to HA
  - platform: wifi_signal
    name: "WiFi Signal Strength"
    update_interval: 60s

  # Pulse counter for water flow
  - platform: pulse_counter
    pin: GPIO12
    id: water_pulse
    update_interval: 30s
    name: "Pulse_Meter"
    count_mode:
      rising_edge: INCREMENT
      falling_edge: DISABLE

  - platform: template
    name: "Pulse_Meter_flow"
    unit_of_measurement: "L/hr"
    accuracy_decimals: 1
    update_interval: 5s
    lambda: return (id(water_pulse).state / 350.0) * 60.0;
    id: pulse_meter_flow

  - platform: template
    name: "Pulse_Meter_percentage"
    lambda: return (id(pulse_meter_flow).state / 420.0) * 100.0;
    unit_of_measurement: "%"
    accuracy_decimals: 0

  - platform: template
    name: "Pulse Meter Flow Gallons"
    unit_of_measurement: "gal/min"
    accuracy_decimals: 1
    update_interval: 5s
    lambda: return (id(water_pulse).state / 350.0) * 60.0 / 22.1724;
    id: pulse_meter_flow_gallons

  - platform: integration
    name: "Water Flow Integration"
    sensor: pulse_meter_flow_gallons
    time_unit: min
    id: water_flow_integration

  - platform: total_daily_energy
    name: "Total Daily Water Consumption"
    power_id: water_flow_integration
    id: total_daily_water_consumption

  # CO2 Pressure sensor
  - platform: adc
    name: "co2 Pressure"
    pin: GPIO32
    id: pressure_adc
    update_interval: 1s
    unit_of_measurement: "V"
    accuracy_decimals: 2
    attenuation: 12db
    filters:
      - sliding_window_moving_average:
          window_size: 60
          send_every: 30

  - platform: template
    name: "co2 Pressure"
    id: pressure
    unit_of_measurement: "PSI"
    accuracy_decimals: 2
    lambda: |-
      return id(pressure_adc).state * 600.0 / 3.3;
    update_interval: 1s

  - platform: template
    name: "co2 Pressure Percentage"
    id: co2_pressure_percentage
    unit_of_measurement: "%"
    accuracy_decimals: 2
    lambda: |-
      return (id(pressure_adc).state * 500.0 / 3.3) / 500.0 * 100.0;
    update_interval: 1s

  # Human readable uptime
  - platform: uptime
    name: $friendly_name Uptime
    id: uptime_sensor
    update_interval: 60s
    on_raw_value:
      then:
        - text_sensor.template.publish:
            id: uptime_human
            state: !lambda |-
              int seconds = round(id(uptime_sensor).raw_state);
              int days = seconds / (24 * 3600);
              seconds = seconds % (24 * 3600);
              int hours = seconds / 3600;
              seconds = seconds % 3600;
              int minutes = seconds /  60;
              seconds = seconds % 60;
              return (
                (days ? to_string(days) + "d " : "") +
                (hours ? to_string(hours) + "h " : "") +
                (minutes ? to_string(minutes) + "m " : "") +
                (to_string(seconds) + "s")
              ).c_str();

  # Fan controller setup
  - platform: template
    name: $friendly_name p term
    id: p_term
    unit_of_measurement: "%"
    accuracy_decimals: 2

  - platform: template
    name: $friendly_name i term
    id: i_term
    unit_of_measurement: "%"
    accuracy_decimals: 2

  - platform: template
    name: $friendly_name d term
    id: d_term
    unit_of_measurement: "%"
    accuracy_decimals: 2

  - platform: template
    name: $friendly_name output value
    unit_of_measurement: "%"
    id: o_term


    accuracy_decimals: 2

  - platform: template
    name: $friendly_name error value
    id: e_term
    accuracy_decimals: 2

  - platform: template
    name: $friendly_name zero 
    id: zero_value
    update_interval: 60s
    lambda: |-
      return 0;

  - platform: template
    name: $friendly_name zero percent
    unit_of_measurement: "%"
    id: zero_value_percent
    update_interval: 60s
    lambda: |-
      return 0;

  # GET TEMP/HUMIDITY FROM DHT22
  - platform: dht
    pin: GPIO3
    model: DHT22
    temperature:
      name: "Temperature"
      id: console_fan_temperature
      accuracy_decimals: 3
      filters:
         - exponential_moving_average:  
             alpha: 0.1
             send_every: 1
    humidity:
      name: "Humidity"
      id: console_fan_humidity
    update_interval: 5s

  # Take the "COOL" value of the pid and send it to the frontend to graph the output voltage
  - platform: pid
    name: "Fan Speed (PWM Voltage)"
    climate_id: console_thermostat
    type: COOL

output:
  # Wire this pin (GPIO16) into the PWM pin of your 12v fan
  - platform: ledc
    id: console_fan_speed
    pin: GPIO16
    frequency: "25000 Hz" 
    min_power: 0%
    max_power: 100%

fan:
  - platform: speed
    output: console_fan_speed
    name: "Console Fan Speed"

climate:
  - platform: pid
    name: "Console Fan Thermostat"
    id: console_thermostat
    sensor: console_fan_temperature
    default_target_temperature: 22°C
    cool_output: console_fan_speed
    
    on_state:
      - sensor.template.publish:
          id: p_term
          state: !lambda 'return -id(console_thermostat).get_proportional_term() * 100.0;'
      - sensor.template.publish:
          id: i_term
          state: !lambda 'return -id(console_thermostat).get_integral_term()* 100.0;'
      - sensor.template.publish:
          id: d_term
          state: !lambda 'return -id(console_thermostat).get_derivative_term()* 100.0;'
      - sensor.template.publish:
          id: o_term
          state: !lambda 'return -id(console_thermostat).get_output_value()* 100.0;'
      - sensor.template.publish:
          id: e_term
          state: !lambda 'return -id(console_thermostat).get_error_value();'

    visual:
      min_temperature: 10 °C
      max_temperature: 50 °C

    control_parameters:
      kp: 0.74116
      ki: 0.00602
      kd: 0.81428
      max_integral: 0.0

switch:
  - platform: restart
    name: "Console Fan ESP32 Restart"

button:
  - platform: template
    name: "PID Climate Autotune"
    on_press:
      - climate.pid.autotune: console_thermostat
```

## 📚 Documentation
For more detailed information on ESPHome configurations and integrations, refer to the [ESPHome Documentation](https://esphome.io/).

## 🤝 Contributing
We welcome contributions to this project! Feel free to open issues or submit pull requests on GitHub. Together, we can improve and expand the capabilities of the Aquarium Cooler Controller Project.

## 📧 Contact
If you have any questions or need further assistance, please open an issue on GitHub or contact me at prokyle123@gmail.com
