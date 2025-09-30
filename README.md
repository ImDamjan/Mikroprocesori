## Projekat iz mikroprocesorskih sistema (STM32CubeIDE)

Ovaj projekat demonstrira čitanje analogne vrednosti sa joystick-a pomoću ADC-a na STM32 mikrokontroleru i slanje informacija o smeru preko UART-a (Putty na računaru). Takođe, projekat kontroliše jednu LED diodu u zavisnosti od stanja joystick-a.
---

## Uključena slika

![STM32 pinout](/mnt/data/17dec457-c82c-4511-95d9-e68919f974e8.png)

Slika prikazuje raspored pinova mikrokontrolera koji smo koristili. Na slici su pinovi koji se koriste za:

* ADC kanal za joystick - PA4
* LED pin - PA5
* UART - USART1

---

## Deklarisane promenljive
* `ADC_HandleTypeDef hadc1;`

  * Strukturа koju HAL koristi za upravljanje ADC1 periferijom. U kodu koristimo `hadc1` da startujemo konverzije i pročitamo vrednosti joystick-a.

* `UART_HandleTypeDef huart1;`

  * UART handler koji se koristi za slanje tekstualnih poruka prema računaru (Putty). `HAL_UART_Transmit(&huart1, ...)` šalje poruke.
    
* `char msg[64];`

  * Bafer u koji se pakuje poruka pre slanja preko UART-a.

* `uint8_t last_direction = 0;`

  * Varijabla koja pamti poslednji izveštajeni smer joystick-a.

---

## `int main(void)`

1. Poziva se `HAL_Init()` i `SystemClock_Config()` 
2. Pozivaju se `MX_GPIO_Init()`, `MX_ADC1_Init()`, `MX_SPI1_Init()`, `MX_USART1_UART_Init()`, `MX_USART2_UART_Init()` 
3. Inicijalizujemo lokalnu promenljivu `last_direction` na 0.
4. Ulaz u beskonačnu petlju (`while (1)`) 

---

## `while (1)`

1. **Konfigurisanje kanala ADC-a:**

   ```c
   ADC_ChannelConfTypeDef sConfig = {0};
   sConfig.Channel = ADC_CHANNEL_4; // PA4
   sConfig.Rank = ADC_REGULAR_RANK_1;
   sConfig.SamplingTime = ADC_SAMPLETIME_3CYCLES_5;
   HAL_ADC_ConfigChannel(&hadc1, &sConfig);
   ```

   * Na ovaj način određujemo koji kanal (pin) ADC-a će da očitava. U ovom slučaju to je kanal 4 (PA4)

2. **Startujemo ADC, čekamo konverziju i uzimamo vrednost:**

   ```c
   HAL_ADC_Start(&hadc1);
   HAL_ADC_PollForConversion(&hadc1, 10);
   uint32_t val = HAL_ADC_GetValue(&hadc1);
   HAL_ADC_Stop(&hadc1);
   ```

   * `val` sada sadrži digitalnu vrednost (0..4095 za 12-bitni ADC) koja zavisi od položaja joystick potenciometra.

   > Ispod se nalaze debug linije zbog odredjivanja smera
   >
   > ```c
   > snprintf(msg, sizeof(msg), "ADC=%lu\r\n", val);
   > HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), 100);
   > ```

3. **Mapiramo ADC vrednost na `direction` (smer):**

   ```c
   uint8_t direction = 0;
   if      (val < 300)     direction = 5; // CLICK
   else if (val < 900)     direction = 1; // LEFT
   else if (val < 1700)    direction = 3; // DOWN
   else if (val < 2600)    direction = 2; // UP
   else if (val < 3400)    direction = 4; // RIGHT
   else                    direction = 0; // CENTER
   ```

4. **Šaljemo poruku samo kada se `direction` promeni:**

   ```c
   if (direction != last_direction)
   {
       switch(direction) {
           case 1: snprintf(msg, sizeof(msg), "Joystick LEFT\r\n"); break;
           case 2: snprintf(msg, sizeof(msg), "Joystick UP\r\n"); break;
           case 3: snprintf(msg, sizeof(msg), "Joystick DOWN\r\n"); break;
           case 4: snprintf(msg, sizeof(msg), "Joystick RIGHT\r\n"); break;
           case 5: snprintf(msg, sizeof(msg), "Joystick CLICK\r\n"); HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET); break;
           default: snprintf(msg, sizeof(msg), "Joystick CENTER\r\n"); HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); break;
       }
       HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), 100);
       last_direction = direction;
   }
   ```

5. **Mali delay:**

   ```c
   HAL_Delay(100);
   ```

   * Koristimo 100 ms pauze između očitavanja — ovo smanjuje broj poruka i omogucava lakse citanje.

---

## Korak po korak

1. Kreiramo novi projekat u STM32CubeIDE i biramo plocu STM32C0316-DK (Discovery kit) (MCU STM32C031C6Tx).
2. Sa nasim modelom pinovi su automatski postavljeni pa je potrebno samo doraditi kod 
4. U **System Core**: omogućiomo takt za GPIO/ADC/USART.
5. Generisemo kod (Project -> Generate Code), otvoramo `main.c` i dodajemo svoju logiku u `USER CODE BEGIN`
6. Kompajliramo i flash-ujemo program na ploču (Run -> Debug ili Run -> Run).

---

## Putty / serijska komunikacija — kako si povezao i konfiguracija

1. Povezali smo razvojnu ploču sa računarom preko ST-LINK-a na port COM10
2. Putty podesavamo na sledeci nacin:

   * Connection type: **Serial**
   * Serial line: `COM10`
   * Speed: **115200** -- obavezno
   * Data bits: **8**
   * Parity: **None**
   * Stop bits: **1**
   * Flow control: **None**

---

## Kako UART radi

* UART pretvara podatke (bajtove) koje pošaljemo u `HAL_UART_Transmit` u sekvencu naponskih impulsa na TX liniji.
* Na računaru, USB-Serial adapter ili ST-LINK VCP konvertuje te napone u USB poruke koje Windows/OS vidi kao virtualni COM port (npr. COM10).
* Terminal program (Putty) čita te poruke i prikazuje ih kao tekst.
* U kodu svaki put kada se promeni smer pozivamo `HAL_UART_Transmit` sa bufferom `msg` i dužinom `strlen(msg)`.

---

