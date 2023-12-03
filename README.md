# Maquina-Expendedora

# Indice
* [Indice][ind]
* [Introducción][int]
* [Material][material]
* [Esquema del circuito][esq]
* [Implementación][imp]
  * [Interrupción][int]
  * [Threads y controlador][tc]
  * [Watchdog][wd]
* [Resultado][res]

[ind]: https://github.com/acruzr2021/Maquina-Expendedora/blob/main/README.md#indice
[int]: https://github.com/acruzr2021/Maquina-Expendedora/blob/main/README.md#introducci%C3%B3n
[material]: https://github.com/acruzr2021/Maquina-Expendedora/blob/main/README.md#material
[esq]: https://github.com/acruzr2021/Maquina-Expendedora/blob/main/README.md#esquema-del-circuito
[imp]: https://github.com/acruzr2021/Maquina-Expendedora#implementaci%C3%B3n
[int]: https://github.com/acruzr2021/Maquina-Expendedora#interrupci%C3%B3n
[tc]: https://github.com/acruzr2021/Maquina-Expendedora#threads-y-controlador
[wd]: https://github.com/acruzr2021/Maquina-Expendedora#watchdog
[res]: https://github.com/acruzr2021/Maquina-Expendedora#resultado



# Introducción

Buenas, soy Alba Cruz, estudiante del grado de Ingeniería en Robótica Software de la URJC y este es mi blog de la práctica 3 de la asignatura Sistemas Empotrados y de Tiempo Real, la cual trata de implementar una máquina espendedora usando el kit de Arduino proporcionado.

# Material

Para la implementación de esta práctica se ha requerido del siguiente material:

| Componente  | Cantidad |
| ------------- | ------------- |
| Arduino Uno  | 1PC |
| DTH11 | 1PC  |
| Sensor de Ultrasonido | 1PC |
| Pantalla LCD | 1PC |
| Joystick | 1PC |
| Led | 2PC |
| Botón | 1PC |
| Resistencia 2K | 3PC |


## Esquema del circuito:

Un pequeño esquema de como ensamblar los componentes para este proyecto.

![empotrados](https://github.com/acruzr2021/Maquina-Expendedora/assets/92941137/30b4c13d-0b6b-48eb-a249-2debfc443dde)

# Implementación

## Interrupción

Para la implementación del botón, incorporamos al código una interrupción hardware que se activará cada vez que el estado del botón cambie. Pretendemos medir cuanto tiempo pasa el botón pulsado. Si el botón pasa de LOW a HIGH, entonces tomará el tiempo de inicio, y cuando cambie de HIGH a LOW, tomará el tiempo final. Luego, comprobará la diferencia de tiempos con el caso de reiniciar (de 2 a 3 segundos pulsado) o con el caso de acceder al panel de administrador (> 5 segundos pulsado).

```cpp
#define BOTON_PIN 2
#define MILIS_TO_SEC 1000

void boton_interrupt(){ 
  if (digitalRead(BOTON_PIN) == LOW){
    start_int = millis();

  } if (digitalRead(BOTON_PIN) == HIGH){
    end = millis();
    Serial.println((end - start_int));

    if (((end - start_int) >= 2*MILIS_TO_SEC) & ((end - start_int) <= 3*MILIS_TO_SEC)){
      check_admin = false;
      restart();

    } else if ((end - start_int) > 5*MILIS_TO_SEC){
      check_admin = !check_admin;
      restart();
    }
  }
}
```

En la función setup:

```cpp
void setup() {

  // código omitido

  pinMode(BOTON_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(BOTON_PIN), boton_interrupt, CHANGE);

  // código omitido

}
```

## Threads y controlador

Para la implementación de los sensores se han usado 3 threads diferentes, de los cuales, dos de ellos se ejecutan mediante un controlador.
Tenemos un thread para el sensor de ultrasonido, que ejecutará independientemente; uno para la medición de la temperatura y otro para el de la humedad, los cuales ejecutarán mediante el uso de un controlador.

```cpp
#define TRIGGER_PIN 8
#define ECHO_PIN 13
#define BOTON 6
#define CM_MS 0.01715

DHT_Unified dht(DHTPIN, DHTTYPE);

void callback_humid() {
  sensors_event_t event;
  dht.humidity().getEvent(&event);
  if (isnan(event.relative_humidity)) {
    Serial.println(F("Error reading humidity!"));
  }
  else {
    hum = event.relative_humidity;
  }
}

void callback_temp() {
  sensors_event_t event;
  dht.temperature().getEvent(&event);
  if (isnan(event.temperature)) {
    Serial.println(F("Error reading temperature!"));
  } else{
    temp = event.temperature;
  }
}

long readUltrasonicDistance() {
  pinMode(TRIGGER_PIN, OUTPUT);
  digitalWrite(TRIGGER_PIN, LOW);
  digitalWrite(TRIGGER_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIGGER_PIN, LOW);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BOTON, INPUT_PULLUP);

  // lee el pin echo y devuelve la onda de sonido en microsegundos
  cm = CM_MS * pulseIn(ECHO_PIN, HIGH);
  return cm;
}
```

En el setup:
```cpp
void setup() {

  //código omitido

  // inicializa sensor DTH11
  dht.begin();

  //thread del sensor de ultrasonido
  distance_thread.enabled = true;
  distance_thread.setInterval(300);  
  distance_thread.onRun(readUltrasonicDistance);

  //threads and controller temperature and humidity
  temp_thread.enabled = true;
  temp_thread.setInterval(250);
  temp_thread.onRun(callback_temp);

  humid_thread.enabled = true;
  humid_thread.setInterval(350);
  humid_thread.onRun(callback_humid);

  controller.add(&temp_thread);
  controller.add(&humid_thread);

  // código omitido

}
```

Esto nos permite planificar y manejar fácilmente tareas periódicas, definiendo tiempos fijos en cada ejecución.


## Watchdog

Como medida de seguridad, incluimos en nuestra implementación un watchdog que reiniciará la placa si en una iteración tarda más de 8 segundos en llegar a la linea de reset.

```cpp
void setup() {

  // código omitido

  // inicia watchdog
  wdt_disable();
  wdt_enable(WDTO_8S);

  // código omitido

}

void loop(){

  if(!check_admin){
    servicio();
  } else{
    admin();
  }

  wdt_reset();
  delay(100);
}
```

# Resultado

Para finalizar, dejo un video con el resultado final de la implementación:

[![Video](https://github.com/acruzr2021/Maquina-Expendedora/blob/main/Screenshot%20from%202023-12-03%2020-34-27.png)](https://youtu.be/gJ3RzG8vNtc?si=yvfdwHYo1AgBy72y)




