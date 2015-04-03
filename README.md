# cities-skylines-detour
Proof of concept for a simple detour for functions in Cities: Skylines (or more generally: Unity 5 x64 on Windows.)

I experimented with three different ways to detour methods (i.e. redirect calls from one method to another) in Unity's Mono version (x64). The code has only been tested on Windows, but there seem to be some mods that use my code to detour methods (and I have not seen complaints of them not working on Linux/Mac).

The *RedirectionHelper* class contains the interesting functions. Three different approaches to detours are explored:

1. Primitive patching. In this approach, *GetFunctionPointer* is used to ensure that the target method (which is to be detoured) and the destination method are both compiled. Then a straight-forward jump is patched into the target function (on machine code level). Surprisingly, this works just fine. Mono does not seem to recompile or move methods (at least in Unity's version, which is based on Mono 2.6.4 I think). It has not been tested with methods created at runtime and generic methods. The main advantage of this approach is that it works even after the target method and its callers have been compiled. On the other hand it (obviously) fails whenever the compiler decides to inline the method.

2. Swapping out the pointer to the native code. This requires digging somewhat deeper into Mono. We first ensure that both our target and the destination are compiled. Then we get the *jit_code_hash* from the Mono domain and reimplement *mono_internal_hash_table_lookup* with the appropriate hash functions to get the *MonoJitInfo* struct for the target method. (We could also find the offset of that function, but I wanted to avoid that -- mainly because I want to reduce the dependency on the specific Mono version. Of course, offsets within structs have the same problem but I think it is unlikely that they are going to change.) Last, the pointer to the native code is replaced by the pointer to the destination. To me, this approach is much cleaner than the first one (and it generally works). The main caveat is that this only detours calls to the target that have not been compiled prior to the detouring. When Mono compiles a method, instead of a call to a function it will emit a jump to a *magic trampoline* that invokes the JIT compiler on the target of the call and then patches the callsite. If that has already happened, we'd have to patch all the callsites as well. So this way of implementing detours relies on doing it before anything else is calling the target method.

3. Swapping out the IL code. .NET languages are compiled into an intermediate language (IL) which is compiled to native code at runtime. Arguably the cleanest way to detour a method is to replace its IL at runtime. While it is conceptually very simple, it does come with some problems. First off, the IL code is assembly specific. A call to a method (a reference to a field, constant etc.) in IL uses a *token* as a target (it is just a number). Each assembly has various metadata tables which link tokens to the actual methods. All this data is static and is compiled into the assembly. So copying IL from one assembly (e.g. your code) to another assembly (e.g. the assembly with the method to be detoured) will not work in most cases: The tokens will not fit. A token for a debug output function may instead link to a method animating a plasma ball (use real world examples, they said...). So to properly detour by replacing IL, you'd have to edit this metadata as well. My code does *not* do this, mainly because I do not think it is worth doing. The code in the *RedirectionHelper* class allows you to detour methods which are in the same assembly as the destination method. For this to work, you have to perform the detour *before* the IL gets compiled into machine code. I found no simple way to *re*-compile a method in Mono, mainly because of the aforementioned callsite patching. (Maybe the patch-locations are actually stored somewhere even after the patching occurred. Again, I have not looked into that.) Another limitation is that only *normal* methods (no generics/runtime generated methods) can be patched with the current setup. The main reason for this is that in order to replace the IL code, the whole header of the method is replaced. The *MonoMethodHeader* struct's size varies from method to method (depending on the number of try/catch clauses), so just copying over one header over the other will not work. Instead, the pointer to the header is replaced. For normal methods, this pointer is simply added to the end of the *MonoMethod* struct. I have not looked into generics, maybe it is straight-forward for them as well.

If you just use the primitive patching, you will not need to *DllImports*.
Another word of note: Because this code is targeted at x64 Windows (where only fastcall is left), we can actually detour a member-method to a static method. *this* is passed in RCX, just as the first parameter of a fastcall.
