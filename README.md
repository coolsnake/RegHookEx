# RegHookEx
External mid-function hooking method to retrieve register data


Here's some sample usage.  I'm testing this on SWBF 2017, so if you have that, feel free to follow along.
Using CE or whatever your scanner/debugger of choice is, I located a writeable viewangle (pitch/yaw that control the local soldier)

![Alt text](https://s18.postimg.org/qbwtabdx5/image.png "Function where viewangle is written")
Looking at this function, and setting a breakpoint, RSI + 0x68 and RSI + 6c are the yaw and pitch values.
![Alt text](https://s18.postimg.org/d7r8xy6jd/image.png "Another view with Reclass.NET")

In order to get the value of RSI, we need to do a `mov` to an address.  We can't do this in the original function, else it'll corrupt the return address, as well as its length.  We'll need to detour the program flow out of that function, to another function we create, do the `mov`, then return program flow back where we left off.

In x86, you can do
```
jmp 0x5dc20000
```

In x64, we need to be more creative.  Credit goes to stevemk14ebr and his [polyhook](https://github.com/stevemk14ebr/PolyHook), I'm using a detouring method similar to his.
```
push rax
mov rax, 0x5dc20000
xchg qword ptr ss:[rsp], rax
ret
```
![Alt text](https://s18.postimg.org/5gaiz5pgp/image.png )

The above is the same function, but the particular spot I'm choosing to hook is a clean spot to do it.

Writing the hook requires a minimum of 16 bytes, and since no instructions are larger than 15 bytes, you'll need to count the nearest end of instructions after the 16th byte from the address you're hooking.  I've highlighted the 23 bytes to be overwritten with the hook.

##Syntax of RegHookEx:
Here's some sample usage, for this given midfunction hook.
```
RegHookEx AngleFuncHook(rpm.hProcess, AngleFunc, 23, RegHookEx::Regs::RSI);
DWORD64 AngleFuncPtr = AngleFuncHook.GetAddressOfHook();
```
Specificly, 
```RegHookEx(
  _In_  HANDLE  HandleToProcess,
  _In_  DWORD64  FunctionAddress,
  _In_  SIZE_T LengthOfInstructions,
  _In_  BYTE  RegisterToRead
);
```
The above sets up the hook, but it isn't actually created until you do the following:
```DWORD64 regHookEx.GetAddressOfHook();
```
From there, you'd dereference at `AngleFuncPtr`, then add 0x68 for the pitch/yaw, as discovered earlier.

RegHookEx also unhooks.  You can either unhook one hook in particular, with:
```AngleFuncHook.DestroyHook();```
Or you can unhook all hooks at once, with the static void.
```RegHookEx::DestroyAllHooks();```

###How does it work?
