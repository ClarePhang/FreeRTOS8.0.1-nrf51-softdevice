# FreeRTOS Flash demo port to nrf51822+softdevice

## Description

This is a port of the simple FreeRTOS blinky demo. Critical region
handling has been updated to use the softdevice api.

Due to a lack of SysTick hardware in the nrf51 the tick timer is
implemented via RTC1. RTC1 auses the LFCLK; so we don't need HFCLK
running most of the time.

The vApplicationIdleHook is implemented to call sd_app_evt_wait(). So
it ought to sleep as well as can be expected.

## Questionable Decisions + Todos

I've configure the FreeRTOS tick with the app_timer api included in
the NRF SDK. It's probably be more efficient to configure and use RTC1
directly and simply `#define xSysTickHandler RTC1_IRQHander`. I chose
not to do this because nearly all the nordic demos rely on app_timer
functionality, which would break if I stole RTC1 away from it.

CPSID/CPSIE Fun:

If CPSID is called in for an extended period of time, then the radio
will miss it's deadlines. For now, I've mostly been commenting out
these instructions, while investigating the implications. Eventually,
the correct way to handle these may be to store/restore a mask of enabled
application interrupts (excluding those blocked or restricted by the
sd) and sprinkle it around as needed. You would think that the
sd_nvic_critical_region_enter would do something like this, however in
SDK 6.0.0 there is only a stub, leaving us to wonder how this is
implemented in the softdevice blob.

port.c(ulSetInterruptMaskFromISR):
  Removed cpsid before the branch. There was no complimentary cspie in
  the vClearInterruptMaskFromISR. Since these only seem to be called
  from the TickHandler and in this port this is not run as an ISR, I
  don't think this should matter.

port.c(xPortPendSVHandler):
  I commented out the cspid/cspie bookending
  the branch to vTaskSwitchContext. This would seem like a "bad idea",
  but the logic in that function has nothing to do with the actual
  context switch. It's more of a hook for stack overflow testing and
  the trace framework. Could be bad news if for task stats and the
  like. No practical effect on my projects yet.


I've reduced the default call stack size to the 1596B required for softdevice operation (down from 0xC00). Task
stacks are allocated from the heap, but if you add more functionality
outside tasks (BLE event handlers, notably) then you'll need to increase
the stack in startup_nrf51.s (and decrease the configTOTAL_HEAP_SIZE
in FreeRTOSConfig.h). There's only ~2kB of RAM left after the s120
softdevice has it's way (it uses 10K+1.5K stack); so memory will be
very tight. Careful application design is needed.

Patches welcome, this code is PoC quality (at best).

## Prerequisites

- GCC ARM Toolchain (arm-eabi-none-gcc, et. al) in your path

- NRF SDK installed locally and the path specified as SDK_PATH in the
  Makefile

- NRF Softdevice loaded onto the development board.

- Tested with s120v1 and SDK 6.0.0

## Usage

1. Edit Makefile as needed. You'll also want to correct the LED_0
   definition in main.c to correspond to your development boards LED.

2. Flash softdevice

3. `make flash`

## Credits

- Build environment based heavily on
  [hlnd/nrf51-pure-gcc-setup](https://github.com/hlnd/nrf51-pure-gcc-setup)

- Heavily influenced by the existing FreeRTOS Flash Demo and the existing FreeRTOS ARM_CM0 code.
