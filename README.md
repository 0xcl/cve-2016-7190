# Overview 
CVE-2016-7190 [0] is a heap overflow in the Array.map() function of ChakraCore that allows to overwrite adjacent memory. The main idea to gain arbitrary read-write access is to first allocate a number of consecutive JavaScript integer Arrays. and to exploit the overflow to manipulate the size of one array. Then, we leverage this array to change the base address of a Uint8Array to whatever address we want to read/write. The reason for this extra step is that the overflow is limited to 2x the size of the overflown integer array.

This vulnerability is very suitable for testing different exploitation strategies because it can be easily re-introduced into current versions of ChakraCore (see [undo-cve-2016-7190.patch](undo-cve-2016-7190.patch)).

# My testing environment
- ChakraCore Release 1.8 (6e56489a4fe940ce6f8a960caf9429700bf2d8db)
-  Microsoft Windows 10 Pro x64 / 10.0.14393 Build 14393
-  Binaries: https://drive.google.com/open?id=1WTXqMLVRHPRoBU9eo-MWEwAY--CQC8e3
-  Symbols: https://drive.google.com/open?id=1kQDrlG97NY5ruVH6i2Mjr43cFuNISp5z


# Example exploits
## Hijacking the control flow 
[In this example](cve-2016-7190-rip-hijack.js), we hijack the control flow by overwriting the vtable pointer of a Uint8Array C++ object to a controlled memory address, and invoking a function of the Uint8Array objects that results in the invocation of a virtual function.


## Data-only attack to generate arbitrary instructions (mitigated by ACG [3])
Control-flow Integrity (CFI) is a defense technique to mitigate control-flow hijacking attacks. The general idea of CFI is to compute a control-flow graph (CFG) of an application during compile time, and then instruments the application with run-time checks to ensure that control flow does not deviate from the statically computed CFG during run time. However, if an application, like a web-browser, supports dynamic code generation, the CFG must be extendable during run time. This imposes a number of challenges:

- how to distinguish a benign and malicious extension of the CFG?
- how to protect dynamically generated code?
- how to ensure that the correct program is generated?

During the time we conducted our research, we focused on the last challenge. The main idea is to manipulate the input (data) of the just-in-time (JIT) compiler. As a result, the JIT compiler would generate malicious code which would by then integrated into the current context--and even be hardened with CFI. Concurrent to our research, theori [4] showed how to bypass CFI by manipulating the output of the JIT compiler. This attack is mitigated by verifying the integrity of the output trough a checksum. This cannot stop our attack because we manipulate the input of the JIT compiler. HOWEVER, by outsourcing the JIT compilation to another process (aka Arbitrary Code Guard [3]) both attacks are mitigated.

[We demonstrate that](ir-corruption/cve-2016-7190-ir-corruption.js) an attacker can exploit a data-only attack against the JIT compiler to generate [arbitrary native code](ir-corruption/cve-2016-7190-ir-corruption.PNG). In particular, we modify the intermediate representation (IR) of the JIT compiler to inject attacker-controlled instructions. When the JIT compiler then generates the native code based on the modified IR it will generate attacker controlled native code. 

Between the time the research was originally conducted and now, the IR of ChakraCore was changed. We did not make the effort of porting the attack, hence, to play around with this attack, please use the following binaries.
* Binaries: https://drive.google.com/open?id=1Il51XlTPwYcUWcuMqchZIrkBC7Uq844v
* Symbols: https://drive.google.com/open?id=1w_TqCEoDFhi2SG48UrBBiVcl3rF_5-yG


## Data-only attack to re-map arbitrary memory as writable
During the analysis of ACG [3] we noticed, amongst others [1,2], that global read-only data, which is stored in the .mrdata section, must be adjusted to integrate the dynamically generated code. This is done through the LdrProtectMrdata() function which uses a lock to make it thread-safe. However, the .mrdata section is first re-mapped as writable before the lock is acquired. Interestingly, the .mrdata section contains the base address and size of the .mrdata section which is later used to re-map this section again as read only. 

An attacker can exploit this to create an attack primitive that allows to re-map read-only memory as writable. Therefore, the attacker executes the following steps:
1. acquire the lock for the .mrdata section
2. trigger the JIT compiler which executes in a separate thread
3. wait until the .mrdata section is mapped as writable
4. change the base address of the .mrdata section
5. release the lock 
As a result, the .mrdata section remained writable. To map arbitrary memory as writeable, the attacker just needs to change the base address of the .mrdata section, and then repeat the steps from above.

The consequence of this attack is that previously trusted data, i.e., data that is mapped as read only becomes untrusted. [In our proof-of-concept](cve-2016-7190-mrdata-unlock.js), we exploit this primitive to bypass Control-flow Guard (CFGuard) by changing the pointer to the CFGuard's verification function which is saved in `_guard_dispatch_icall_fptr`.  The effect can be observed by attaching the debugger and changing the `bypass_cfguard` variable.


# References
[0] https://bugs.chromium.org/p/project-zero/issues/detail?id=923  
[1] http://alex-ionescu.com/publications/euskalhack/euskalhack2017-cfg.pdf  
[2] https://sites.google.com/site/bingsunsec/dataonlyattack  
[3] https://blogs.windows.com/msedgedev/2017/02/23/mitigating-arbitrary-native-code-execution/  
[4] http://theori.io/research/chakra-jit-cfg-bypass  
