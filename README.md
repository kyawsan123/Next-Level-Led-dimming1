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
volatile unsigned int timer_minutes = 0; // Max 1440 mins = 24:00
volatile unsigned int seconds_counter = 0;
unsigned char timer_state = 0;           // 0: Display OFF, 1: Display ON at 00:00, 2: Active Countdown
unsigned char dim_level = 0;             // 0:Low, 1:Mid, 2:High
unsigned char white_status = 0;          // 0:Off, 1:On

// Button State Flags
unsigned char btn_dim_pressed = 0;
unsigned char btn_timer_pressed = 0;
unsigned char btn_white_pressed = 0;

// Common Cathode 7-Segment Codes (0-9)
unsigned char segment_code[] = {0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F};

void setup(void);
void update_system_outputs(void);
void display_process(void);

// Timer1 Interrupt (100ms interval)
void __interrupt() isr(void) {
    if (TMR1IF) {
        TMR1H = 0x0B; TMR1L = 0xDC; // Reload for 100ms
        
        if (timer_state == 2 && timer_minutes > 0) {
            seconds_counter++;
            if (seconds_counter >= 600) { // 60 seconds = 1 minute
                timer_minutes--;
                seconds_counter = 0;
                if (timer_minutes == 0) {
                    timer_state = 1; // Time is up, reset to 00:00 state
                }
            }
        }
        TMR1IF = 0;
    }
}

void main(void) {
    setup();
    
    while(1) {
        // 1. Timer Button Logic (RA1)
        if (BTN_TIMER == 0) {
            if (btn_timer_pressed == 0) {
                if (timer_state == 0) {
                    timer_state = 1; // First click: Display 00:00, Lamp OFF
                    timer_minutes = 0;
                } 
                else if (timer_state == 1) {
                    timer_state = 2; // Second click: Start 15 mins countdown, Lamp Low
                    timer_minutes = 15;
                    dim_level = 0; 
                    seconds_counter = 0;
                    update_system_outputs();
                } 
                else {
                    timer_minutes += 15; // Subsequent clicks: +15 mins up to 24:00
                    seconds_counter = 0;
                    if (timer_minutes > 1440) {
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

        // 2. Dimming Button Logic (RA0)
        if (BTN_DIM == 0) {
            if (btn_dim_pressed == 0) {
                if (timer_state == 2) {
                    dim_level++;
                    if (dim_level > 2) dim_level = 0;
                    update_system_outputs();
                }
                btn_dim_pressed = 1;
            }
        } else {
            btn_dim_pressed = 0;
        }

        // 3. White LEDs Button Logic (RA6)
        if (BTN_WHITE == 0) {
            if (btn_white_pressed == 0) {
                white_status = !white_status;
                if (white_status) {
                    WHITE_LEDS = 1; // White LEDs ON
                } else {
                    WHITE_LEDS = 0; // White LEDs OFF
                }
                btn_white_pressed = 1;
            }
        } else {
            btn_white_pressed = 0;
        }

        // Turn off blue lamp and indicators if not in active countdown state
        if (timer_state != 2) {
            CCPR1L = 0;      
            PORTC &= 0x07;   
        }

        display_process(); 
    }
}

void setup(void) {
    ANSEL = 0; ANSELH = 0; 
    
    TRISA = 0x43; // RA0, RA1, RA6 = Inputs, RA2, RA4, RA5 = Outputs
    TRISB = 0x00; // PORTB Outputs (7-Segment Data)
    TRISC = 0x00; // PORTC Outputs 
    
    PORTB = 0x00;
    PORTA |= 0x30; 
    PORTC |= 0x03; 
    WHITE_LEDS = 0; 
    
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
    
    PORTC &= 0x07; 
    
    if (dim_level == 0) { 
        CCPR1L = 50;        // Low Level
        PORTC |= 0x08;      // RC3 = Low Red Indicator ON
    } else if (dim_level == 1) { 
        CCPR1L = 150;       // Medium Level
        PORTC |= 0x10;      // RC4 = Mid Red Indicator ON
    } else { 
        CCPR1L = 255;       // High Level
        PORTC |= 0x20;      // RC5 = High Red Indicator ON
    }
}

void display_process(void) {
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
        PORTA |= 0x30; 
        PORTC |= 0x03; 
        PORTB = 0x00; 

        PORTB = segment_code[digits[i]]; 
        
        if (i == 1) {
            PORTB |= 0x80; // Colon (:) on RB7
        }
        
        if(i == 0)      RC0 = 0; 
        else if(i == 1) RC1 = 0; 
        else if(i == 2) RA4 = 0; 
        else            RA5 = 0; 
        
        __delay_ms(1); 
    }
    
    PORTA |= 0x30; 
    PORTC |= 0x03; 
}
