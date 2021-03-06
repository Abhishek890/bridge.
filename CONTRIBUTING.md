Thank you for considering to contribute to *bridge.*
Please follow these rules when making contributions to this project.

## Starting development
Run ```npm i``` inside the directory you've cloned bridge. to. You can now use ```npm run dev``` and ```npm run build```
to build versions of bridge. and start the development environment.

## Code Rules
### Verified as working
All code contributed to this repository should be verified as working, meaning you've tested the
functionality at least once and didn't encounter unexpected behaviour. Your build also may not contain errors inside the console.

### Code Style
#### General
Although not required by the language JavaScript itself, you should end all instructions with a ";". 
This avoids unnecessary bugs with how the code is interpreted.

#### Names
Variable, constant and parameter names should all be snake_case. This also includes Vue's computed properties.

```javascript
// Example
let my_var = "Hello World!";
```

Method and function names are prefered to be CamelCase.
```javascript
// Example
function doSomethingNow(par_1, par_2) {
  doSomething(par_1 + par_2);
}
```

#### Method Chaining
Wherever it makes sense, you should chain methods by inserting a new line first. This makes the code a lot easier to read.

```javascript
// Example
doSomething()
  .then(data => data.string())
  .then(str => console.log(str))
  .catch(err => console.error(err))
  .finally(updateUI());
```
