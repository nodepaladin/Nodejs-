# Module  
> 稳定等级2 -Stable  

Nodejs有一个简单的模块载入系统。Nodejs中，文件和模块是一一对应的（每个文件被当做一个单独的模块）  
例如，文件<code>foo.js</code>：  
```
const circle = require('./circle.js');
console.log(`The area of a circle of radius 4 is ${circle.area(4)}`);
```  
<code>foo.js</code>的第一行导入了其同目录下的模块<code>circle.js</code>  
这是<code>circle.js</code>: 
```
const {PI} = Math;
exports.area = (r) => PI * r * r;
exports.circuference = (r) => 2 * PI * r;

```  
<code>circle.js</code>模块导出了方法area()和<code>circumference.js</code>。要在模块下挂载方法或者对象，可以将他们添加到<code>exports</code>
对象上。  
