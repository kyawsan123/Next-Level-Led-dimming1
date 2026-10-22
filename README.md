
#include <xc.h>

// Configuration Bits
#pragma config FOSC = INTRC_NOCLKOUT 
#pragma config WDTE = OFF            
#pragma config PWRTE = OFF           
#pragma config MCLRE = ON            
#pragma config CP = OFF              
#pragma config CPD = OFF             
#pragma config BOREN = OFF           
#pragma config IESO = OFF            
#pragma config FCMEN = OFF           
#pragma config LVP = OFF             

#define _XTAL_FREQ 4000000 // 4MHz

// Push Buttons & Outputs Mapping
#define BTN_DIM      RA0  // 1. Blue LED Dimming Level Button (Input)
#define BTN_TIMER    RA1  // 2. Timer Button & State Control (Input)
#define BTN_WHITE    RA6  // 3. White LEDs On/Off Button (Input)
#define WHITE_LEDS   RA2  // White LEDs Output Pin (RA2)

// Global Variables
volatile unsigned int timer_minutes = 0; // မိနစ်အရေအတွက် (Max 1440 mins = 24:00)
volatile unsigned int seconds_counter = 0;
unsigned char timer_state = 0;           // 0: Display OFF, 1: Display ON at 00:00 (Lamp OFF), 2: Active Countdown (Lamp ON)
unsigned char dim_level = 0;             // 0:Low, 1:Mid, 2:High
unsigned char white_status = 0;          // 0:Off, 1:On

// Button State Flags (Flicker-Free နှိပ်ရန်)
unsigned char btn_dim_pressed = 0;
unsigned char btn_timer_pressed = 0;
unsigned char btn_white_pressed = 0;

// Common Cathode 7-Segment Codes (0-9)
unsigned char segment_code[] = {0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F};

void setup(void);
void update_system_outputs(void);
void display_process(void);

// Timer1 Interrupt (100ms interval) - တိကျသော Countdown အတွက်
void __interrupt() isr(void) {
    if (TMR1IF) {
        TMR1H = 0x0B; TMR1L = 0xDC; // Reload for 100ms
        
        / State 2 (Active Countdown) ဖြစ်နေမှသာ အချိန်လျှော့ချမည်
        if (timer_state == 2 && timer_minutes > 0) {
            seconds_counter++;
            if (seconds_counter >= 600) { // 60 seconds = 1 minute
                timer_minutes--;
                seconds_counter = 0;
                if (timer_minutes == 0) {
                    timer_state = 1; // အချိန်ကုန်သွားပါက မီးပိတ်ပြီး 00:00 အနေအထားသို့ ရောက်မည်
                }
            }
        }
        TMR1IF = 0;
    }
}

void main(void) {
    setup();
    
    while(1) {
        // -------------------------------------------------------------
        // ၁။ Timer Button Logic (RA1) - အဆင့်လိုက် လုပ်ဆောင်ချက်
        // -------------------------------------------------------------
        if (BTN_TIMER == 0) {
            if (btn_timer_pressed == 0) {
                if (timer_state == 0) {
                    // ပထမတစ်ချက်နှိပ်ခြင်း: Display မှာ 00:00 ပြမည်၊ မီးအပြာ မလင်းသေးပါ
                    timer_state = 1;
                    timer_minutes = 0;
                } 
                else if (timer_state == 1) {
                    // ဒုတိယတစ်ချက်နှိပ်ခြင်း: ၁၅ မိနစ် စတင်ပြသမည်၊ မီးအပြာ Low ဖြင့် စတင်လင်းမည်
                    timer_state = 2;
                    timer_minutes = 15;
                    dim_level = 0; // Low ဖြင့် စတင်ရန်
                    seconds_counter = 0;
                    update_system_outputs();
                } 
                else {
                    // ထပ်မံနှိပ်လျှင် ၁၅ မိနစ်စီ တိုးမည် (24:00 အထိ)
                    timer_minutes += 15;
                    seconds_counter = 0;
                    if (timer_minutes > 1440) { // ၂၄ နာရီ (1440 မိနစ်) ကျော်လျှင် ပြန်ပိတ်မည်
                        timer_minutes = 0;
                        timer_state = 1;
                    }
                    update_system_outputs();
                }
                btn_timer_pressed = 1;
            }
        } else {
            btn_timer_pressed = 0;
        }

        // -------------------------------------------------------------
        // ၂။ Dimming Button Logic (RA0) - မီးအပြာ Low/Mid/High ချိန်ရန်
        // -------------------------------------------------------------
        if (BTN_DIM == 0) {
            if (btn_dim_pressed == 0) {
                if (timer_state == 2) { // Timer စတင်အလုပ်လုပ်နေမှသာ ချိန်နိုင်မည်
                    dim_level++;
                    if (dim_level > 2) dim_level = 0; // Low -> Mid -> High -> Low
                    update_system_outputs();
                }
                btn_dim_pressed = 1;
            }
        } else {
            btn_dim_pressed = 0;
        }

        // -------------------------------------------------------------
        // ၃။ White LEDs Button Logic (RA6) - မီးအဖြူ ၂ လုံး ဖွင့်/ပိတ် (RA2 ဖြင့် ထွက်မည်)
        // -------------------------------------------------------------
        if (BTN_WHITE == 0) {
            if (btn_white_pressed == 0) {
                white_status = !white_status; // On <-> Off Toggle
                if (white_status) {
                    WHITE_LEDS = 1; // RA2 HIGH (White LEDs ON)
                } else {
                    WHITE_LEDS = 0; // RA2 LOW (White LEDs OFF)
                }
                btn_white_pressed = 1;
            }
        } else {
            btn_white_pressed = 0;
        }

        // -------------------------------------------------------------
        // မီးအပြာ Lamp PWM နှင့် Red Indicators ထိန်းချုပ်မှု
        // -------------------------------------------------------------
        if (timer_state != 2) {
            CCPR1L = 0;      // Blue LED PWM OFF
            PORTC &= 0x07;   // Red Indicator LEDs များကို ပိတ်ရန်
        }

        // Display ကို အဆက်မပြတ် Refresh လုပ်နေခြင်း
        display_process(); 
    }
}

void setup(void) {
    ANSEL = 0; ANSELH = 0; // All Digital I/O
    
    // TRISA Configuration: RA0, RA1, RA6 = Inputs (1), RA2, RA4, RA5 = Outputs (0)
    TRISA = 0x43; // Binary: 0100 0011 (RA0, RA1, RA6 are inputs)
    TRISB = 0x00; // PORTB Outputs (7-Segment Data lines A-G + DP)
    TRISC = 0x00; // PORTC Outputs (RC0-RC1 Digits, RC2 PWM, RC3-RC5 Indicators)
    
    PORTB = 0x00;
    PORTA |= 0x30; // RA4, RA5 = 1 (Digits Off)
    PORTC |= 0x03; // RC0, RC1 = 1 (Digits Off)
    WHITE_LEDS = 0; // White LEDs Off initially
    
    // PWM Config (CCP1 - RC2 Pin)
    T2CON = 0x04;  
    PR2 = 255;     
    CCP1CON = 0x0C;
    
    // Timer1 Config (100ms interval)
    T1CON = 0x11;  
    TMR1H = 0x0B;  
    TMR1L = 0xDC;
    TMR1IE = 1; GIE = 1; PEIE = 1;
}

void update_system_outputs(void) {
    if (timer_state != 2) return;
    
    PORTC &= 0x07; // Red Indicator မီးများရှင်းလင်းခြင်း
    
    if (dim_level == 0) { 
        CCPR1L = 50;        // Low Level (Blue LED Lamp မှိန်မှိန်လင်းမည်)
        PORTC |= 0x08;      // RC3 = Low Red Indicator ON
    } else if (dim_level == 1) { 
        CCPR1L = 150;       // Medium Level
        PORTC |= 0x10;      // RC4 = Mid Red Indicator ON
    } else { 
        CCPR1L = 255;       // High Level (အလင်းဆုံး)
        PORTC |= 0x20;      // RC5 = High Red Indicator ON
    }
}

void display_process(void) {
    // State 0 (စစဖွင့်ချင်း) တွင် Display လုံးဝမပြပါ (ပိတ်ထားမည်)
    if (timer_state == 0) {
        PORTB = 0x00;
        PORTA |= 0x30; 
        PORTC |= 0x03; 
        return;
    }

    int hrs = timer_minutes / 60;
    int mins = timer_minutes % 60;
    int digits[4] = {hrs/10, hrs%10, mins/10, mins%10};

    for(int i=0; i<4; i++) {
        // Digit တွေ မောင်းတဲ့အခါ RA4 နဲ့ RA5 ကိုပဲ သုံးမှာမို့ RA2 (White LEDs) ကို မထိခိုက်စေရန်
        // PORTA |= 0x30 လို့ ရေးထားတာက RA4, RA5 ကိုပဲ 1 ပေးတာမို့ RA2 အလုပ်လုပ်တာကို မနှောင့်ယှက်ပါဘူး
        PORTA |= 0x30; 
        PORTC |= 0x03; 
        PORTB = 0x00; 

        // 7-Segment တွင် ဂဏန်းများ ထည့်သွင်းခြင်း
        PORTB = segment_code[digits[i]]; 
        
        // အလယ်က Colon ( : ) စက် ကို Digit 2 (i=1) ချိန်တွင် ဖွင့်ခြင်း
        if (i == 1) {
            PORTB |= 0x80; // RB7 ဖွင့်ရန်
        }
        
        // 5463AS-1 Display Pins Drive Configuration
        if(i == 0)      RC0 = 0; // Digit 1
        else if(i == 1) RC1 = 0; // Digit 2
        else if(i == 2) RA4 = 0; // Digit 3
        else            RA5 = 0; // Digit 4
        
        __delay_ms(1); 
    }
    
    PORTA |= 0x30; 
    PORTC |= 0x03; 
}
