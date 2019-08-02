---
layout: post
title:  "Create a single page React app with Odoo custom module"
date:   2019-08-02 00:16:33 +0700
tags: [odoo,react]
---

Odoo add-on allows very flexible customization. For most custom modules, adding an form view or table view to admin page is enough. However sometimes the custom module may need to create an entirely different UI layout (think Gantt chart, custom dashboard...). In this post, we are going through the steps required to add a non-standard interface to Odoo, a single page React based To-do app. The purpose is, in other words, to create a scaffold for standalone screen similar to that of the `point-of-sale` module. 

As usual, we start with an empty module scaffold:

```bash
./odoo-bin scaffold todo ~/src/odoo-addons/
```
### 1. Create controller routes.

Firstly, let odoo knows the endpoint of main html page. Unlike regular Odoo module, our single page todo app resides in a separate url endpoint on its own. To add the route, open `controllers/controllers.py`:

```python
# -*- coding: utf-8 -*-
from odoo import http
from odoo.http import request
import json

class Todo(http.Controller):
    @http.route('/todo/todo/', auth='public')
    def index(self, **kw):
        context = {
            'session_info': json.dumps(request.env['ir.http'].session_info())
        }
        return request.render('todo.index', qcontext=context)
```
Notice that `todo.index` is a template that specifies html content of the page. For the purpose of this demo, it contains referrences to external javascript files, including ReactJS sources and our own javascript code to instantiate React app. 

```xml
<template id="index" name="todo index">&lt;!DOCTYPE html&gt;
	<html>
	        <head>
	        <title>Custom React Todo</title>
	        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1"/>
	        <meta http-equiv="content-type" content="text/html, charset=utf-8" />               
	    </head>
	    <body>
	        <div class="o_main_content" id="todos-example"/>
	    </body>

	    <script src="https://unpkg.com/react@16/umd/react.development.js" ></script>
	    <script src="https://unpkg.com/react-dom@16/umd/react-dom.development.js" ></script>
	        
	    <t t-call-assets="todo.assets"/>
	</html>
</template>
```

### 2. Specify custom javascript

Our main custom javascript code resides in the above `<t t-call-assets="todo.assets"/>`. It tells Odoo QWeb renderer that it should inject the javascript bundle `todo.assets` at that line in the output html. 

```xml
<template id="assets" name="todo assets">
    <script type="text/javascript" src="/todo/static/src/jsx/main.jsx"></script>
</template>
```

Since the source code was in jsx, which is not supported out of the box by Odoo asset bundler, there are two way to work around this. The manual way is to run Babel transpiler on the source file and include the generated javascript source instead. A automated but more complicated way is to override Odoo asset bundler's with our own and call Babel transpiler before bundling as detailed in our [previous post](/run-babel-bundler-odoo/). That would requires the production environment to have `babel-cli` installed though.

Either way, once the javascript resource is included, we can checkout the result page at `/todo/todo`. 

![](/content/images/react-odoo-todo-sample.png)

### 3. Future improvements

Though we have created a React based Todo app, there are a few other things that can be done, such as saving the todo data to Odoo's backend. That would require setting up api endpoints, and authenticating service calls with Odoo. Odoo's web clients comes with `web.assets_backend` bundle to make api calls, but it is a little bit bloated. Next time we can look into way of making our own requests without using that huge bundle. 
