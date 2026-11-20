# Radiacion/DSP_proyecto
Repositorio para proyectp
Codigo completo de la cantonera electronica :
#include <SPI.h>
#include <MFRC522.h>
#include <Adafruit_Fingerprint.h>
#include <HardwareSerial.h>

// RFID
#define SS_PIN 5
#define RST_PIN 4

MFRC522 rfid(SS_PIN, RST_PIN);

// SENSOR DE HUELLA
HardwareSerial mySerial(2);
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);

// PINES
#define RELE 21
#define LED_ROJO 33
#define LED_VERDE 26
#define LED_AZUL 27
#define BUZZER 25

// PINES DE SENAL PARA HUELLA
#define PIN_REGISTRO_OK   22   // se activa al agregar huella correctamente
#define PIN_NO_REGISTRADA 32   // se activa si la huella NO esta registrada (cambia si quieres otro)
// La huella registrada activa el pin 25 (BUZZER) mediante beep()

#define TIEMPO_PIN_REGISTRO 1000

// CONFIGURACION
#define RELE_ON LOW
#define RELE_OFF HIGH

#define RGB_ANODO_COMUN false

#define TIEMPO_APERTURA 3000

// TARJETA DE ACCESO PERMITIDO
byte UID_USER[4] = {
  0x03, 0x0E, 0x89, 0x23
};

// TARJETA ESPECIAL
byte UID_ADMIN[4] = {
  0xB0, 0x44, 0xA0, 0x04
};

bool sensorHuellaOK = false;

// Prototipos
void procesarRFID();
void procesarHuella();
void registrarHuella();
void accesoPermitido();
void accesoAdministrador();
void accesoDenegado();
void abrirCantonera();
void encenderLED(bool rojo, bool verde, bool azul);
void apagarLED();
void beep(int cantidad);
void esperarRetirarDedo();

// =================================
// CONFIGURACION INICIAL
// =================================

void setup() {

  Serial.begin(115200);

  pinMode(RELE, OUTPUT);
  pinMode(LED_ROJO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  pinMode(LED_AZUL, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(PIN_REGISTRO_OK, OUTPUT);
  pinMode(PIN_NO_REGISTRADA, OUTPUT);

  digitalWrite(RELE, RELE_OFF);
  digitalWrite(BUZZER, LOW);
  digitalWrite(PIN_REGISTRO_OK, LOW);
  digitalWrite(PIN_NO_REGISTRADA, LOW);

  apagarLED();

  // RFID
  SPI.begin(18, 19, 23, 5);
  rfid.PCD_Init();

  // HUELLA
  mySerial.begin(57600, SERIAL_8N1, 16, 17);
  finger.begin(57600);

  if (finger.verifyPassword()) {
    sensorHuellaOK = true;
    Serial.println("Sensor de huella conectado");
  } else {
    Serial.println("ERROR: Sensor de huella no detectado");
  }

  Serial.println("-------------------------");
  Serial.println("SISTEMA DE ACCESO INICIADO");
  Serial.println("Acerque tarjeta o coloque dedo");
  Serial.println("Comandos serial:");
  Serial.println("  e = registrar huella nueva");
  Serial.println("  b = borrar todas las huellas");
  Serial.println("-------------------------");
}

// =================================
// BUCLE PRINCIPAL
// =================================

void loop() {

  // Comandos por Monitor Serial
  if (Serial.available()) {

    char c = Serial.read();

    if (c == 'e' || c == 'E') {
      registrarHuella();
    }
    else if (c == 'b' || c == 'B') {
      if (sensorHuellaOK) {
        finger.emptyDatabase();
        Serial.println("TODAS LAS HUELLAS BORRADAS");
      }
    }
  }

  if (rfid.PICC_IsNewCardPresent() &&
      rfid.PICC_ReadCardSerial()) {

    procesarRFID();

    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();

    delay(300);
  }

  if (sensorHuellaOK) {
    procesarHuella();
  }
}

// =================================
// COMPARAR UID
// =================================

bool compararUID(byte tarjeta[]) {

  if (rfid.uid.size != 4) {
    return false;
  }

  for (byte i = 0; i < 4; i++) {

    if (rfid.uid.uidByte[i] != tarjeta[i]) {
      return false;
    }
  }

  return true;
}

// =================================
// PROCESAR RFID
// =================================

void procesarRFID() {

  Serial.print("TARJETA DETECTADA: ");

  for (byte i = 0; i < rfid.uid.size; i++) {

    if (rfid.uid.uidByte[i] < 0x10) {
      Serial.print("0");
    }

    Serial.print(rfid.uid.uidByte[i], HEX);
    Serial.print(" ");
  }

  Serial.println();

  if (compararUID(UID_ADMIN)) {

    Serial.println("TARJETA ESPECIAL");
    accesoAdministrador();

  }

  else if (compararUID(UID_USER)) {

    Serial.println("ACCESO PERMITIDO");
    accesoPermitido();

  }

  else {

    Serial.println("TARJETA NO REGISTRADA");
    accesoDenegado();
  }
}

// =================================
// PROCESAR HUELLA
// =================================

void procesarHuella() {

  uint8_t p = finger.getImage();

  if (p != FINGERPRINT_OK) {
    return;
  }

  Serial.println("HUELLA DETECTADA");

  p = finger.image2Tz();

  if (p != FINGERPRINT_OK) {

    accesoDenegado();
    esperarRetirarDedo();
    return;
  }

  p = finger.fingerSearch();

  if (p == FINGERPRINT_OK) {

    Serial.print("HUELLA RECONOCIDA. ID: ");
    Serial.println(finger.fingerID);

    accesoPermitido();   // activa pin 25 (beep)

  } else {

    Serial.println("HUELLA NO REGISTRADA");

    digitalWrite(PIN_NO_REGISTRADA, HIGH);
    accesoDenegado();
    digitalWrite(PIN_NO_REGISTRADA, LOW);
  }

  esperarRetirarDedo();
}

// =================================
// REGISTRO DE HUELLAS
// =================================

int buscarIDLibre() {

  for (int id = 1; id <= 127; id++) {
    if (finger.loadModel(id) != FINGERPRINT_OK) {
      return id;   // este ID esta vacio
    }
  }

  return -1;       // memoria llena
}

bool capturarImagen(uint8_t slot) {

  unsigned long inicio = millis();

  while (finger.getImage() != FINGERPRINT_OK) {

    if (millis() - inicio > 10000) {
      return false;   // 10 s sin dedo
    }

    delay(50);
  }

  return finger.image2Tz(slot) == FINGERPRINT_OK;
}

void registrarHuella() {

  if (!sensorHuellaOK) {
    Serial.println("Sensor no disponible");
    return;
  }

  int id = buscarIDLibre();

  if (id == -1) {
    Serial.println("MEMORIA LLENA");
    accesoDenegado();
    return;
  }

  Serial.print("REGISTRANDO HUELLA EN ID ");
  Serial.println(id);

  encenderLED(0, 0, 1);

  // Primera captura
  Serial.println("Coloque el dedo...");

  if (!capturarImagen(1)) {
    Serial.println("ERROR en primera captura");
    accesoDenegado();
    return;
  }

  Serial.println("Retire el dedo");
  delay(1000);
  esperarRetirarDedo();

  // Segunda captura
  Serial.println("Coloque el mismo dedo otra vez...");

  if (!capturarImagen(2)) {
    Serial.println("ERROR en segunda captura");
    accesoDenegado();
    return;
  }

  // Crear y guardar modelo
  if (finger.createModel() != FINGERPRINT_OK) {
    Serial.println("Las huellas no coinciden");
    accesoDenegado();
    esperarRetirarDedo();
    return;
  }

  if (finger.storeModel(id) == FINGERPRINT_OK) {

    Serial.print("HUELLA GUARDADA. ID: ");
    Serial.println(id);

    encenderLED(0, 1, 0);
    digitalWrite(PIN_REGISTRO_OK, HIGH);
    beep(3);
    delay(TIEMPO_PIN_REGISTRO);
    digitalWrite(PIN_REGISTRO_OK, LOW);
    apagarLED();

  } else {

    Serial.println("ERROR al guardar la huella");
    accesoDenegado();
  }

  esperarRetirarDedo();
}

// =================================
// ACCESO PERMITIDO
// =================================

void accesoPermitido() {

  Serial.println("ACCESO PERMITIDO");

  encenderLED(0, 1, 0);

  beep(1);

  abrirCantonera();

  apagarLED();
}

// =================================
// ACCESO ESPECIAL
// =================================

void accesoAdministrador() {

  Serial.println("ACCESO ESPECIAL");

  encenderLED(0, 0, 1);

  beep(2);

  abrirCantonera();

  apagarLED();
}

// =================================
// ACCESO DENEGADO
// =================================

void accesoDenegado() {

  Serial.println("ACCESO DENEGADO");

  encenderLED(1, 0, 0);

  beep(2);

  delay(1000);

  apagarLED();
}

// =================================
// CONTROL DE CANTONERA
// =================================

void abrirCantonera() {

  Serial.println("CANTONERA ABIERTA");

  digitalWrite(RELE, RELE_ON);

  delay(TIEMPO_APERTURA);

  digitalWrite(RELE, RELE_OFF);

  Serial.println("CANTONERA CERRADA");
}

// =================================
// CONTROL LED RGB
// =================================

void encenderLED(bool rojo, bool verde, bool azul) {

  digitalWrite(LED_ROJO,
    RGB_ANODO_COMUN ? !rojo : rojo);

  digitalWrite(LED_VERDE,
    RGB_ANODO_COMUN ? !verde : verde);

  digitalWrite(LED_AZUL,
    RGB_ANODO_COMUN ? !azul : azul);
}

void apagarLED() {

  encenderLED(0, 0, 0);
}

// =================================
// BUZZER
// =================================

void beep(int cantidad) {

  for (int i = 0; i < cantidad; i++) {

    digitalWrite(BUZZER, HIGH);
    delay(120);

    digitalWrite(BUZZER, LOW);
    delay(150);
  }
}

// =================================
// ESPERAR RETIRAR DEDO
// =================================

void esperarRetirarDedo() {

  while (finger.getImage() != FINGERPRINT_NOFINGER) {
    delay(50);
  }
}
