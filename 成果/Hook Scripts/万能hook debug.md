```
// ==UserScript==
// @name         防止 debugger Hook
// @namespace    Anti-Debugger
// @version      1.0
// @description  Hook Function 构造防止 debugger 调用
// @author       You
// @match        https://*/*
// @match        http://*/*
// @grant        none
// ==/UserScript==

(function () {
    'use strict';
    console.log("Anti-Debugger Hook 初始化成功");

    // 保存原始 Function 构造器
    const originalConstructor = Function.prototype.constructor;

    // 重写构造器，拦截含有 "debugger" 的字符串
    Function.prototype.constructor = function (...args) {
        if (args.length && typeof args[0] === 'string' && args[0].includes("debugger")) {
            console.warn("已拦截尝试构造包含 debugger 的 Function：", args[0]);
            return function () { /* 被拦截 */ };
        }
        return originalConstructor.apply(this, args);
    };
})();
```

