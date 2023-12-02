# Práctica 3: Controlador Máquina Expendedora
---------------------------------------------------------
### Sistema empotrados y de tiempo real

La práctica se dividía en varias partes las cuales eran: Arranque, Servicio y Admin.

La primera parte de Arranque se realizó únicamente en el setup. Lo que hace es realizar la función arranque tres veces, una por minuto, la cual, lo que realiza es escribir "CARGANDO..." por pantalla y encender un led y apagarlo 
```c
  for (int i = 0; i < 3; i += 1) {
    arranque();
  }
```
```c
void arranque(){
  analogWrite(PIN_LED, 255);
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("CARGANDO...");
  delay(500);

  analogWrite(PIN_LED, 0);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("");
  delay(500);
}
```

La parte de Servicio se divide en varios trozos.
En primer lugar, una funcion la cual detecta si hay alguien a menos de un metro de distancia y si no lo hay pone "ESPERANDO CLIENTE"
```c
if(lector_distancia()){

(...)

} else {
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("ESPERANDO");
  lcd.setCursor(0,1);
  lcd.print("CLIENTE");
}
```
Siendo lector_distancia() una funcion que lee y cambia una variable llamada  par aque en ncada vuelta del bucle no vuelva a buscar a alguien a menos de 1 metro.
```c
bool lector_distancia() {
  if (distancia_1m == false) {
    long t; //timepo que demora en llegar el eco
    long d; //distancia en centimetros
    digitalWrite(TRIGGER, HIGH);
    delayMicroseconds(10);          //Enviamos un pulso de 10us
    digitalWrite(TRIGGER, LOW);
    
    t = pulseIn(ECHO, HIGH); //obtenemos el ancho del pulso
    d = t/59;             //escalamos el tiempo a una distancia en cm
    
    delay(100); 
    
    if (d < distancia) { //d < 1m
      distancia_1m = true;
    }
  }
  return distancia_1m;
```
Ahora en el caso de que ha leido la distancia muestra la humedad y temperatura. Se muestra hasta que pasen 5 minutos o hasta que se pulse el boton para el reinicio.
```c
void lector_temperatura_humedad(){

  // Leemos la humedad relativa
  float h = dht.readHumidity();
  // Leemos la temperatura en grados centígrados (por defecto)
  float t = dht.readTemperature();
  if (isnan(h) || isnan(t)){
    Serial.println("Error obteniendo los datos del sensor DHT11");
    return;
  }

  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("Hum: ");
  lcd.setCursor(5,0);
  lcd.print(h);

  lcd.setCursor(0,1);
  lcd.print("Temp:");
  lcd.setCursor(6,1);
  lcd.print(t);

  for (int i = 0; i < (TIEMPO_T_H/500); i += 1) {
    // si se activa el reinicio
    if (reinicio == true) {
      return -1;
    }
    delay(500);
  }
}
```
Después de mostrarse la temperatura y humedad se pasa a enseñar los productos y sus precios. Aquí puedes subir hacia arriba o abajo y elegir el producto pulsando el joystick
