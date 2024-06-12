---
date: 2019-09-11
updated: 2019-11-20
issueid: 8
tags:
title: TypeScript中import=require 和es6 module import区别
---
这个是在Github 看到的评论，感觉写的不错，就直接搬过来了

比较

`export default` 和 `import x = require('')`区别

### export default ... ([Default Export](https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export#Description))

```typescript
// calculator.ts                                // compiled.js
// =============                                // ===========
export default class Calculator {               // var Calculator = /** @class */ (function () {'
'    public add(num1, num2) {                    //     function Calculator() {}
        return num1 + num2;                     //     Calculator.prototype.add = function (num1, num2) {
    }                                           //         return num1 + num2;
}                                               //     };
                                                //     return Calculator;
                                                // }());
                                                // exports["default"] = Calculator;
```

### import ... from "module";

```typescript
// importer.ts                                  // compiled.js
// ===========                                  // ===========
import Calculator from "./calculator";          // exports.__esModule = true;
                                                // var calculator = require("./calculator");
let calc = new Calculator();                    // var calc = new calculator["default"]();
                                                // console.log(calc.add(2, 2));
console.log(calc.add(2, 2));                    //
```

##### Notes:

* A default export can be imported with any name.
* Functionally equivalent to `import * as Calculator from "./calculator";` and then instantiating it using `new Calculator.default()`.

---

### export = ...

```typescript
// calculator.ts                                // compiled.js
// =============                                // ===========
export = class Calculator {                     // module.exports = /** @class */ (function () {
    public add(num1, num2) {                    //     function Calculator() {}
        return num1 + num2;                     //     Calculator.prototype.add = function (num1, num2) {
    }                                           //         return num1 + num2;
}                                               //     };
                                                //     return Calculator;
                                                // }());
```

### import ... = require("module");

```typescript
// importer.ts                                  // compiled.js
// ===========                                  // ===========
import Calculator = require("./calculator");    // exports.__esModule = true;
                                                // var Calculator = require("./calculator");
let calc = new Calculator();                    // var calc = new Calculator();
                                                // console.log(calc.add(2, 2));
console.log(calc.add(2, 2));                    //
```

##### Notes:

* This syntax is only used when importing a CommonJS module.

---

### export ... ([Named Export](https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export#Description))

```typescript
// calculator.ts                                // compiled.js
// =============                                // ===========
export class Calculator {                       // exports.__esModule = true;
    public add(num1, num2) {                    // var Calculator = /** @class */ (function () {
        return num1 + num2;                     //     function Calculator() {}
    }                                           //     Calculator.prototype.add = function (num1, num2) {
}                                               //         return num1 + num2;
                                                //     };
                                                //     return Calculator;
                                                // }());
                                                // exports.Calculator = Calculator;
```

### import { ... } from "module";

```typescript
// importer.ts                                  // compiled.js
// ===========                                  // ===========
import { Calculator } from "./calculator";      // exports.__esModule = true;
                                                // var calculator = require("./calculator");
let calc = new Calculator();                    // var calc = new calculator.Calculator();
                                                // console.log(calc.add(2, 2));
console.log(calc.add(2, 2));                    //
```

##### Notes:

* Named exports are useful to export several values.
* During the import, you must use the same name of the corresponding object.
