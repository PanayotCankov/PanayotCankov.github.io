**native instance** - Objective-C class instance or Java instance based on the context.
**root instance** - An instance that is kept on the stack, in static field, accessible from the global object atc. and as such will be considered reachable and be alive after a garbage collection.

When GCs run they first block the threads to collect all root instances from the stack. Then resume the execution until they mark all reachable objects in a separate thread. Then block again the threads to complete the marking. And at the end start finalizing and deallocating unreachable instances. While the actual GC implementation may be much more sophisticated, all implementations in VMs used for UI aim at minimizing the time the main thread is blocked.

## Splice
**splice** - let's introduce a new term. Splice we will call a bond, made between a native (Java or Objective-C) class instance and its representation by a JavaScript instance.

Splice includes APIs to:
 - Get the native instance given the JavaScript instance
 - Get the JavaScript instance given the native instance
 - Synchronize the life-cycle of the two object
 - Throw when methods on the native or the JavaScript instance are called while the other is garbage collected or deallocated
 - Proxy method calls to and fro the JavaScript and native instances.

When is a Splice created?
 - Native instance is returned from a constructor, method, property, block, anonymous interface, lambda etc. called from JavaScript
 - Native instance is passed as an argument to a constructor, method, property, block, anonymous interface, lambda etc. implemented in JavaScript

### iOS
The Objective-C runtime has no GC but runs on ref-counting instead. The retain and release of each Objective-C instance can be intercepted and code can be injected on retain/release. The Objective-C has association API that allows native objects to be assigned dynamically key-value pairs. The JavaScriptCore has API to retain/release JavaScript instances that can be used to root/unroot them (allow or deny them from being garbage collected). The Splice will refer to linking an Objective-C instance of a class with a JavaScript instance.

#### Splice Live-Cycle
When the splice is created, it increases the ref-count of the Objective-C instance by 1, and if the ref-count is >1 the splice roots the JavaScript instance. From that point on:
 - If the Objective-C instance ref-count is incremented from 1 to 2, the JavaScript instance is rooted.
 - If the Objective-C instance ref-count is decremented from 2 to 1, the JavaScript instance is unrooted.
 - Only while the Objective-C instance ref-count = 1, the JavaScript instance is unrooted, and the JavaScript is eligible for garbage collection. When the GC fail to find access to the JavaScript instance from live JavaScript objects, it will mark it for collection. Then when the JavaScript instance is finalized, it schedule decrementation of the Objective-C instance ref-count, deallocating the native instance.

#### Properties of the Implementation
You can make a reference cycle in Objective-C that will leak native and JavaScript instances, because the JavaScript GC does not traverse the native objects and fields to find cycles. Native tools can point to leaked instances.

You can let instances be prematurely collected. Using weak properties or APIs that involve methods such as `setTarget:selector:...` that will add the Objective-C instance as native target, but holds weak Objective-C references, these does not increment the ref-count of the Objective-C instance. The target may remain with ref-count 1. If the JavaScript GC collects the JavaScript instances of the Splice, it will then delete the Objective-C instance. The annoying part is the code will behave properly for a while - until the non deterministic completion of a GC. Enabling zombie objects in Xcode can be used to trace prematurely collected instances.

Overall the implementation is really Objective-C friendly and predictable. When working with native APIs it requires some additional care about memory management. Is very UI friendly and does not introduce big pauses on the main UI thread.

## Android


{% include disqus.html %}