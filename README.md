# proposal-safe-prototype
An ES proposal to fix the foot-gun of non-function objects in prototypes.
---

## Motivation
ES is a prototype-based language, but as of late, many new features have been proposed, some of which attempt to obviate the use of the prototype for its intended purpose as the template for an instantiation by `new`. The sole reason is because of the lack of a copy-on-write semantic for prototype properties with a non-function object as the value.

## Solution
This proposal is fairly straight forward, but may not be easy to implement. The idea is to implement a copy-on-write semantic for any object assigned to `__proto__` of another object. In the interest of preserving as many as possible of the use-cases putting the foot-gun behavior to good use, there are a few limitations.
* Only objects created via the `new` keyword will receive the new behavior. This excludes the return value of constructors when that value is not `this`.
* The new behavior is not affected by `"use strict"`.

## Issues
There will almost certainly be a few cases where the foot-gun behavior was used constructively on objects created with `new`. However, these occurances are expected to be exceedingly few and far between. The common practice of declaring object properties in the constructor should mean that occurances of the foot-gun behavior are rare enough that this change will not constitute a major behavioral break.

## Behavior
It's that 2nd step that's so problematic. The behavior would be as follows:
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
    copies: new WeakMap(),
    get(target, prop, receiver) {
      let def = Object.getKnownPropertyDescriptor(target, prop);
      let retval = Reflect.get(target, prop, receiver);

      if (def && ("value" in def) && isObject(retval)) {
        let isRoot = !handler.copies.has(target);
        let result = Object.assign({
          [ROOT]: (isRoot) ? target: handler.copies.get(target)[ROOT],
          [OWNER]: inst,
          [PARENT]: (isRoot) ? Object.assign(target, {
            [OWNER]: inst,
            [PROPERTY]: prop
          }) : target,
          [PROPERTY]: prop
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
  return inst;
}

var test = makeSafeProto({a:1, __proto__: {b:2, c: { d:'x'}}});
console.log(`Before: test = ${JSON.stringify(test, null, '\t')}`);
test.c.e=1;
console.log(`After: test = ${JSON.stringify(test, null, '\t')}`);
```
