## Taste Control

- Creator: Emway
- Category: hardware
- Difficulty: Medium
- Solves: 18

# Challenge Description: 
The council has decided that all new dishes have to go through a taste control test. This all went well until our neighbouring restaurant -Chef de Villain- presented authentic Dutch food. 
Ever since, the council has installed a watchdog that will reject or accept a dish after a single bite.
The challenge can be accessed by selecting the Taste control option.


# Walkthrough  

We are given two things: 
- The esp32 ofc 
- An C program named `taste-control`

## The esp32

If we connect the esp32 with puTTY, we see this: 
![Main menu](/assets/images/TUDCTF2025/mainmenu.png)

We open option 8 (the description says we can access the chalange by selecting the 'Taste control' option) and we see this:
![Option 8](/assets/images/TUDCTF2025/option8.png)

As you can see we are greeted with 3 options. I chose option 1 first and when we do this we can choose which byte we would like to taste. The obvious answer would be to choose byte 0 and this is what we get:
![Choosing to taste byte 0](/assets/images/TUDCTF2025/option8-1-2.png)

We don't get a lot of information from this, so let's check out option 2.
Option 2 is asking us to prepare a byte, and like with the tasting option, the most logical option would be to prepare byte 0.
![Choosing to prepare byte 0](/assets/images/TUDCTF2025/option8-2-error.png)

It seems that the program crashed. When I first did the challenge I thought the crash was unintended, and tried multiple times to see if I still get the same error, and I did, and then I came to the realization that I maybe should look at the C code first, so let's do that.

## taste_control.c

I'll show the entire code first, and then explain/highlight the important parts

```
#include "taste_control.h"
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_task_wdt.h"
#include "mbedtls/aes.h"
#include "terminal.h"
#include "usb_protocol.h"
#include "flagstore.h"
#include "rawdata.h"

static unsigned char flag[32];
RAWDATA uint8_t print_buf[64];
static int idx = -1;

unsigned int crctable[256] = {
    0x0000, 0x1189, 0x2312, 0x329B, 0x4624, 0x57AD, 0x6536, 0x74BF,
    0x8C48, 0x9DC1, 0xAF5A, 0xBED3, 0xCA6C, 0xDBE5, 0xE97E, 0xF8F7,
    0x0919, 0x1890, 0x2A0B, 0x3B82, 0x4F3D, 0x5EB4, 0x6C2F, 0x7DA6,
    0x8551, 0x94D8, 0xA643, 0xB7CA, 0xC375, 0xD2FC, 0xE067, 0xF1EE,
    0x1232, 0x03BB, 0x3120, 0x20A9, 0x5416, 0x459F, 0x7704, 0x668D,
    0x9E7A, 0x8FF3, 0xBD68, 0xACE1, 0xD85E, 0xC9D7, 0xFB4C, 0xEAC5,
    0x1B2B, 0x0AA2, 0x3839, 0x29B0, 0x5D0F, 0x4C86, 0x7E1D, 0x6F94,
    0x9763, 0x86EA, 0xB471, 0xA5F8, 0xD147, 0xC0CE, 0xF255, 0xE3DC,
    0x2464, 0x35ED, 0x0776, 0x16FF, 0x6240, 0x73C9, 0x4152, 0x50DB,
    0xA82C, 0xB9A5, 0x8B3E, 0x9AB7, 0xEE08, 0xFF81, 0xCD1A, 0xDC93,
    0x2D7D, 0x3CF4, 0x0E6F, 0x1FE6, 0x6B59, 0x7AD0, 0x484B, 0x59C2,
    0xA135, 0xB0BC, 0x8227, 0x93AE, 0xE711, 0xF698, 0xC403, 0xD58A,
    0x3656, 0x27DF, 0x1544, 0x04CD, 0x7072, 0x61FB, 0x5360, 0x42E9,
    0xBA1E, 0xAB97, 0x990C, 0x8885, 0xFC3A, 0xEDB3, 0xDF28, 0xCEA1,
    0x3F4F, 0x2EC6, 0x1C5D, 0x0DD4, 0x796B, 0x68E2, 0x5A79, 0x4BF0,
    0xB307, 0xA28E, 0x9015, 0x819C, 0xF523, 0xE4AA, 0xD631, 0xC7B8,
    0x48C8, 0x5941, 0x6BDA, 0x7A53, 0x0EEC, 0x1F65, 0x2DFE, 0x3C77,
    0xC480, 0xD509, 0xE792, 0xF61B, 0x82A4, 0x932D, 0xA1B6, 0xB03F,
    0x41D1, 0x5058, 0x62C3, 0x734A, 0x07F5, 0x167C, 0x24E7, 0x356E,
    0xCD99, 0xDC10, 0xEE8B, 0xFF02, 0x8BBD, 0x9A34, 0xA8AF, 0xB926,
    0x5AFA, 0x4B73, 0x79E8, 0x6861, 0x1CDE, 0x0D57, 0x3FCC, 0x2E45,
    0xD6B2, 0xC73B, 0xF5A0, 0xE429, 0x9096, 0x811F, 0xB384, 0xA20D,
    0x53E3, 0x426A, 0x70F1, 0x6178, 0x15C7, 0x044E, 0x36D5, 0x275C,
    0xDFAB, 0xCE22, 0xFCB9, 0xED30, 0x998F, 0x8806, 0xBA9D, 0xAB14,
    0x6CAC, 0x7D25, 0x4FBE, 0x5E37, 0x2A88, 0x3B01, 0x099A, 0x1813,
    0xE0E4, 0xF16D, 0xC3F6, 0xD27F, 0xA6C0, 0xB749, 0x85D2, 0x945B,
    0x65B5, 0x743C, 0x46A7, 0x572E, 0x2391, 0x3218, 0x0083, 0x110A,
    0xE9FD, 0xF874, 0xCAEF, 0xDB66, 0xAFD9, 0xBE50, 0x8CCB, 0x9D42,
    0x7E9E, 0x6F17, 0x5D8C, 0x4C05, 0x38BA, 0x2933, 0x1BA8, 0x0A21,
    0xF2D6, 0xE35F, 0xD1C4, 0xC04D, 0xB4F2, 0xA57B, 0x97E0, 0x8669,
    0x7787, 0x660E, 0x5495, 0x451C, 0x31A3, 0x202A, 0x12B1, 0x0338,
    0xFBCF, 0xEA46, 0xD8DD, 0xC954, 0xBDEB, 0xAC62, 0x9EF9, 0x8F70
};

void taste_inspection_task(void *pvParameters) {
    esp_task_wdt_deinit();
    
    // Watchdog to eat the data if it leaks
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 2000,
        .idle_core_mask = 0,
        .trigger_panic = true
    };
    
    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));
    
    // setup AES
    mbedtls_aes_context aes;
    mbedtls_aes_init(&aes);
    mbedtls_aes_setkey_dec(&aes, key, 256);
    
    // Decode flag with crazy crc IV
    uint8_t tmp[32];
    mbedtls_aes_crypt_cbc(&aes, MBEDTLS_AES_DECRYPT, 32, iv, flag, tmp);
    mbedtls_aes_free(&aes);
    print_buf[idx] = tmp[idx];
    
    // Wait for AES to finish decoding
    vTaskDelay(pdMS_TO_TICKS(2000));
    print("%lu", print_buf[idx]);
    print_buf[idx] = (uint8_t) crctable[idx];
    idx = -1;
}

void inspect_index(int i) {
    idx = i;
    xTaskCreate(taste_inspection_task, "taste_ctrl", 4096, NULL, 5, NULL);
    while (idx != -1) {
        // execute quadruple DES with CRC
    }
}

// Start challenge
void run_watchdog_chal() {
    set_msg("TASTE CONTROL");
    
    // Start menu
    while (1) {
        terminal_clear_screen();
        printf("\nChoose one of the following options:\n    1) Taste a byte of the flag\n    2) Prepare a byte for tasting\n    3) Exit\n\n    > ");
        int choice = strtol(readline(), NULL, 10);
        printf("\n\n");
        if (choice == 1) {
            // Option 1
            printf("which byte would you like to taste (0-31)?\n\n    > ");
            int i = strtol(readline(), NULL, 10);
            printf("\n\n");
            if (i >= 0 && i < 32) {
                printf("Byte %d = %02x\n\n", i, cache[i]);
            } else {
                puts("Invalid choice - ");
            }
            puts("Press ENTER to return");
            readline();
        } else if (choice == 2) {
            // Option 2
            printf("which byte would you like to taste (0-31)?\n\n    > ");
            int i = strtol(readline(), NULL, 10);
            printf("\n\n");
            if (i >= 0 && i < 32) {
                printf("Starting inspection of byte %d", i);
                inspect_index(i);
            } else {
                puts("Invalid choice - ");
            }
            puts("Press ENTER to return");
            readline();
            // Option 3
        } else if (choice == 3) {
            break;
        } else {
            // Option None
            puts("Invalid choice - Press ENTER to return");
            readline();
        }
    }   
}
```



So first we see the `static unsigned char flag[32];`, which holds the flag ofc and is encrypted.
Secondly, we see the `RAWDATA uint8_t print_buf[64];`, and `RAWDATA` is used to place a variable in a no-init section of RAM, meaning the content of it survives a reboot when something crashes.
Then we have another variable and the look-up table but they don't have anything that is important.

Then we go the vulnerable part: `taste_inspection_task`
```
void taste_inspection_task(void *pvParameters) {
    esp_task_wdt_deinit();
    
    // Watchdog to eat the data if it leaks
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 2000,
        .idle_core_mask = 0,
        .trigger_panic = true
    };
    
    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));
    
    // setup AES
    mbedtls_aes_context aes;
    mbedtls_aes_init(&aes);
    mbedtls_aes_setkey_dec(&aes, key, 256);
    
    // Decode flag with crazy crc IV
    uint8_t tmp[32];
    mbedtls_aes_crypt_cbc(&aes, MBEDTLS_AES_DECRYPT, 32, iv, flag, tmp);
    mbedtls_aes_free(&aes);
    print_buf[idx] = tmp[idx];
    
    // Wait for AES to finish decoding
    vTaskDelay(pdMS_TO_TICKS(2000));
    print("%lu", print_buf[idx]);
    print_buf[idx] = (uint8_t) crctable[idx];
    idx = -1;
}
```

What this code does is basically this (I'm not that familiar with C but I hope this explains it enough and is correct):
1. The task sets a Watchdog Timer for 2000ms. A watchdog timer is used to monitor tasks, and makes sure that the task is able to execute in a given time period.
2. `.trigger_panic = true` basically means that if the time runs out, it crashes the system (what happens when we taste a byte)  
3. It decrypts the flag into a temporary buffer (`tmp`) and selects the byte you chose at index `idx`
4. It copies the decrypted byte into `print_buf` 
5. Then it sleeps for 2000ms 

What we have to notice here, is that the Watchdog Timer is 2000ms, and the sleep is also 2000ms, meaning that no matter what you do, the timer will always run out and the program will crash. 
Another very important thing we can notice, is that the decrypted byte was copied into `print_buf`, and like I explained earlier, `RAWDATA` is used to place a variable in a no-init section of RAM, meaning the content of it survives a reboot when something crashes.


Now to `inspect_index`:
```
void inspect_index(int i) {
    idx = i;
    xTaskCreate(taste_inspection_task, "taste_ctrl", 4096, NULL, 5, NULL);
    while (idx != -1) {
        // execute quadruple DES with CRC
    }
}
```
The while loop is an infinite loop, and makes it so that we have to wait for the Watchdog timer to kill it.

Last but not least, we have the main menu. The main menu shows the options we saw on puTTY.
In option 1 we see this: `printf("Byte %d = %02x\n\n", i, cache[i]);`. I assumed that the cache has the data that was written to `print_buf`. It is not written anywhere explicitly, but it is the most logical option because otherwise there is no way to solve the challenge afaik. 
Option 2 calls `inspect_index` and like I already explained, it writes the decrypted byte into `print_buf` and the system crashes.