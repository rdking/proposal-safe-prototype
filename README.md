# proposal-safe-prototype
An ES proposal to fix the foot-gun of non-function objects in prototypes.
---

## Motivation
ES is a prototype-based language, but as of late, many new features have been proposed, some of which attempt to obviate the use of the prototype for its intended purpose as the template for an instantiation by `new`. The sole reason is because of the lack of a copy-on-write semantic for prototype properties with a non-function object as the value.

## Solution
This proposal is fairly straight forward, but may not be easy to implement. It only has 2 parts:

1. Implement a `"use safe-proto"` directive in the same fashion as `"use strict"`.
2. Implement a copy-on-write semantic for any object assigned to `__proto__` of any other object that is enabled by the afore-mentioned new directive.

## Behavior
It's that 2nd step that's so problematic. The behavior would be as follows:
* Any object attached to the `prototype` of a function is flagged as a prototype object.
* Prior to any write attempt on any portion of an object property of `__proto__`, all [[Get]]s return an instance-specific shallow copy of the object. Any nested [[Get]] also returns an instance-specific shallow copy.
* Nested prototype objects are returned uncopied and break the nested copying process.
* A write attempt on a non-prototype object property of `__proto__` causes the unique shallow copies to be committed as an own property to the object owning the `__proto__`.
* Object values received by accessors are ignored as they represent a calculated result.

Presently, the desired behavior can be approximated with a Membrane around the object attached to `__proto__`. The only differences in behavior are that:
1. This proposal will not use proxies, and
2. The membrane cannot flag prototype objects to facilitate the 3rd rule above.

Below is the Proxy version of the desired behavior.

```js
import from "proposal-inherited-keys";

function makeSafeProto(inst) {
  const ROOT = Symbol("ROOT");
  const PARENT = Symbol("PARENT");
  const PROPERTY = Symbol("PROPERTY");
  const knownProtos = [
    Object.prototype,
    Function.prototype,
    Array.prototype,
    Map.prototype,
    WeakMap.prototype,
    Set.prototype,
    WeakSet.prototype
  ];
  
  function isObject(val) {
    return (val && (typeof(val) == "object"));
  }
  
  let handler = {
    copies: new WeakMap(),
    get(target, prop, receiver) {
      let def = Object.getKnownPropertyDefinition(target, prop);
      let retval = Reflect.get(target, prop, receiver);
      if (def && ("value" in def) && isObject(retval)) {
        let oldProto = Object.getPrototypeOf(retval);
        let proto = new Proxy(oldProto, handler);
        retval = Object.assign({}, retval);
        Object.setPrototypeOf(retval, proto);
        retval = new Proxy(retval, handler);
      }
      
      return retval;
    },
    set(target, prop, value, receiver) {
    }
    defineProperty(target, prop, def) {
    },
    deleteProperty(target, prop) {
    },
    preventExtensions(target) {
    },
    setPrototypeOf(target, prototype) {
    }
  };
  
  //Shallow copy the instance...
  let retval = Object.assign({}, inst);
  //Make the prototype a proxy object
  let proto = Object.getPrototypeOf(retval);
  if (
  Object.setPrototypeOf(retval, new Proxy(proto, handler));
  handler.set(inst
  return retval;
}
```
