# ArduinoLoRaDS18B20
Internet of things (IoT): a temperature sensor built using Arduino Uno + LoRa + Temperature sensor

**Things to use:**

1. Arduino Uno rev.3: https://www.arduino.cc/en/Main/ArduinoBoardUno
2. LoRa: http://www.microchip.com/wwwproducts/en/RN2483
3. Temperature sensor: https://www.maximintegrated.com/en/products/analog/sensors-and-sensor-interface/DS18B20.html

**On the Arduino, install the following libraries:**

1. OneWire.zip from http://www.tweaking4all.com/hardware/arduino/arduino-ds18b20-temperature-sensor/ if it is already not present in File /Examples /OneWire /sample.
2. RN2483-Arduino-Library by JP Meijers, ver 1.0.0 was tested 

**Connections/Procedure:**

1. Make the connections as shown here: https://www.thethingsnetwork.org/forum/t/how-to-build-your-first-ttn-node-arduino-rn2483/1574 However, LoRa RN2483 expects 3.3V on the UART pins. Hence, ensure that the 5V driven from Arduino goes via a voltage divider circuit to get it down to 3.3V. See http://www.instructables.com/id/Pi-Cubed-How-to-connect-a-33V-Raspberry-Pi-to-a-5V/ for more details. Note that it can be directly connected to LoRa but not adviceable, as the LoRa can be bricked.
2. Rig up the circuit as shown here: http://www.tweaking4all.com/hardware/arduino/arduino-ds18b20-temperature-sensor/
3. Some of the pins being used were modified for my needs, change them as you need.
4. Complete code is provided here: ArduinoUnoLoRaTemperatureSensor.ino

Note that you might have to provide these numbers as needed:
ABP: initABP(String addr, String AppSKey, String NwkSKey);
OTAA: initOTAA(String AppEUI, String AppKey);

**Pitfalls:**

1. The COM port is unstable

**Main code snippet of ArduinoUnoLoRaTemperatureSensor.ino**

```CPP
#include <rn2483.h>
#include <OneWire.h>

/*
Arduino + LoRa
https://www.thethingsnetwork.org/forum/t/how-to-build-your-first-ttn-node-arduino-rn2483/1574
*/
//create an instance of the rn2483 library, using the given Serial port
// https://github.com/itead/ITEADLIB_Arduino_WeeESP8266/issues/36
SoftwareSerial Serial1(11,10); /* RX:11, TX:10 */ 
rn2483 myLora(Serial1);
char dataToLoRa[9];

// http://www.tweaking4all.com/hardware/arduino/arduino-ds18b20-temperature-sensor/
OneWire  ds(A0);  // on pin A5 (a 4.7K resistor is necessary)
float savedCelsius=0;

// the setup routine runs once when you press reset:
void setup() 
{
  //output LED pin
  pinMode(13, OUTPUT);
  led_on();
  
  // Open serial communications and wait for port to open:
  Serial.begin(57600); //serial port to computer
  Serial1.begin(57600); //serial port to radio

  while(!Serial); //wait for Serial to be available - remove this line after successful test run
  Serial.println("Startup");

  initialize_radio();

  //transmit a startup message
  myLora.tx("TTN Mapper on TTN Uno node");

  led_off();
  delay(2000);
}

void initialize_radio()
{
  delay(100); //wait for the RN2483's startup message
  Serial1.flush();
  
  //print out the HWEUI so that we can register it via ttnctl
  String hweui = myLora.hweui();
  while(hweui.length() != 16)
  {
    Serial.println("Communication with RN2483 unsuccesful. Power cycle the TTN UNO board.");
    delay(10000);
    hweui = myLora.hweui();
  }
  Serial.println("When using OTAA, register this DevEUI: ");
  Serial.println(hweui);
  Serial.println("RN2483 firmware version:");
  Serial.println(myLora.sysver());

  //configure your keys and join the network
  Serial.println("Trying to join TTN");
  bool join_result = false;
  
  //ABP: initABP(String addr, String AppSKey, String NwkSKey);
  //join_result = myLora.initABP("02017201", "8D7FFEF938589D95AAD928C2E2E7E48F", "AE17E567AECC8787F749A62F5541D522");
  
  //OTAA: initOTAA(String AppEUI, String AppKey);
  //join_result = myLora.initOTAA("70B3D57ED00001A6", "A23C96EE13804963F8C2BD6285448198");

  while(!join_result)
  {
    Serial.println("Unable to join. Are your keys correct, and do you have TTN coverage?");
    delay(60000); //delay a minute before retry
    join_result = myLora.init();
  }
  Serial.println("Successfully joined TTN");
  
}

/*
http://forum.arduino.cc/index.php?topic=37391.0
*/
char * floatToString(char * outstr, double val, byte precision, byte widthp){
 char temp[16];
 byte i;

 // compute the rounding factor and fractional multiplier
 double roundingFactor = 0.5;
 unsigned long mult = 1;
 for (i = 0; i < precision; i++)
 {
   roundingFactor /= 10.0;
   mult *= 10;
 }
 
 temp[0]='\0';
 outstr[0]='\0';

 if(val < 0.0){
   strcpy(outstr,"-\0");
   val = -val;
 }

 val += roundingFactor;

 strcat(outstr, itoa(int(val),temp,10));  //prints the int part
 if( precision > 0) {
   strcat(outstr, ".\0"); // print the decimal point
   unsigned long frac;
   unsigned long mult = 1;
   byte padding = precision -1;
   while(precision--)
     mult *=10;

   if(val >= 0)
     frac = (val - int(val)) * mult;
   else
     frac = (int(val)- val ) * mult;
   unsigned long frac1 = frac;

   while(frac1 /= 10)
     padding--;

   while(padding--)
     strcat(outstr,"0\0");

   strcat(outstr,itoa(frac,temp,10));
 }

 // generate space padding
 if ((widthp != 0)&&(widthp >= strlen(outstr))){
   byte J=0;
   J = widthp - strlen(outstr);
   
   for (i=0; i< J; i++) {
     temp[i] = ' ';
   }

   temp[i++] = '\0';
   strcat(temp,outstr);
   strcpy(outstr,temp);
 }
 
 return outstr;
}

// the loop routine runs over and over again forever:
void loop() 
{
  ///////////////////////////////////////////////////////////////////////// From DS18B20 Start
  byte i;
  byte present = 0;
  byte type_s;
  byte data[12];
  byte addr[8];
  float celsius, fahrenheit;
  
  if ( !ds.search(addr)) {
    Serial.println("No more addresses.");
    Serial.println();
    ds.reset_search();
    delay(250);
    return;
  }
  
  Serial.print("ROM =");
  for( i = 0; i < 8; i++) {
    Serial.write(' ');
    Serial.print(addr[i], HEX);
  }

  if (OneWire::crc8(addr, 7) != addr[7]) {
      Serial.println("CRC is not valid!");
      return;
  }
  Serial.println();
 
  // the first ROM byte indicates which chip
  switch (addr[0]) {
    case 0x10:
      Serial.println("  Chip = DS18S20");  // or old DS1820
      type_s = 1;
      break;
    case 0x28:
      Serial.println("  Chip = DS18B20");
      type_s = 0;
      break;
    case 0x22:
      Serial.println("  Chip = DS1822");
      type_s = 0;
      break;
    default:
      Serial.println("Device is not a DS18x20 family device.");
      return;
  } 

  ds.reset();
  ds.select(addr);
  ds.write(0x44, 1);        // start conversion, with parasite power on at the end
  
  delay(1000);     // maybe 750ms is enough, maybe not
  // we might do a ds.depower() here, but the reset will take care of it.
  
  present = ds.reset();
  ds.select(addr);    
  ds.write(0xBE);         // Read Scratchpad

  Serial.print("  Data = ");
  Serial.print(present, HEX);
  Serial.print(" ");
  for ( i = 0; i < 9; i++) {           // we need 9 bytes
    data[i] = ds.read();
    Serial.print(data[i], HEX);
    Serial.print(" ");
  }
  Serial.print(" CRC=");
  Serial.print(OneWire::crc8(data, 8), HEX);
  Serial.println();

  // Convert the data to actual temperature
  // because the result is a 16 bit signed integer, it should
  // be stored to an "int16_t" type, which is always 16 bits
  // even when compiled on a 32 bit processor.
  int16_t raw = (data[1] << 8) | data[0];
  if (type_s) {
    raw = raw << 3; // 9 bit resolution default
    if (data[7] == 0x10) {
      // "count remain" gives full 12 bit resolution
      raw = (raw & 0xFFF0) + 12 - data[6];
    }
  } else {
    byte cfg = (data[4] & 0x60);
    // at lower res, the low bits are undefined, so let's zero them
    if (cfg == 0x00) raw = raw & ~7;  // 9 bit resolution, 93.75 ms
    else if (cfg == 0x20) raw = raw & ~3; // 10 bit res, 187.5 ms
    else if (cfg == 0x40) raw = raw & ~1; // 11 bit res, 375 ms
    //// default is 12 bit resolution, 750 ms conversion time
  }
  celsius = (float)raw / 16.0;
  fahrenheit = celsius * 1.8 + 32.0;
  Serial.print("  Temperature = ");
  Serial.print(celsius);
  Serial.print(" Celsius, ");
  Serial.print(fahrenheit);
  Serial.println(" Fahrenheit");
  ///////////////////////////////////////////////////////////////////////// From DS18B20 End
  if (celsius != savedCelsius)
  {
    savedCelsius = celsius;
    led_on();

    Serial.println("TXing");
    //myLora.tx("!"); //one byte, blocking function
    floatToString(dataToLoRa, celsius, 2, 0);
    Serial.println(dataToLoRa);
    myLora.tx(dataToLoRa);
    
    led_off();
    delay(200);
  }
}

void led_on()
{
  digitalWrite(13, 1);
}

void led_off()
{
  digitalWrite(13, 0);
}
```
