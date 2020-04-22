/*
MatrixFrame
18x18 NeoPixel Matrix (324 px total)
Sparkfun Pro Micro (5v, 16MHz)
Spectrum Analyzer
*/
#include <Adafruit_GFX.h>
#include <Adafruit_NeoMatrix.h>
#include <Adafruit_NeoPixel.h>
#ifndef PSTR
 #define PSTR // Make Arduino Due happy
#endif

//Declare Spectrum Shield pin connections
#define STROBE 4
#define RESET 5
#define DC_One A0
#define DC_Two A1 

//Define spectrum variables
int freq_amp;
int Frequencies_One[7];
int Frequencies_Two[7]; 

// MATRIX DECLARATION:
#define NEO_MATRIX_WIDTH 18
#define NEO_MATRIX_HEIGHT 18

#define NEOPIXEL_PIN 6 // Shield maps it to pin 6

Adafruit_NeoMatrix matrix = Adafruit_NeoMatrix(NEO_MATRIX_WIDTH, NEO_MATRIX_HEIGHT, NEOPIXEL_PIN,
  NEO_MATRIX_BOTTOM     + NEO_MATRIX_LEFT +
  NEO_MATRIX_COLUMNS + NEO_MATRIX_ZIGZAG,
  NEO_GRB            + NEO_KHZ800);


enum RANGE {
  BASS = 0,
  MID_RANGE = 1,
  TREBLE = 2,
  ALL = 3
};

enum SCHEME {
  MAGNITUDE_HUE = 0,
  MAGNITUDE_HUE_2 = 1,
  HSV_COLOR_WHEEL = 2
};

/********************Setup *************************/
void setup() {
  Serial.begin(9600);
  Serial.println("ProtoStax Audio Visualizer Demo");
  Serial.println("**************************************************");

  matrix.begin();
  matrix.setTextWrap(false);
  matrix.setBrightness(40);
  matrix.fillScreen(0);
  matrix.show();
  
  //Set spectrum Shield pin configurations
  pinMode(STROBE, OUTPUT);
  pinMode(RESET, OUTPUT);
  pinMode(DC_One, INPUT);
  pinMode(DC_Two, INPUT);  
  digitalWrite(STROBE, HIGH);
  digitalWrite(RESET, HIGH);
  
  //Initialize Spectrum Analyzers
  digitalWrite(STROBE, LOW);
  delay(1);
  digitalWrite(RESET, HIGH);
  delay(1);
  digitalWrite(STROBE, HIGH);
  delay(1);
  digitalWrite(STROBE, LOW);
  delay(1);
  digitalWrite(RESET, LOW);
}


/************************** Loop*****************************/
void loop() {
  static int scheme = 0;
  while (Serial.available() > 0) {
    scheme = Serial.parseInt();
  }
  
  Read_Frequencies();
  Graph_Frequencies(ALL, scheme);
  // Print_Frequencies();
  delay(50);
 
}

int max_bass_freq = 0;
int max_mid_freq = 0;
int max_treble_freq = 0;

/*******************Pull frquencies from Spectrum Shield********************/
void Read_Frequencies(){
  max_bass_freq = 0;
  max_mid_freq = 0;
  max_treble_freq = 0;

  //Read frequencies for each band
  for (freq_amp = 0; freq_amp<7; freq_amp++)
  {
    Frequencies_One[freq_amp] = (analogRead(DC_One) + analogRead(DC_One) ) >> 1 ;
    Frequencies_Two[freq_amp] = (analogRead(DC_Two) + analogRead(DC_Two) ) >> 1; 

    if (freq_amp >= 0 && freq_amp < 2) {
        if (Frequencies_One[freq_amp] > max_bass_freq) 
          max_bass_freq = Frequencies_One[freq_amp];
        if (Frequencies_Two[freq_amp] > max_bass_freq) 
          max_bass_freq = Frequencies_Two[freq_amp];  
    }
    else if (freq_amp >= 2 && freq_amp < 5) {
        if (Frequencies_One[freq_amp] > max_mid_freq) 
          max_mid_freq = Frequencies_One[freq_amp];
        if (Frequencies_Two[freq_amp] > max_mid_freq) 
          max_mid_freq = Frequencies_Two[freq_amp];  
    }
    else if (freq_amp >= 5 && freq_amp < 7) {
        if (Frequencies_One[freq_amp] > max_treble_freq) 
          max_treble_freq = Frequencies_One[freq_amp];
        if (Frequencies_Two[freq_amp] > max_treble_freq) 
          max_treble_freq = Frequencies_Two[freq_amp];  
    }    

    digitalWrite(STROBE, HIGH);
    digitalWrite(STROBE, LOW);
  }
}

//int FREQ_DIV_FACTOR = 204; // 1023/5=204
//int FREQ_DIV_FACTOR = 127; //1023/8=127
int FREQ_DIV_FACTOR = 56;   //1023/18=56

/*******************Light LEDs based on frequencies*****************************/
void Graph_Frequencies(RANGE r, SCHEME s){
   int from = 0;
   int to = 0;

   switch(r) {
    case BASS:
      from = 0; //0
      to = 6;   //2
      break;
    case MID_RANGE:
      from = 6; //2
      to = 10;   //5
      break;
    case TREBLE:
      from = 10; //5
      to = 17;   //7
      break;
    case ALL:
      from = 0;
      to = 17; //7
      break;
    default:
      break;
   }
  
   // Serial.print("max freq is "); Serial.println(max_freq);
   // FREQ_DIV_FACTOR = max_bass_freq/4;
   // Serial.print("FREQ_DIV_FACTOR is "); Serial.println(FREQ_DIV_FACTOR); 
   
   static uint16_t hue = 0; //21845 22250 to -250 
   uint16_t hueDelta = 200;
   hue += hueDelta;

    
   uint16_t bassHue = 22250; 
   uint16_t midHue = 22250; //54613 
   uint16_t trebleHue = 22250; //43690

      
   matrix.fillScreen(0);
   uint32_t rgbcolor;
   for(int row= from; row<to; row++)
   {

     int freq = (Frequencies_Two[row] > Frequencies_One[row])?Frequencies_Two[row]:Frequencies_One[row]; 

     
     int numCol = (freq/FREQ_DIV_FACTOR);
     if (numCol > 18) numCol = 18;    //if (numCol > 8) numCol = 18;
    
     for (int col = 0 ; col < numCol ; col++) {
       switch(s) {
        case MAGNITUDE_HUE:
          bassHue = 22250; 
          midHue = 22250; //54613 
          trebleHue = 22250; //43690
          if (row >= 0 && row < 6) {  //0, 2
            rgbcolor = matrix.ColorHSV(bassHue - (7416 * col) );      
          } else if (row >= 6 && row < 10) { //2, 5
            rgbcolor = matrix.ColorHSV(midHue - (7416 * col) );      
            
          } else if (row >= 10 && row < 17) { //5, 7
            rgbcolor = matrix.ColorHSV(trebleHue - (7416 * col) );      
          } 
          break;          
        case MAGNITUDE_HUE_2:
          bassHue = 54613; 
          midHue = 54613; //54613 
          trebleHue = 54613; //43690        
          if (row >= 0 && row < 6) {  //0, 2
            rgbcolor = matrix.ColorHSV(bassHue - (7416 * col) );      
          } else if (row >= 6 && row < 10) { //2, 5
            rgbcolor = matrix.ColorHSV(midHue - (7416 * col) );      
            
          } else if (row >= 10 && row < 17) {   //5, 7
            rgbcolor = matrix.ColorHSV(trebleHue - (7416 * col) );      
          }        
          break;
        case HSV_COLOR_WHEEL:
          rgbcolor = matrix.ColorHSV(hue);
          break;
       }

      
        matrix.setPassThruColor(rgbcolor);      
        matrix.drawPixel(col, row, (uint16_t)0); // color does not matter here 
        matrix.setPassThruColor();
        
        //matrix.show();
     
     }
     matrix.show();
   }
}

// Used for debugging 
void Print_Frequencies() {
  for (int i = 0; i < 7; i++) {
    Serial.print("FreqOne["); Serial.print(i); Serial.print( "]:"); Serial.println((Frequencies_One[i]/FREQ_DIV_FACTOR)%5); 
    Serial.print("FreqTwo["); Serial.print(i); Serial.print( "]:"); Serial.println((Frequencies_Two[i]/FREQ_DIV_FACTOR)%5); 

  }
}
