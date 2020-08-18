# SharedMap

 * ***zero-dependency***
 * high-performance
 * unordered
 * Vanilla JS implementation of SharedMap,
 * a synchronous multi-threading capable,
 * fine-grain-locked with deadlock recovery,
 * static memory allocated,
 * coalesced-chaining HashMap,
 * backed by **SharedArrayBuffer**
 * that supports deleting
 * and is capable of auto-defragmenting itself on delete unless almost full
 * compatible with both Node.js and SharedArrayBuffer-enabled browsers

 [![License: LGPL v3](https://img.shields.io/badge/License-LGPL%20v3-blue.svg)](https://www.gnu.org/licenses/lgpl-3.0)
 [![Node.js CI](https://github.com/mmomtchev/SharedMap/workflows/Node.js%20CI/badge.svg)](https://github.com/mmomtchev/SharedMap/actions?query=workflow%3A%22Node.js+CI%22)
 [![codecov](https://codecov.io/gh/mmomtchev/SharedMap/branch/master/graph/badge.svg)](https://codecov.io/gh/mmomtchev/SharedMap)

## Introduction

Due to its legacy as a simple Web page glue, JS has what is probably the absolutely worst support of multi-threading among all languages created in the last few decades.

As the language matures, driven by a remarkably well implemented engine (V8) and the unique promise of unifying back-end, front-end and desktop application development, real multi-threading for CPU-bound tasks is becoming an absolute necessity.
In a true JS spirit, a feature after feature is added, some projects implementing it, others boycotting it, leaving it to the crowd to eventually decide what is worth supporting and what is not. As much as this can appear appalling to computer language experts, it is in a quite a way reminiscent of how the Linux kernel imposed itself vs the tech giants 20 years ago, and this is how JS is today on its way to total dominance as the leading general-purpose language.

The current situation with **SharedArrayBuffer** is a perfect example of this JS spirit. In the hope that at some point in the near future Firefox will re-enable it by default, and Safari will implement it, **SharedMap** is proposed as a working solution for computationally-heavy back-end programs executing in Node.js.

**SharedMap** is browser-compatible in theory, but on the front-end side, when one of the major browsers is completely missing (*Safari*), and another one requires the user to go through a security warning to enable an obscure feature (*Firefox*), its usefulness will be severely limited. An ES Module interface is provided and also TS definitions courtesy of [@takase1121](https://github.com/takase1121), so feel free to try it in your Chrome/Edge-exclusive project.

**SharedMap** was motivated by [igc-xc-score](https://github.com/mmomtchev/igc-xc-score), a linear optimization solver for scoring paragliding flights. When I started it, I initially tried Python because of its flawless multi-threading and then I slowly realized that the single-threaded V8 implementation was faster than the 4-way multi-threaded Python3 (and PyPy) implementation. Love it or hate it, JS is here to stay for the years to come.

## About

Because of the severe limitations that **SharedArrayBuffer** imposes, **SharedMap** supports only numbers and strings and uses fixed memory allocation. It uses synchronous locking, implemented on top of the **Atomics** interface.

The HashMap is a coalesced HashMap and has almost no performance drop up to 95% fill ratio and it is still usable up to 99.99%.
This chart shows the performance drop for a SharedMap with 370k English words and 4:8:1 ratio of *set/get/delete* operations:

![Performance Chart](https://gist.githubusercontent.com/mmomtchev/01f50eedac8d2a61346a9a0f373c24e4/raw/b49d0130e16c24137b56efa87540c84544b1630f/performance.png)

The default hash function is MurmurHash2 which works very well for words. You can provide your own hash function by override the *hash* property.

It supports deleting and will rechain itself when needed. The rechaining can be quite small and can be further optimized.

It supports single-line locking with deadlock recovery. Unless a deadlock is detected, *get* and *set* require only a shared global lock and lock exclusively no more than two lines so multiple operations can run in parallel. *delete* requires an exclusive lock and it is slow.

There are also thread-safe implementations of *map()* and *reduce()* and a public method allowing a program to temporarily lock out writers.

## Installation

```bash
npm install sharedmap
```

## Usage examples
```js
const SharedMap = require('SharedMap');

const MAPSIZE = 128 * 1024 * 1024;
// Size is in UTF-16 codepoints
const KEYSIZE = 48;
const OBJSIZE = 16;
const NWORKERS = require('os').cpus().length;

if (workerThreads.isMainThread) {
    const myMap = new SharedMap(MAPSIZE, KEYSIZE, OBJSIZE);
    workers = new Array(NWORKERS).fill(undefined);
    for (let w in workers) {
        workers[w] = new workerThreads.Worker('./test.js',
            { workerData: { map: myMap } });
    }
} else {
    // You can also send it through a MessagePort
    const myMap = workerThreads.workerData.map;

    // You have to manually restore the prototype
    Object.setPrototypeOf(myMap, SharedMap.prototype);

    myMap.set('prop1', 'val1');
    myMap.set('prop2', 12);

    // Numbers will be converted to strings
    console.assert(myMap.get('prop1') == 'val1');
    console.assert(myMap.get('prop2') == '12');

    // You can store objects if you serialize them
    myMap.set('prop3', JSON.Stringify('a'));

    myMap.delete('prop2');
    console.assert(myMap.hash('prop2') == false);
    console.assert(myMap.length === 1);

    // SharedMap.keys() is a generator
    for (let k of myMap.keys())
        // could fail if another thread deletes k under our nose
        console.assert(myMap.has(k));

    myMap.lockWrite();
    for (let k of myMap.keys())
        // will never fail, but locks out writers
        console.assert(myMap.has(k));
    myMap.unlockWrite();

    // Both are thread-safe without lock, but there could
    // be values added/deleted/modified while the operation runs
    // These operations will be atomic, so all values read will
    // be coherent
    // map.get(key)=currentValue is guaranteed while the callback runs
    //
    // Don't manipulate the map in the callback, see the explicit locking
    // example below if you need to do it
    const sum = map.reduce((a, x) => a += (+x), 0);
    const allKeys = Array.from(myMap.keys());

    // Update with explicit locking
    // Other threads can continue reading, set operations will be atomic
    // In real-life you will also handle the exceptions
    myMap.lockWrite();
    for (let k of myMap.keys({lockWrite: true}))
        myMap.set(k,
            myMap.get(k, {lockWrite: true}).toUpperCase(),
            {lockWrite: true});
    myMap.unlockWrite();
    
    myMap.clear();
}
```

## Manual

[jsdoc](https://mmomtchev.github.io/SharedMap/)

## TODO and contributing

* Avoid unncessary copying when rechaining on delete

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License
[LGPL](https://choosealicense.com/licenses/lgpl-3.0/)
