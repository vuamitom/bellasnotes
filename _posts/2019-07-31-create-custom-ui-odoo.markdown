---
layout: post
title:  "Create a single page React app with Odoo custom module"
date:   2019-07-31 00:16:33 +0700
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

### 2. Specify custom js and css

Add a custom asset bundler to transpile jsx with babel.

### 3. Create api

TODO: put my answer here https://www.odoo.com/forum/help-1/question/possible-to-put-react-js-library-with-odoo-116416
