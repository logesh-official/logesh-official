#include <Arduino.h>
#include <GSM.h> // Include GSM library for SMS communication
#include <GPS.h> // Include GPS library for location tracking

// Define pins for sensors and output
const int ldrPin = A0; // Analog pin for LDR sensor
const int relayPin = 12; // Digital pin for relay control
const int gpsTxPin = 2; // GPS receiver TX pin
const int gpsRxPin = 3; // GPS receiver RX pin

// Define variables for fault detection
int ldrValue = 0; // Stores LDR sensor reading
int faultStatus = 0; // Flag for fault detection (0: No fault, 1: Fault detected)

// Define variables for location tracking
GPS gps(gpsRxPin, false); // GPS object with RX pin and auto-detect baud rate
double latitude = 0.0; // Stores latitude value
double longitude = 0.0; // Stores longitude value

// Initialize GSM module
GSM gsm(SIM800); // GSM object with SIM800 modem

void setup() {
  // Initialize serial communication
  Serial.begin(9600);

  // Initialize LDR sensor
  pinMode(ldrPin, INPUT);

  // Initialize relay pin as output
  pinMode(relayPin, OUTPUT);

  // Initialize GPS receiver
  gps.begin(115200);
  gps.sendCommand(PMTK_SET_NMEA_OUTPUT, PMTK_NMEA_ONLY_GLL); // Set GPS output to GLL format
}

void loop() {
  // Read LDR sensor value
  ldrValue = analogRead(ldrPin);

  // Check for fault condition (LDR value significantly low)
  if (ldrValue < 100) {
    faultStatus = 1;
  } else {
    faultStatus = 0;
  }

  // Control street light based on fault status
  if (faultStatus == 0) {
    digitalWrite(relayPin, HIGH); // Turn on street light
  } else {
    digitalWrite(relayPin, LOW); // Turn off street light
  }

  // If fault detected, send SMS alert with location information
  if (faultStatus == 1) {
    // Get GPS coordinates
    while (!gps.available()) {
      delay(100);
    }
    gps.parse(gps.read());
    latitude = gps.latitude();
    longitude = gps.longitude();

    // Send SMS alert
    gsm.BeginSMS("+919876543210"); // Replace with recipient's phone number
    gsm.SendMessage("Street light fault detected at location: ");
    gsm.SendMessageString("Latitude: ");
    gsm.SendFloat(latitude, 6);
    gsm.SendMessageString(" Longitude: ");
    gsm.SendFloat(longitude, 6);
    gsm.EndSMS();

    // Delay before next fault detection check
    delay(300000); // Check for faults every 5 minutes
  }
}
