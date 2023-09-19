
# GPS-TRACKER-AND-SOS-BUTTON-WITH-RECORDING

This is a Multipurpose GPS tracker project and it can act like an audio spy
gadget where you can listen to conversations secretly over cellular
network, this can also act as an SOS button whereby the just press of a button
you can share your current location via SMS and also it will make a call on
that SOS number and can track the location of this A9G board by just sending one SMS. Yes
by sending an SMS this board will return a Google Maps link with it’s
current location in it.

## Demo

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/jYwSQYcKL9Y/0.jpg)](https://www.youtube.com/watch?v=jYwSQYcKL9Y)



## click on the download button to download required files



[![AGPL License](https://img.shields.io/badge/downloads-fdt?logo=%F0%9F%98%8A&label=code&labelColor=grey&color=green
)](https://github.com/thinkerrobotics/GPS-TRACKER-AND-SOS-BUTTON-WITH-RECORDING/archive/refs/heads/main.zip)

## circuit
![AGPL License](https://github.com/thinkerrobotics/GPS-TRACKER-AND-SOS-BUTTON-WITH-RECORDING/assets/90499260/fdcdf9c3-1c4d-4b58-8048-3d8ae1d5039f)




## code
# BEFORE UPLOAD THE CODE DO THE MDIFICATIONS AS FOLLOWS





- MAKE SURE YOU INSTALL ESP32 BOARDS I  YOUR ARDUNO
- Necessary LIBRARIES
- CHANGE THE MOBILE NUMBER
- CHANGE SOS_Time
- CHANGE THE PIN NUMBERS ACCORDING TO YUR NEED








```C++
 #include "WiFi.h"

#define SOS D3
#define SLEEP_PIN D2 // Make this pin HIGH to make A9G board to go to sleep mode

String SOS_NUM = "+91xxxxxxxxxx"; // Add a number on which you want to receive call or SMS

int SOS_Time = 5; // Press the button 5 sec

// Necessary Variables
boolean stringComplete = false;
String inputString = "";
String fromGSM = "";
bool CALL_END = 1;
char* response = " ";
String res = "";
int c = 0;

void setup()
{
  Serial.begin(115200);                      // For Serial Monitor
  Serial1.begin(115200, SERIAL_8N1, D0, D1); // For A9G Board


  // Making Radio OFF for power saving
  WiFi.mode(WIFI_OFF);  // WiFi OFF
  btStop();   // Bluetooth OFF

  pinMode(SOS, INPUT_PULLUP);
  pinMode(SLEEP_PIN, OUTPUT);

  // Waiting for A9G to setup everything for 20 sec
  delay(20000);


  digitalWrite(SLEEP_PIN, LOW); // Sleep Mode OFF

  Serial1.println("AT");               // Just Checking
  delay(1000);

  Serial1.println("AT+GPS = 1");      // Turning ON GPS
  delay(1000);

  Serial1.println("AT+GPSLP = 2");      // GPS low power
  delay(1000);

  Serial1.println("AT+SLEEP = 1");    // Configuring Sleep Mode to 1
  delay(1000);

  Serial1.println("AT+CMGF = 1");
  delay(1000);

  Serial1.println("AT+CSMP  = 17,167,0,0 ");
  delay(1000);

  Serial1.println("AT+CPMS = \"SM\",\"ME\",\"SM\" ");
  delay(1000);
  
  digitalWrite(SLEEP_PIN, HIGH); // Sleep Mode ON
}

void loop()
{

  {
    //listen from GSM Module
    if (Serial1.available())
    {
      char inChar = Serial1.read();

      if (inChar == '\n') {

        //check the state
        if (fromGSM == "SEND LOCATION\r")
        {
          Get_gmap_link(0);  // Send Location without Call
          digitalWrite(SLEEP_PIN, HIGH);// Sleep Mode ON
          
        }

        else if (fromGSM == "RING\r")
        {
          digitalWrite(SLEEP_PIN, LOW); // Sleep Mode OFF
          Serial.println("---------ITS RINGING-------");
          Serial1.println("ATA");
        }

        else if (fromGSM == "NO CARRIER\r")
        {
          Serial.println("---------CALL ENDS-------");
          CALL_END = 1;
          digitalWrite(SLEEP_PIN, HIGH);// Sleep Mode ON
        }

        //write the actual response
        Serial.println(fromGSM);
        //clear the buffer
        fromGSM = "";

      }
      else
      {
        fromGSM += inChar;
      }
      delay(20);
    }

    // read from port 0, send to port 1:
    if (Serial.available()) {
      int inByte = Serial.read();
      Serial1.write(inByte);
    }

    // When SOS button is pressed
    if (digitalRead(SOS) == LOW && CALL_END == 1)
    {
      Serial.print("Calling In.."); // Waiting for 5 sec
      for (c = 0; c < SOS_Time; c++)
      {
        Serial.println((SOS_Time - c));
        delay(1000);
        if (digitalRead(SOS) == HIGH)
          break;
      }

      if (c == 5)
      {
        Get_gmap_link(1);  // Send Location with Call
      }

      //only write a full message to the GSM module
      if (stringComplete)
      {
        Serial1.print(inputString);
        inputString = "";
        stringComplete = false;
      }

    }
  }
}


//---------------------------------------------  Getting Location and making Google Maps link of it. Also making call if needed
void Get_gmap_link(bool makeCall)
{


  digitalWrite(SLEEP_PIN, LOW);
  delay(1000);
  Serial1.println("AT+LOCATION = 2");
  Serial.println("AT+LOCATION = 2");

  while (!Serial1.available());
  while (Serial1.available())
  {
    char add = Serial1.read();
    res = res + add;
    delay(1);
  }

  res = res.substring(17, 38);
  response = &res[0];

  Serial.print("Recevied Data - "); Serial.println(response); // printin the String in lower character form
  Serial.println("\n");

  if (strstr(response, "GPS NOT"))
  {
    Serial.println("No Location data");
    //------------------------------------- Sending SMS without any location
    Serial1.println("AT+CMGF=1");
    delay(1000);
    Serial1.println("AT+CMGS=\"" + SOS_NUM + "\"\r");
    delay(1000);

    Serial1.println ("Unable to fetch location. Please try again");
    delay(1000);
    Serial1.println((char)26);
    delay(1000);
  }
  else
  {

    int i = 0;
    while (response[i] != ',')
      i++;

    String location = (String)response;
    String lat = location.substring(2, i);
    String longi = location.substring(i + 1);
    Serial.println(lat);
    Serial.println(longi);

    String Gmaps_link = ( "http://maps.google.com/maps?q=" + lat + "+" + longi); //http://maps.google.com/maps?q=38.9419+-78.3020
    //------------------------------------- Sending SMS with Google Maps Link with our Location
    Serial1.println("AT+CMGF=1");
    delay(1000);
    Serial1.println("AT+CMGS=\"" + SOS_NUM + "\"\r");
    delay(1000);

    Serial1.println ("I'm here " + Gmaps_link);
    delay(1000);
    Serial1.println((char)26);
    delay(1000);

    Serial1.println("AT+CMGD=1,4"); // delete stored SMS to save memory
    delay(5000);
  }
  response = "";
  res = "";
  if (makeCall)
  {
    Serial.println("Calling Now");
    Serial1.println("ATD" + SOS_NUM);
    CALL_END = 0;
  }
}

```


## Authors

- [@techiesms](https://github.com/techiesms)

