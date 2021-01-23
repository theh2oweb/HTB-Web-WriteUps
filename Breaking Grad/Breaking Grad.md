# Breaking Grad

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/Challenge.png" width="700px">

In this challenge, we are provided with the source code and a Dockerfile to create a container and run the web app on our local host.
Once the application is up and running, we can access it in our browser at 127.0.0.1:1337

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/MainPage.png" width="600px">

Inspecting the source code, we see that it's a Node.js application using express. In routes/index.js we can see all available pages:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/routes%3Aindex.png" width="500px">


## Prototype Pollution Vulnerability

Looking around, I saw a function that caught my attention in helpers/ObjectHelper.js:

<img src="https://github.com/hacefresko/HTB-Web-WriteUps/blob/main/Breaking%20Grad/images/helpers:ObjectHelper.png" width="500px">

Merge functions usually lead to Prototype Pollution Vulnerabilities, so let's investigate it a bit.

As we can see in routes/index.js, when sending a POST request to '/api/calculate', first thing done is passing req.body to function ObjectHelper.clone.
This function calls the merge function passing as first parameter an empty object {} and as second parameter target, which is req.body.
So, at first glance, ObjectHelper.clone adds every property from req.body (every POST parameter) to an empty new object. 

This means that if we pass { \_\_proto\_\_ : { lol : 'lol' }}, we will set a new property named lol with value 'lol' to the \_\_proto\_\_ of this new object. 
Since it's a new empty object, the function which created it is Object, so target.\_\_proto\_\_ points to the Object prototype.

We can basically set any property we want to Object prototype by sending { \_\_proto\_\_ : { \<property\> : \<value of the property\> }} 
as req.body inside a POST request to '/api/calculate'.

However, we can bypass this. Every object has a constructor property, which points to the function which created it. Also, we can reffer to constructor.prototype, 
which is the prototype of the function which created the object. This means that our target.\_\_proto\_\_ is the same as target.constructor.prototype. 
So, our final payload to inject properties to Object is { constructor : { prototype : { \<property\> : \<value\> }}}

We can check that it works by sending a POST request to /api/calculate. Normally, this will return {"pass":"no0ooo00pe"} if the object student has either a property
'name' with value 'Baker' or 'Purvis' or a property 'paper' with value greater or equal than 10 (helper/StudentHelper.js). So, if we succesfully inject a property 'name' with value 'Baker'
in Object, the server will return {"pass":"no0ooo00pe"} although the object student doesn't have the property itself (because of inheritance and the 
prototype chain).

helper/StudentHelper.js:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/helpers%3AStudentHelper.png" width="400px">

Exploit used:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/exploit1.png" width="600px">

Result of the execution:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/exploitDemo1.png" width="400px">

## Remmote Code Execution

When accessing /debug/version, child_process.fork is executed inside helper/DebugHelper.js:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/helpers%3ADebugHelper.png" width="400px">

Looking at the documentation, fork accepts two interesting parameters: 

    execPath <string> Executable used to create the child process.
    execArgv <string[]> List of string arguments passed to the executable. Default: process.execArgv.

If we pass 'ls' as execPath, server will execute 'ls VersionCheck.js'. If we pass 'ls' as execPath and '.' as execArgv,
server will execute 'ls . VersionCheck.js'.

Function fork will look for this arguments in the object passed. If it doesn't have them, it will look for them in Object.
As we can set any property we want to Object, we can pass any argument we want to function fork, so we have Remote Code Execution :D

We can write a simple script to exploit this:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/exploit2.png" width="800px">

By passing 'ls' and '.' we see that the flag is in the same directory. We can get it by passing 'cat' and 'flag_name':

Result of 'ls . VersionCheck.js':

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/exploitDemo2.png" width="400px">

Result of 'cat flag_name VersionCheck.js':

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Breaking%20Grad/images/exploitDemo3.png" width="800px">
