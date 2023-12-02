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
Ahora en el caso de que ha leido la distancia muestra la humedad y temperatura. Se muestra hasta que pasen 5 minutos o hasta que se pulse el boton para el reinicio o cambiar a modo admin.
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
Después de mostrarse la temperatura y humedad se pasa a enseñar los productos y sus precios. Aquí puedes subir hacia arriba o abajo y elegir el producto pulsando el joystick. También puede pasar que se pulse el botón para reinicio o modo admin y se saldría entonces de servicio.
```c
int servicio(){
  int lectura = analogRead(VRY);
  
  if(lectura<350){
    
    producto_elegido = (producto_elegido + 1);
    
    lcd.clear();
    lcd.setCursor(0,0);
  }
  else if(lectura>700){
    
    producto_elegido = (producto_elegido - 1);
    lcd.clear();
    lcd.setCursor(0,0);
  }
  lcd.setCursor(0,0);
  lcd.print(productos[abs(producto_elegido)%5]);
  lcd.setCursor(0,1);
  lcd.print(precios[abs(producto_elegido)%5]);  
  delay(100);
  
  lectura = digitalRead(R3);
  //si se elige producto
  if(lectura == LOW){
    return abs(producto_elegido)%5;
  }
  delay(100); 
  return -1;
}
```
Al elegirse el producto, se pasa a preparando cafe, otra funcion que se puede interrumpir pulsando el botón para cambiar a admin o repetir. En esta función se ha usado un led el cual se encuentra en los pones PWM y se incrementa la intensidad según el número aleatorio entre 4 y 8s. Además, como es posible que se quede dentro de este for durante 8s, lo cual haría que salte el watchdog, se le incluye un reset antes y después del for para que esto no ocurra.
```c
void preparando_cafe() {
  // Generar tiempo aleatorio entre 4 y 8 segundos
  int tiempoPreparacion = random(tiempoMin, tiempoMax); 
  // Calcular el incremento de tiempo para el bucle
  int incrementoTiempo = tiempoPreparacion / 255; 
  int i = 0;
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("Preparando");
  lcd.setCursor(0,1);
  lcd.print("Cafe ...");

  wdt_reset(); // reset aqui ya que si el numero random es 8 s se resetearia
  for (i = 0; i <= 255; i++) {
    analogWrite(PIN_LED_PREPARAR_CAFE, i); // Establecer la intensidad del LED2
    delay(incrementoTiempo);
    if (reinicio == true) {
      analogWrite(PIN_LED_PREPARAR_CAFE, 0);
      return -1;
    }
  }
  wdt_reset(); // reset aqui ya que si el numero random es 8 s se resetearia
```
Por último, en servicio, se resetean todas las variables globales para que se pueda repetir el proceso nuevamente y, además, se imprime un mensaje para retirar la bebida.
```c
    if (servicio() != -1) {
      preparando_cafe();
      if (reinicio == false) {
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("RETIRE BEBIDA");
        analogWrite(PIN_LED_PREPARAR_CAFE, 0);
        distancia_1m = false;
        temperatura_humedad = false;
        delay(RETIRAR_BEBIDA);
      }
```
Para activar el reinicio o entrar a admin, he empleado una interrupción la cual se activa a la que cambie el dato que recibe del boton. La he usado con CHANGE para que realice las dos funciones en una sola función.
```c
  pinMode(BOTTON, INPUT);
  attachInterrupt(digitalPinToInterrupt(BOTTON),interrupcion_callback,CHANGE);
```
```c
void interrupcion_callback() {
  int estado = digitalRead(BOTTON);
  //el boton ha sido pulsado
  if (estado == HIGH) {
    t1_boton = millis();
  } else {
    //el boton se ha dejado de pulsar
    if(millis() - t1_boton > BOTTON_ADMIN) {
      admin_mode = !admin_mode;
      reinicio = true;
    }
    if (millis() - t1_boton >= MIN_BOTTON_REINICIO && 
    millis() - t1_boton <= MAX_BOTTON_REINICIO) {
      reinicio = true;
    }
    else {
      reinicio = false;
    }
  }
}
```

