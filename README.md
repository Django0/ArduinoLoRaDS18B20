# ArduinoLoRaDS18B20
Internet of things (IoT): a temperature sensor built using Arduino Uno + LoRa + Temperature sensor

Things to use: 
1) Arduino Uno rev.3: https://www.arduino.cc/en/Main/ArduinoBoardUno
2) LoRa: http://www.microchip.com/wwwproducts/en/RN2483
3) Temperature sensor: https://www.maximintegrated.com/en/products/analog/sensors-and-sensor-interface/DS18B20.html

On the Arduino, install the following libraries: 
a) OneWire.zip from http://www.tweaking4all.com/hardware/arduino/arduino-ds18b20-temperature-sensor/ if it is already not present in  File > Examples > OneWire > sample. This is for the temperature sensor.
b) RN2483-Arduino-Library by JP Meijers, ver 1.0.0 was tested 

Connections/Procedure:
i) Make the connections as shown here: https://www.thethingsnetwork.org/forum/t/how-to-build-your-first-ttn-node-arduino-rn2483/1574
However, LoRa RN2483 expects 3.3V on the UART pins. Hence, ensure that the 5V driven from Arduino goes via a voltage divider circuit to get it down to 3.3V. See http://www.instructables.com/id/Pi-Cubed-How-to-connect-a-33V-Raspberry-Pi-to-a-5V/ for more details. Note that it can be directly connected to LoRa but not adviceable, as the LoRa can be bricked.
ii) Rig up the circuit as shown here: http://www.tweaking4all.com/hardware/arduino/arduino-ds18b20-temperature-sensor/
iii) Some of the pins being used were modified for my needs, change them as you need. 
iv) Complete code is provided here: ArduinoUnoLoRaTemperatureSensor.ino
Note that you might have to provide these numbers as needed:
ABP: initABP(String addr, String AppSKey, String NwkSKey);
OTAA: initOTAA(String AppEUI, String AppKey);

Pitfalls:
I) The COM port is unstable