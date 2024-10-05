# Medicine-Reminder-System-Using-Blynk-App
#include <Wire.h>
#include <RTClib.h>
#include <LiquidCrystal_I2C.h>

// Enter your Blynk auth token
#define BLYNK_TEMPLATE_ID "TMPL3yWZqS0px"
#define BLYNK_TEMPLATE_NAME "medication reminder"
#define BLYNK_AUTH_TOKEN "jk7swRi3W-2PrsgwJ8vfDT6uGWZNU_kg"

// Include the library files
#define BLYNK_PRINT Serial
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

// Enter Wifi credentials
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "Ani"; // Enter your WIFI name
char pass[] = "ananya123"; // Enter your WIFI password

// Set the LCD address to 0x27 for a 16 chars and 2 line display
LiquidCrystal_I2C lcd(0x27, 16, 2);

const int buttonPin = D5;
const int buzzerPin = D6;

RTC_DS3231 rtc;
char t[32];

// Define the times you want to trigger the alert
const int numAlertTimes = 3; // Change this to the number of alert times you want
const int alertHours[numAlertTimes] = {12, 12, 18}; // Hours of the alert times
const int alertMinutes[numAlertTimes] = {0, 5, 45}; // Minutes of the alert times
String medicines[numAlertTimes] = {"Aspirin", "Paracetamol", "Cetirizine"};
int alerted[numAlertTimes] = {0, 0, 0}; // has the alert already been sent for these times

void setup() {
  Serial.begin(9600);

  Wire.begin(2, 0);
  lcd.init();
  lcd.backlight();
  lcd.print("* WELCOME *");
  delay(1000);

  pinMode(buzzerPin, OUTPUT);
  pinMode(buttonPin, INPUT_PULLUP);
  digitalWrite(buzzerPin, LOW);

  lcd.clear(); // Clear the LCD
  lcd.setCursor(0, 0); // Set cursor to first row
  lcd.print("Connecting WiFi");
  Serial.println("Connecting WiFi....");
  lcd.setCursor(0, 1); // Set cursor to second row
  lcd.print("Re-Start Hotspot");
  delay(2000);

  // Initialize the Blynk library
  Blynk.begin(auth, ssid, pass, "blynk.cloud", 80);

  lcd.clear(); // Clear the LCD
  lcd.setCursor(0, 0); // Set cursor to first row
  lcd.print("WiFi Connected");
  lcd.setCursor(0, 1); // Set cursor to second row
  lcd.print("Successfully");
  Serial.println("WiFi Connected Successfully.");
  delay(2000);

  if (!rtc.begin()) {
    Serial.println("RTC module is NOT found");
    Serial.flush();
    while (1);
  }
  rtc.adjust(DateTime(2024, 3, 9, 11, 59, 45));
}

void loop() {
  DateTime now = rtc.now();
  sprintf(t, "%02d:%02d:%02d %02d/%02d/%02d", now.hour(), now.minute(), now.second(), now.day(), now.month(), now.year());
  Serial.print(F("Date/Time: "));
  Serial.println(t);

  lcd.clear(); // Clear the LCD
  lcd.setCursor(0, 0); // Set cursor to first row
  lcd.print("TIME : " + String(now.hour()) + ":" + String(now.minute()) + ":" + String(now.second()));

  // Check if it's time to trigger any of the alerts
  for (int i = 0; i < numAlertTimes; i++) {
    if ((now.hour() == alertHours[i]) && (now.minute() == alertMinutes[i]) && (alerted[i] == 0)) {
      Serial.print(F("FOUND "));
      alert(medicines[i]);
      alerted[i] = 1;
      break; // Exit the loop if an alert time is found
    } else if (alerted[0] == 0) {
      lcd.setCursor(0, 1); // Set cursor to second row
      lcd.print("Next Alarm " + String(alertHours[0]) + ":" + String(alertMinutes[0]));
    } else if (alerted[1] == 0) {
      lcd.setCursor(0, 1); // Set cursor to second row
      lcd.print("Next Alarm " + String(alertHours[1]) + ":" + String(alertMinutes[1]));
    } else if (alerted[2] == 0) {
      lcd.setCursor(0, 1); // Set cursor to second row
      lcd.print("Next Alarm " + String(alertHours[2]) + ":" + String(alertMinutes[2]));
    }
  }

  delay(1000);
}

void alert(String medicineName) {
  Serial.print(F("ALERT "));
  lcd.clear(); // Clear the LCD
  lcd.setCursor(0, 0); // Set cursor to first row
  lcd.print("Take medicine: "); // Print message on LCD
  lcd.setCursor(0, 1); // Set cursor to second row
  lcd.print(medicineName); // Print medicine name (change as needed)
  delay(1000);

  uint8_t temp_timer = 30;
  uint8_t previoussecond = 0;
  uint8_t currsecond = 0;
  uint8_t temp_thanks = 0;

  while (temp_timer > 0) {
    DateTime now = rtc.now();
    digitalWrite(buzzerPin, HIGH); // Turn on the buzzer

    lcd.setCursor(14, 1); // Set cursor to second row
    lcd.print(String(temp_timer));

    currsecond = now.second();

    if (previoussecond != currsecond) {
      temp_timer--;
    }

    previoussecond = currsecond;

    if (digitalRead(buttonPin) == LOW) { // If the button is pressed
      digitalWrite(buzzerPin, LOW); // Turn off the buzzer
      lcd.clear(); // Clear the LCD
      lcd.setCursor(0, 0); // Set cursor to first row
      lcd.print("Thank You Mr.Ram");
      lcd.setCursor(0, 1); // Set cursor to second row
      lcd.print("Get well soon"); // Print message on LCD
      delay(3000);
      temp_thanks = 1;
      break;
    }

    delay(100); // Small delay for debounce
  }

  if (temp_timer == 0) { // If the timer runs out
    digitalWrite(buzzerPin, LOW); // Turn off the buzzer
    lcd.clear(); // Clear the LCD
    lcd.setCursor(0, 0); // Set cursor to first row
    lcd.print("Mobile Alert");
    lcd.setCursor(0, 1); // Set cursor to second row
    lcd.print("Sending..."); // Print message on LCD
    delay(2000);
    lcd.print("Sent Successfully"); // Print message on LCD
    delay(2000);
    // Run the Blynk library
    Blynk.run();
    String str = "Patient ID.-1529 (Mr. Ram, Age-52) not taken medicine : " + medicineName;
    Blynk.logEvent("medication_reminder", str);
  }
}
