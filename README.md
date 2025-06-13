# Projeto Radio FM com RDA5807M
Montagem de radio usando módulo RDA5807M, RTC 1307 e encoder KY-040.

Para este projeto, pensando no uso de capacitdades de RDS, usei o RDA5807M, que possui a capacidade de recepção.
Algumas considerações: 
Nitidamente zero ruído captado e muito melhor captação de sinal que o TEA5767. Mesmo com o desafio, por estar destreinado, de adaptar o circuito para encaixe na breadboard, me deixou muito surpreso com a melhor performance. Nem os resistores de pull-up foram necessários.
Adaptei o circuito anterior para uso em 3.3V (Oled, RTC 1307 (adaptado, removendo os resistores R2 e R3) e rádio) mantendo somente o RE (KY-040) em 5V.
Dificuldades com o tempo de atualização do RDS, mas ajustado no sketch para olhar na troca de estações a cada 5s.
Devido a tamanho do display, optei por não exibir as mensagens de programação, informações da estação e horário (quando apresentado), embora os dados estivessem disponíveis.

## Lista de material utilizado:
- Arduino Nano
- display oled 128x64, SSD1306
- Tiny RTC 1307 com AT24C32 (eeprom), adaptado para 3.3V (removidos R2 e R3)
- RDA 5807M (módulo FM)
- Rotary encoder KY-040
- jumpers


## Esquema (fritzing)
![RADIO TEA5767_bb](https://github.com/user-attachments/assets/952aa1f5-5e07-4b08-ad13-c2511f6fe8fa)

## Adaptação do módulo RDA5807M na placa ilhada para uso em breadboard.
Próximo:
![20250610_115513](https://github.com/user-attachments/assets/e7841f6d-c34d-449c-aeb2-666c619ba278)

Com a visão de antena e conector P2 Fêmea
![20250610_115447](https://github.com/user-attachments/assets/d35e35a1-8e5c-4ea1-ba86-cef109d03777)

## Prototipação e Montagem na Breadboard
![20250613_170247](https://github.com/user-attachments/assets/e3b79b39-1365-4f68-9eab-7f50f9020096)

![20250613_170252](https://github.com/user-attachments/assets/02efd313-becb-4b3c-8be6-75d8906f1647)


![20250613_170705](https://github.com/user-attachments/assets/78b15551-3362-470a-bbcf-4dae3cb15f3f)

## Sketch
```
#include <RDA5807.h>
#include <Wire.h>
#include <GyverOLED.h>
#include "RTClib.h"
#include <RotaryEncoder.h>

// inicia estrutura de tempo (t)
RTC_DS1307 rtc;
const static char *DiasdaSemana[] = { "DOM", "SEG", "TER", "QUA", "QUI", "SEX", "SAB" };
DateTime now;
unsigned long tanterior_RTC;// declara variável de atualização
//encoder
RotaryEncoder encoder(A2, A3, RotaryEncoder::LatchMode::TWO03); //encoder acelerado
int nowPos, newPos, startPos = 0;
//display
GyverOLED<SSD1306_128x64, OLED_NO_BUFFER> oled;
//radio
double frequencia = 87.5;
RDA5807 radio;
// RDS
char *rdsMsg;
char *stationName;
char *rdsTime;
char *rdsProgram;
unsigned long t_anterior_RDS;
#define MAX_DELAY_STATUS 5000

//Públicas
char buf[20];

void setup() {
  Wire.begin();
  Serial.begin(9600);
  //Display
  oled.init();
  //RTC
  if (!rtc.begin()) {  
    Serial.println("DS1307 não encontrado");
    while (1);
  }
  if (!rtc.isrunning()) {               
    Serial.println("DS1307 rodando!");
    //rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));  //CAPTURA A DATA E HORA EM QUE O SKETCH É COMPILADO
  }
  now = rtc.now();
  //verifica ultima estação sintonizada (Rotary Encoder). Pega valor da NVRAM e atualiza frequencia
  Serial.print("li no setup: ");Serial.println(rtc.readnvram(0));
  startPos = rtc.readnvram(0);
  if (startPos>127 || startPos <0){
    startPos = 0;
  }
  nowPos = startPos+newPos;
  frequencia = 87.5 + ((double) nowPos * (0.2));
  
  // Inicializa o radio
  radio.setup();
  radio.setRDS(true); // Liga RDS
  radio.setVolume(6);
  radio.setFrequency((int)(frequencia*100));
  radio.setRDS(true);
  radio.setRdsFifo(true);
  radio.setLnaPortSel(3);
  radio.setAFC(true);
  refreshDisplay();
}

void loop() {
  //Atualiza RTC - 1s
  unsigned long atual = millis();
  if (atual - tanterior_RTC > 60000) {
    now = rtc.now();
    tanterior_RTC = atual;
    refreshDisplay();
  }
  //atualiza RDS
  unsigned long atual_RDS = millis();
  if (atual_RDS - t_anterior_RDS > MAX_DELAY_STATUS){
    if (radio.getRdsReady() &&  radio.hasRdsInfo()){
      showRDS();
    }
    if(stationName != NULL){
      Serial.println(stationName);
    }
    t_anterior_RDS = atual_RDS;
  }

  //static int pos = 0;
  encoder.tick();
  newPos = encoder.getPosition();
  if (nowPos != startPos+newPos) {
    nowPos = startPos+newPos;
    frequencia = 87.5 + ((double)nowPos * (0.2));
    Serial.print("Valor pos: ");Serial.println(newPos);
    Serial.print("Freq: ");Serial.print(frequencia);Serial.println(" MHz");
    radio.setFrequency((int)(frequencia*100)); 
    rtc.writenvram(0, (byte)nowPos);
    //Serial.print("gravei: ");Serial.println(nowPos);
    //Serial.print("li: ");Serial.println(rtc.readnvram(0)); 
    if (radio.getRdsReady() &&  radio.hasRdsInfo()){
      showRDS();
    }
    if(stationName != NULL){
      Serial.println(stationName);
      oled.print(String(stationName));
    }
    refreshDisplay();
  }
}

void refreshDisplay(void) {  //função que irá atualizar o display, na inicialização, loop e mudança de estação
  //iniciliza
  oled.clear();
  oled.home();
  //linha de tempo
  oled.setScale(1);
  oled.setCursorXY(0, 0);
  sprintf(buf, "%02d/%02d/%02d  %s   %02d:%02d", now.day(), now.month(), now.year() - 2000, DiasdaSemana[now.dayOfTheWeek()], now.hour(), now.minute());
  oled.print(buf);
  oled.fastLineH(10, 0, 128);
//  oled.fastLineH(56, 0, 128);
  //status
  oled.fastLineH(31, 102, 126);
  oled.fastLineV(126, 31, 16);
  byte PosxSinal = 102;
  for (int x = 0; x < radio.getRssi()/4 ; x++) {
      oled.fastLineV(PosxSinal, 31, 31-x);
      PosxSinal = PosxSinal+2;
  }
  oled.setCursorXY(112, 38);
  if (radio.isStereo()) {
    oled.print("ST");
  }else {
    oled.print("MO");
  }

  oled.setCursorXY(88, 50);
  if (radio.getVolume()) {
    sprintf(buf, "Vol: %d", radio.getVolume());
    oled.print(buf);
  }
    
  //Frequencia selecionada
  oled.setScale(3);
  oled.setCursorXY(10, 36);
  dtostrf(frequencia, 3, 1, buf);
  oled.print(buf);

  //Informações RDS
  //Estação
  oled.setCursorXY(16, 18);
  oled.setScale(1);
  oled.print(String(stationName));
  oled.update();
}

// Show current frequency
void showStatus()
{
  char aux[80];
  sprintf(aux, "\nYou are tuned on %u MHz | RSSI: %3.3u dbUv | Vol: %2.2u | Stereo: %s\n", radio.getFrequency(), radio.getRssi(), radio.getVolume(), (radio.isStereo()) ? "Yes" : "No");
  Serial.print(aux);
}

void showRDS()
{
    showStatus();
    Serial.println("--- RDS ---");

    stationName = radio.getRdsStationName();
    if (stationName != NULL)  Serial.println(stationName);
    rdsMsg = radio.getRdsStationInformation();
    if (rdsMsg != NULL)  Serial.println(rdsMsg);
    rdsTime = radio.getRdsTime();
    if (rdsTime != NULL)  Serial.println(rdsTime);
    rdsProgram = radio.getRdsProgramInformation();
    if (rdsProgram != NULL)  Serial.println(rdsProgram);
}

```
