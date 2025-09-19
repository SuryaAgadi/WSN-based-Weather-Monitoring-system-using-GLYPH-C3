#define LDR 2 
#define IR 3
#define LED 9
void setup()
{
  pinMode(LDR, INPUT);
  pinMode(IR, INPUT);
  pinMode(LED, OUTPUT);
  Serial.begin(9600);
}
void loop()
{
  int LDR
  
  LDR_sensor=digitalRead(LDR);
  Serial.print("LDR sensor data= ");
  Serial.println(LDR_sensor);

  int IR_sensor=digitalRead("IR sensor data= ");
  delay(1000);
  if(LDR_sensor==0)
  {
    Serial.println("DAY");
    analogWrite(LED, 0);
  }
  if(LDR_sensor==1)
  {
    Serial.println("NIGHT");
    analogWrite(LED, 100); 

    if(IR_sensor==0 && LDR_sensor==1)
    {
      analogWrite(LED, 255);
      Serial.println("*NIGHT* the object is detected");
    }
  }
}
