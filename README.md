
#include <M5StickCPlus2.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <Preferences.h>
#include <cstring>
#include <time.h>

// ==================================================
// 1. CONFIGURACION
// ==================================================

const uint8_t CANAL_WIFI = 1;

// Zona horaria de España peninsular
const char* zonaHoraria =
  "CET-1CEST,M3.5.0,M10.5.0/3";

// ==================================================
// 2. DATOS RECIBIDOS
// ==================================================

typedef struct {
  float temperatura;
  float humedad;
  uint32_t secuencia;
  uint32_t horaEnvio;
} DatosSensor;

// Comprobar que la estructura ocupa 16 bytes
static_assert(sizeof(DatosSensor) == 16,
              "Error: estructura DatosSensor incorrecta");

DatosSensor datosRecibidos = {0, 0, 0, 0};

volatile bool hayDatos = false;
volatile uint32_t ultimoPaquete = 0;

portMUX_TYPE muxDatos = portMUX_INITIALIZER_UNLOCKED;

// ==================================================
// 3. MEMORIA PERSISTENTE
// ==================================================

Preferences preferencias;

// ==================================================
// 4. ESTADISTICAS DE PAQUETES
// ==================================================

uint32_t paquetesRecibidos = 0;
uint32_t paquetesPerdidos = 0;
uint32_t ultimoNumeroSecuencia = 0;

bool primeraRecepcion = true;

// Ventana movil de 100 paquetes
const uint32_t TAM_VENTANA = 100;

uint8_t ventana[TAM_VENTANA];

uint32_t inicioVentana = 0;
uint32_t cantidadVentana = 0;

uint32_t recibidosVentana = 0;
uint32_t perdidosVentana = 0;

// ==================================================
// 5. FUNCIONES DE ESTADISTICAS
// ==================================================

void reiniciarEstadisticas() {

  paquetesRecibidos = 0;
  paquetesPerdidos = 0;

  ultimoNumeroSecuencia = 0;
  primeraRecepcion = true;

  inicioVentana = 0;
  cantidadVentana = 0;

  recibidosVentana = 0;
  perdidosVentana = 0;

}

// Añadir paquete a la ventana movil

void anadirVentana(bool recibido) {

  uint32_t posicion;

  if (cantidadVentana < TAM_VENTANA) {

    posicion =
      (inicioVentana + cantidadVentana) % TAM_VENTANA;

    cantidadVentana++;

  } else {

    posicion = inicioVentana;

    // Eliminar el dato mas antiguo

    if (ventana[posicion] == 1) {
      recibidosVentana--;
    } else {
      perdidosVentana--;
    }

    inicioVentana =
      (inicioVentana + 1) % TAM_VENTANA;

  }

  ventana[posicion] = recibido ? 1 : 0;

  if (recibido) {
    recibidosVentana++;
  } else {
    perdidosVentana++;
  }

}

// Actualizar estadisticas

void actualizarEstadisticas(uint32_t secuencia) {

  // Primera recepcion

  if (primeraRecepcion) {

    primeraRecepcion = false;

    ultimoNumeroSecuencia = secuencia;

    paquetesRecibidos = 1;

    anadirVentana(true);

    return;

  }

  // Diferencia entre secuencias

  uint32_t diferencia =
    secuencia - ultimoNumeroSecuencia;

  // Paquete duplicado

  if (diferencia == 0) {
    return;
  }

  // Paquete antiguo o posible reinicio

  if (diferencia >= 0x80000000UL) {

    if (ultimoNumeroSecuencia > 10 &&
        secuencia <= 10) {

      reiniciarEstadisticas();

      primeraRecepcion = false;

      ultimoNumeroSecuencia = secuencia;

      paquetesRecibidos = 1;

      anadirVentana(true);

    }

    return;

  }

  // Contabilizar paquete recibido

  paquetesRecibidos++;

  // Calcular paquetes perdidos

  uint32_t perdidos = diferencia - 1;

  paquetesPerdidos += perdidos;

  // Actualizar ventana movil

  uint32_t perdidasAAnadir = perdidos;

  if (perdidasAAnadir > TAM_VENTANA - 1) {
    perdidasAAnadir = TAM_VENTANA - 1;
  }

  for (uint32_t i = 0; i < perdidasAAnadir; i++) {
    anadirVentana(false);
  }

  anadirVentana(true);

  ultimoNumeroSecuencia = secuencia;

}

// ==================================================
// 6. CONVERSION DEL MES
// ==================================================

int numeroMes(const char *mes) {

  const char *meses[] = {
    "Jan", "Feb", "Mar", "Apr", "May", "Jun",
    "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"
  };

  for (int i = 0; i < 12; i++) {

    if (strncmp(mes, meses[i], 3) == 0) {
      return i + 1;
    }

  }

  return 1;

}

// ==================================================
// 7. AJUSTE AUTOMATICO DEL RTC
// ==================================================

void ajustarRTC() {

  if (!M5.Rtc.isEnabled()) {

    Serial.println("ERROR: RTC no detectado");

    return;

  }

  char mesTexto[4];

  int dia, anio;
  int hora, minuto, segundo;

  sscanf(__DATE__, "%3s %d %d",
         mesTexto, &dia, &anio);

  sscanf(__TIME__, "%d:%d:%d",
         &hora, &minuto, &segundo);

  int mes = numeroMes(mesTexto);

  String compilacion =
    String(__DATE__) + " " + __TIME__;

  preferencias.begin("reloj", false);

  String compilacionAnterior =
    preferencias.getString("build", "");

  bool alimentacionCorrecta =
    M5.Rtc.getVoltLow();

  if (compilacion != compilacionAnterior ||
      !alimentacionCorrecta) {

    M5.Rtc.setDateTime({
      {anio, mes, dia},
      {hora, minuto, segundo}
    });

    preferencias.putString("build", compilacion);

    Serial.println("RTC actualizado automaticamente");

  } else {

    Serial.println("RTC conserva la hora anterior");

  }

  preferencias.end();

  auto fecha = M5.Rtc.getDateTime();

  Serial.printf(
    "Fecha RTC: %02d/%02d/%04d\n",
    fecha.date.date,
    fecha.date.month,
    fecha.date.year
  );

  Serial.printf(
    "Hora RTC: %02d:%02d:%02d\n",
    fecha.time.hours,
    fecha.time.minutes,
    fecha.time.seconds
  );

}

// ==================================================
// 8. RECEPCION ESP-NOW
// ==================================================

void alRecibirDatos(
  const esp_now_recv_info_t *info,
  const uint8_t *data,
  int longitud
) {

  // Comprobar longitud del paquete

  if (longitud != sizeof(DatosSensor)) {
    return;
  }

  DatosSensor nuevosDatos;

  memcpy(
    &nuevosDatos,
    data,
    sizeof(DatosSensor)
  );

  uint32_t tiempoActual = millis();

  portENTER_CRITICAL(&muxDatos);

  // Actualizar estadisticas

  actualizarEstadisticas(nuevosDatos.secuencia);

  // Guardar datos recibidos

  memcpy(
    &datosRecibidos,
    &nuevosDatos,
    sizeof(DatosSensor)
  );

  ultimoPaquete = tiempoActual;

  hayDatos = true;

  portEXIT_CRITICAL(&muxDatos);

}

// ==================================================
// 9. SETUP
// ==================================================

void setup() {

  Serial.begin(115200);

  Serial.println("INICIANDO RECEPTOR");

  // Inicializar M5StickC Plus2

  auto cfg = M5.config();

  M5.begin(cfg);

  delay(500);

  // Configurar zona horaria para convertir
  // las marcas de tiempo Unix recibidas.

  setenv("TZ", zonaHoraria, 1);

  tzset();

  // Ajustar RTC del M5StickC Plus2

  ajustarRTC();

  // Configurar Wi-Fi

  WiFi.mode(WIFI_STA);

  WiFi.setSleep(false);

  esp_err_t resultadoCanal = esp_wifi_set_channel(
    CANAL_WIFI,
    WIFI_SECOND_CHAN_NONE
  );

  if (resultadoCanal != ESP_OK) {

    Serial.print("Error al configurar canal: ");

    Serial.println(esp_err_to_name(resultadoCanal));

  }

  // Preparar pantalla

  M5.Lcd.fillScreen(BLACK);

  M5.Lcd.setTextColor(WHITE);

  M5.Lcd.setTextSize(2);

  M5.Lcd.setCursor(10, 10);

  M5.Lcd.println("RECEPTOR");

  M5.Lcd.setTextSize(1);

  M5.Lcd.setCursor(10, 40);

  M5.Lcd.println("Iniciando ESP-NOW...");

  // Mostrar MAC

  Serial.print("MAC receptor: ");

  Serial.println(WiFi.macAddress());

  // Inicializar ESP-NOW

  if (esp_now_init() != ESP_OK) {

    Serial.println("Error al iniciar ESP-NOW");

    M5.Lcd.setCursor(10, 60);

    M5.Lcd.println("Error ESP-NOW");

    return;

  }

  // Registrar callback de recepcion

  esp_now_register_recv_cb(alRecibirDatos);

  Serial.println("ESP-NOW iniciado correctamente");

  Serial.println("Receptor listo");

  M5.Lcd.setCursor(10, 60);

  M5.Lcd.println("Esperando emisor...");

}

// ==================================================
// 10. LOOP
// ==================================================

void loop() {

  M5.update();

  static uint32_t ultimaActualizacion = 0;

  if (millis() - ultimaActualizacion >= 200) {

    ultimaActualizacion = millis();

    DatosSensor copiaDatos;

    uint32_t tiempoUltimoPaquete;

    bool datosDisponibles;

    uint32_t recibidos;
    uint32_t perdidos;

    uint32_t recibidosUltimos;
    uint32_t perdidosUltimos;
    uint32_t cantidadUltimos;

    // Copiar datos de forma segura

    portENTER_CRITICAL(&muxDatos);

    memcpy(
      &copiaDatos,
      &datosRecibidos,
      sizeof(DatosSensor)
    );

    tiempoUltimoPaquete = ultimoPaquete;

    datosDisponibles = hayDatos;

    recibidos = paquetesRecibidos;

    perdidos = paquetesPerdidos;

    recibidosUltimos = recibidosVentana;

    perdidosUltimos = perdidosVentana;

    cantidadUltimos = cantidadVentana;

    portEXIT_CRITICAL(&muxDatos);

    // Calcular PLR acumulada

    uint32_t esperados = recibidos + perdidos;

    float plrAcumulada = 0.0;

    float plrMovil = 0.0;

    if (esperados > 0) {

      plrAcumulada =
        (float)perdidos / esperados * 100.0;

    }

    if (cantidadUltimos > 0) {

      plrMovil =
        (float)perdidosUltimos /
        cantidadUltimos * 100.0;

    }

    // Limpiar zona de datos

    M5.Lcd.fillRect(0, 35, 240, 100, BLACK);

    M5.Lcd.setTextSize(1);

    M5.Lcd.setCursor(5, 37);

    if (!datosDisponibles) {

      M5.Lcd.setTextColor(YELLOW);

      M5.Lcd.println("ESPERANDO EMISOR");

    } else {

      uint32_t tiempoSinDatos =
        millis() - tiempoUltimoPaquete;

      // Estado de comunicacion

      if (tiempoSinDatos > 5000) {

        M5.Lcd.setTextColor(YELLOW);

        M5.Lcd.println("SIN ACTUALIZACION");

      } else {

        M5.Lcd.setTextColor(GREEN);

        M5.Lcd.println("RECIBIENDO");

      }

      M5.Lcd.setTextColor(WHITE);

      // Temperatura

      M5.Lcd.printf(
        "Temp: %.1f C\n",
        copiaDatos.temperatura
      );

      // Humedad

      M5.Lcd.printf(
        "Humedad: %.1f %%\n",
        copiaDatos.humedad
      );

      // Numero de paquete

      M5.Lcd.printf(
        "Paquete: %lu\n",
        (unsigned long)copiaDatos.secuencia
      );

      // Hora real de envio

      if (copiaDatos.horaEnvio != 0) {

        time_t instante = copiaDatos.horaEnvio;

        struct tm horaEnvio;

        localtime_r(&instante, &horaEnvio);

        M5.Lcd.printf(
          "Envio: %02d:%02d:%02d\n",
          horaEnvio.tm_hour,
          horaEnvio.tm_min,
          horaEnvio.tm_sec
        );

      } else {

        M5.Lcd.println("Envio: sin hora");

      }

      // Paquetes recibidos

      M5.Lcd.printf(
        "Recibidos: %lu\n",
        (unsigned long)recibidos
      );

      // Paquetes perdidos

      M5.Lcd.printf(
        "Perdidos: %lu\n",
        (unsigned long)perdidos
      );

      // PLR acumulada

      M5.Lcd.printf(
        "PLR total: %.2f %%\n",
        plrAcumulada
      );

      // PLR ventana movil

      M5.Lcd.printf(
        "PLR 100: %.2f %%\n",
        plrMovil
      );

      // Mostrar informacion por Serial

      Serial.printf(
        "Temp: %.1f C | Humedad: %.1f %% | "
        "Secuencia: %lu | Envio: ",

        copiaDatos.temperatura,
        copiaDatos.humedad,
        (unsigned long)copiaDatos.secuencia
      );

      if (copiaDatos.horaEnvio != 0) {

        time_t instante = copiaDatos.horaEnvio;

        struct tm horaEnvio;

        localtime_r(&instante, &horaEnvio);

        Serial.printf(
          "%02d:%02d:%02d",
          horaEnvio.tm_hour,
          horaEnvio.tm_min,
          horaEnvio.tm_sec
        );

      } else {

        Serial.print("SIN HORA");

      }

      Serial.printf(
        " | Recibidos: %lu | Perdidos: %lu | "
        "PLR total: %.2f %% | PLR movil: %.2f %% | "
        "Hace %lu ms\n",

        (unsigned long)recibidos,
        (unsigned long)perdidos,
        plrAcumulada,
        plrMovil,
        (unsigned long)tiempoSinDatos
      );

    }

  }

  delay(10);

}
