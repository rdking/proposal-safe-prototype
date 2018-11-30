# proposal-safe-prototype
An ES proposal to fix the foot-gun of non-function objects in prototypes.
---

## Motivation
ES is a prototype-based language, but as of late, many new features have been proposed, some of which attempt to obviate the use of the prototype for its intended purpose as the base object for an instantiation by `new`. The sole reason is because of the lack of a copy-on-write semantic for prototype properties with a non-function object as the value.

## Solution
This proposal is fairly straight forward, but may not be easy to implement. The idea is to implement a copy-on-write semantic for any object assigned to `__proto__` of another object. In the interest of preserving as many as possible of the use-cases putting the foot-gun behavior to good use, there are a few conditions:
* A new well-known Symbol must be defined: `Symbol.SafeProto = Symbol("SafeProto")` to mark objects that should exhibit the new behavior.
* A new well-known Symbol must be defined: `Symbol.UnsafeProto = Symbol("UnsafeProto")` to mark objects that **must not** exhibit the new behavior.
* All native prototypes must have a `Symbol.UnsafeProto` property.
* Only objects containing `Symbol.SafeProto` as a key (without regard to value) will receive the new behavior.
* The new behavior is not affected by `"use strict"`.

## Behavior
The behavior would be as follows:
* Any object attached to the `prototype` of a function is flagged as a prototype object.
* Prior to any write attempt on any portion of an object property of `__proto__`, all [[Get]]s return an instance-specific shallow copy of the object. Any nested [[Get]] also returns an instance-specific shallow copy.
* Nested prototype objects are returned uncopied and break the nested copying process.
* A write attempt on a non-prototype object property of `__proto__` causes the unique shallow copies to be committed as an own property to the object owning the `__proto__`.
* Object values received by accessors are ignored as they represent a calculated result.

Presently, the desired behavior can be approximated with a Membrane around the object attached to `__proto__`. The only differences in behavior are that:
1. This proposal will not use proxies,
2. This proposal is expected to work for all objects,
3. This proposal is expected to "replay" an object definition to produce the copy, and
4. The membrane example cannot flag prototype objects to facilitate the 3rd rule above.

Below is an example version of the desired behavior.

```js
//Defined to trigger new behavior
Object.defineProperty(Symbol, "SafeProto", { value: Symbol("SafeProto")});
//Defined to ensure existing behavior
Object.defineProperty(Symbol, "UnsafeProto", { value: Symbol("UnsafeProto")});

if (!("getKnownPropertyDescriptor" in Object)) {
  Object.defineProperty(Object, "getKnownPropertyDescriptor", {
    value: function getKnownPropertyDescriptor(obj, prop) {
      let retval;
      
      if (prop in obj) {
        while (obj && !retval) {
          retval = Object.getOwnPropertyDescriptor(obj, prop);
          if (!retval) {
            obj = Object.getPrototypeOf(obj);
          }
        }
      }
      
      return retval;
    }
  });
}

function makeSafeProto(inst) {
  const ROOT = Symbol("ROOT");          //Top of the tree for the property
  const OWNER = Symbol("OWNER");        //Object on which to place copy
  const PARENT = Symbol("PARENT");      //Copy 1 level up or null if root
  const PROPERTY = Symbol("PROPERTY");  //Name of the property for this copy
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
  
  function makeInstanceProperty(target) {
    function copyBack(t) {
      let {[PARENT]:parent, [PROPERTY]:property} = t;
      delete target[ROOT];
      delete target[OWNER];
      delete target[PARENT];
      delete target[PROPERTY];
      parent[property] = target;
      return parent;
    }
  
    let {[ROOT]:root, [PARENT]:parent, [PROPERTY]:property} = target;
    while (parent && (parent !== root)) {
      target = copyBack(target);
      ({[ROOT]:root, [PARENT]:parent, [PROPERTY]:property} = target);
    }
    if (parent === root) {
      let {[OWNER]:owner} = target;
      let def = Object.getOwnPropertyDescriptor(parent, property);
      def.value = target;
      Object.defineProperty(owner, property, def);
    }
  }
  
  let handler = {
    protos: new WeakMap(),
    copies: new WeakMap(),
    get(target, prop, receiver) {
      let def = Object.getKnownPropertyDescriptor(target, prop);
      let retval = Reflect.get(target, prop, receiver);

      if (!(Symbol.UnsafeProto in target) && (Symbol.SafeProto in target)
          && (handler.protos.get(target) !== receiver) && def
          && ("value" in def) && isObject(retval)) {
        let isRoot = !handler.copies.has(target);
        let result = Object.assign({
          [ROOT]: (isRoot) ? target: handler.copies.get(target)[ROOT],
          [OWNER]: inst,
          [PARENT]: (isRoot) ? Object.assign(target, {
            [OWNER]: inst,
            [PROPERTY]: prop
          }) : target,
          [PROPERTY]: prop,
          [Symbol.SafeProto]: void 0
        }, retval);
        retval = new Proxy(result, handler);
        handler.copies.set(result, retval);
      }

      return retval;
    },
    set(target, prop, value, receiver) {
      let retval = Reflect.set(target, prop, value, receiver);
      makeInstanceProperty(target);
      return retval;
    },
    defineProperty(target, prop, def) {
      let retval = Reflect.defineProperty(target, prop, def);
      makeInstanceProperty(target);
      return retval;
    },
    deleteProperty(target, prop) {
      let retval = Reflect.deleteProperty(target, prop);
      makeInstanceProperty(target);
      return retval;
    },
    preventExtensions(target) {
      let retval = Reflect.preventExtensions(target);
      makeInstanceProperty(target);
      return retval;
    },
    setPrototypeOf(target, prototype) {
      let retval = Reflect.setPrototypeOf(target, prototype);
      makeInstanceProperty(target);
      return retval;
    }
  };
  
  //Make the prototype a proxy object
  let proto = Object.getPrototypeOf(inst);
  //Wrap the prototype to protect its object properties.
  let newProto = new Proxy(proto, handler);
  Object.setPrototypeOf(inst, newProto);
  handler.protos.set(proto, newProto);
  return inst;
}

var test = makeSafeProto({
  a: 1,
  __proto__: {
    [Symbol.SafeProto]: void 0,
    b:2,
    c: {
      d:'x'
    }
  }
});
console.log(`Before test #1: test = ${JSON.stringify(test, null, '\t')}`);
test.c.e = 1;
console.log(`After test #1: test = ${JSON.stringify(test, null, '\t')}`);
delete test.c;
console.log(`Before test #2: test = ${JSON.stringify(test, null, '\t')}`);
Object.getPrototypeOf(test).c.e = 2;
console.log(`After test #2: test = ${JSON.stringify(test, null, '\t')}`);
```
