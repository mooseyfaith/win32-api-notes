# win32-api-notes
Win32 API code collection for many problems in game development

## Optional XInput Support (without crashing if its missing)
XInput is the most easy way to use XBox 360-Controllers in your application.
Normaly you would compile your program including XInput.h and linking against Xinput.lib.
This way your program will load the appropriate xinput.dll at runtime, so you can check for connected controllers and poll their input.

But there is a problem. If your user has no xinput.dll your program will crash.
The following code will help you to dynamically check for xinput and use it if its available, otherwise the program will pretend, that no controllers are available, but will still be able to run regardless.
The idea is to only include XInput.h and not link against XInput.lib, but instead to try to dynamically load the xinput.dll at runtime. In case XInput is not available at all, it is useful to have function stubs, that the code can call, without any effect. The function stubs will just behave as if there are no controllers present.
```c++
#include <Windows.h>
#include <Xinput.h>

// defining the signatures for the xinput functions
// these macros can be used to create a named function with the specified input and output.
// wich makes it easy to change the signature of all functions, wich where created using these macros.
// in this specific case it is probably overkill, but I just find this to be a nice way to declare function signatures in general.

#define X_INPUT_GET_STATE_DEC(name) DWORD name(DWORD dwUserIndex, XINPUT_STATE *pState)
typedef X_INPUT_GET_STATE_DEC((*XInput_GetState_Function));

#define X_INPUT_GET_CAPABILITIES_DEC(name) DWORD name(DWORD dwUserIndex, DWORD dwFlags, XINPUT_CAPABILITIES *pCapabilities)
typedef X_INPUT_GET_CAPABILITIES_DEC((*XInput_GetCapabilities_Funtion));

// global function pointers
// since XInput.h defines XInputGetState and XInputGetCapabilities as functions
// we have to use a different name for our global function pointers.
// you could also add these to a struct, instead of having global variables
XInput_GetState_Function       MyXInputGetState;
XInput_GetCapabilities_Funtion MyXInputGetCapabilities;

// define xinput function stubs, in case xinput is not available

// same as:
// DWORD XInputGetStateStub(DWORD dwUserIndex, XINPUT_STATE *pState)
X_INPUT_GET_STATE_DEC(XInputGetStateStub)
{
    return ERROR_DEVICE_NOT_CONNECTED;
}

// same as:
// DWORD XInputGetCapabilitiesStub(DWORD dwUserIndex, DWORD dwFlags, XINPUT_CAPABILITIES *pCapabilities)
X_INPUT_GET_CAPABILITIES_DEC(XInputGetCapabilitiesStub)
{
    return ERROR_DEVICE_NOT_CONNECTED;
}

// check for different versions of xinput
void init_xinput()
{
  HMODULE xinput_dll = LoadLibraryA("xinput1_4.dll");

  if (!xinput_dll)
      xinput_dll = LoadLibraryA("xinput1_3.dll");

  if (!xinput_dll)
      xinput_dll = LoadLibraryA("xinput9_1_0.dll");
      
  // TODO: add more cases if there are newer xinput dlls
  
  if (xinput_dll)
  {
      MyXInputGetState        = (XInput_GetState_Function)GetProcAddress(      xinput_dll, "XInputGetState");
      MyXInputGetCapabilities = (XInput_GetCapabilities_Funtion)GetProcAddress(xinput_dll, "XInputGetCapabilities");
  }
  
  // if we could not initialize XInputGetState, use the stub funtions
  // they're null by default, since they're global variables
  if (!MyXInputGetState)
  {
      MyXInputGetState = XInputGetStateStub;
      MyXInputGetCapabilities = XInputGetCapabilitiesStub;
  }
}
```

## Measuring elapsed Time in Seconds
There are two functions in Win32 API to measure elapsed time.
These are mainly used for animations or advancing the game state, since they represent the time in the real world, therfore the perceived time of the user.

QueryPerformanceFrequency is used to get the number of 'ticks' wich represents one second. Once your program runs, this number is allways the same, so you don't need to query it multiple times.

QueryPerformanceCounter is used to get the time represented in 'ticks'.

'ticks' are an abstract counter wich is the internal unit Windows uses to measure time. Typically hardware clocks are used to determin elapsed time.
Since there could be different hardware wich runs at different known frequencies, the counter might grow slower or faster depending on the frequency. Thats why whe have to dynamically ask for what one second is in 'ticks'.

In order to measure elapsed seconds of a piece of code, you need to store the time in ticks before and after the code is executed.
Then you subtract the beginning time from the end time and devide be the ticks-per-second retreived by QueryPerformanceFrequency.
The following are exaples of how you can use this.

```c++
#include <stdint.h> // I personally use my own type names like u64 instead of uint64_t :D

void init_ticks_per_second(uint64_t *ticks_per_second)
{
    LARGE_INTEGER win32_ticks_per_second;
    QueryPerformanceFrequency(&win32_ticks_per_second);
    *ticks_per_second = win32_ticks_per_second.QuadPart;
}

void foo()
{
    uint64_t ticks_per_second;
    
    init_ticks_per_second(&ticks_per_second);
    
    LARGE_INTEGER last_time_ticks;
    QueryPerformanceCounter(&last_time_ticks);
    
    while (true)
    {
        // do something ...
        
        LARGE_INTEGER time_ticks;
        QueryPerformanceCounter(&time_ticks);
        
        // using float, since we are interested in frations of a second
        float delta_seconds = (time_ticks.QuadPart - last_time_ticks.QuadPart) / (float)ticks_per_second;
        
        // update last_time_ticks,
        // otherwise you measure the time from the beginning of the loop, to the current iteration.
        // this way you measure the time between 2 loop iterations
        last_time_ticks = time_ticks;
    }
}

float get_elapsed_seconds(uint64_t *timer_ticks, uint64_t ticks_per_second)
{
    LARGE_INTEGER time_ticks;
    QueryPerformanceCounter(&time_ticks);

    // using float, since we are interested in frations of a second
    float delta_seconds = (time_ticks.QuadPart - *timer_ticks) / (float)ticks_per_second;

    // update last_time_ticks,
    // otherwise you measure the time from the beginning of the loop, to the current iteration.
    // this way you measure the time between 2 loop iterations
    *timer_ticks = time_ticks.QuadPart;
    
    return delta_seconds;
}

void foo2()
{
    uint64_t ticks_per_second;
    
    init_ticks_per_second(&ticks_per_second);
       
    uint64_t timer_ticks;
    // can be used to init timer, just ignore the return value
    get_elapsed_seconds(&timer_ticks, ticks_per_second);
    
    while (true)
    {
        // do something ...
        
       float delta_seconds = get_elapsed_seconds(&timer_ticks, ticks_per_second);
    }
}

uint64_t get_elapsed_micro_seconds(uint64_t *timer_ticks, uint64_t ticks_per_second)
{
    LARGE_INTEGER time_ticks;
    QueryPerformanceCounter(&time_ticks);

    // multiply difference by 1000 (1 second = 1000 milliseconds),
    // before deviding by ticks_per_second
    uint64_t delta_micro_seconds = (time_ticks.QuadPart - *timer_ticks) * 1000 / ticks_per_second;

    // update last_time_ticks,
    // otherwise you measure the time from the beginning of the loop, to the current iteration.
    // this way you measure the time between 2 loop iterations
    *timer_ticks = time_ticks.QuadPart;
    
    return delta_micro_seconds;
}
```
