```javascript
let obj = { [Symbol('test')]: 'abc', a: 33, [Symbol('test2')]: '123' }
Object.getOwnPropertySymbols(obj)  // (2) [Symbol(test), Symbol(tt)]
```