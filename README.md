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
