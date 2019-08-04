---
layout: post
title:  "Make api request in Odoo without jQuery"
date:   2019-08-03 21:16:33 +0700
tags: [odoo,performance]
---

Odoo is a powerful workhorse that comes with a ton of functionalities built-in. That sometimes comes at a cost of bloated code with more functionalities than needed. For example, the common javascript bundle `web.assets_common.js` is pretty large (about 1MB), including anything from jQuery, Bootstrap to QWeb. If our aim is to create an independent front-end client with a UI framework of choice, say React, why bother including all that. So in this post, we will be trying to create a very thin api client to Odoo's backend. 

![Odoo common web bundle is large](/content/images/odoo-web-common-js.png)

Complete code sample for this example can be found [here](https://github.com/beolla/samples/tree/master/todo).

### 1. Creating a simple Rpc object.

Constructing a Rpc object to simulate behavior of Odoo's web client request is pretty straight forward. We make use of the new Javascript's Fetch Api for simplicity. 

```javascript

class Rpc {

    _buildQuery(options) {
        // refer to Odoo's addons/web/static/src/core/js/rpc.js
    }

    query(params, options) {
        let query = this._buildQuery(params),
            id = Math.floor(Math.random() * 1000 * 1000 * 1000),
            data = {
                jsonrpc: "2.0",
                method: 'call',
                params: query.params,
                id: id
            },
            payload = {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(data, date_to_utc)
            };
        payload = Object.assign(Object.assign({}, options), payload);
        return new Promise((resolve, reject) => {
            fetch(query.route, payload).then(res => res.json()).then(jsonRes => {
                if (jsonRes.error) {
                    reject(jsonRes.error);
                }
                else {
                    resolve(jsonRes.result);
                }
            }).catch(reject);

        });
    }
}
```

### 2. Using it in a sample React based Odoo module 

In previous example on [creating a React based Todo Odoo module](/create-react-ui-odoo/), newly created item is just kept ephermerally. Nothing is saved once the user leaves the page. We can make use of our service call to save item to server and fetch them on page load. 

```javascript

class TodoApp extends React.Component {
  constructor(props) {
    super(props);
    this.state = { items: [], text: '' };
    this.handleChange = this.handleChange.bind(this);
    this.handleSubmit = this.handleSubmit.bind(this);
    this.rpc = new Rpc();
    this.model = 'todo.todo';
    this.context = {lang:"en_US",tz:"Europe/Brussels",uid:2};
    this.fetch();
    
  }

  fetch() {
    var params = {
        model: this.model,
        context: this.context,
        method: 'search_read',
        fields: ['id', 'name']
    };

    this.rpc.query(params)
      .then(res => {        
        let items = [];
        for (let i = 0; i < res.length; i ++) {
          items.push({id: res[i].id, text: res[i].name});
        }

        this.setState(state => ({
          items: items,
          text: ''
        }));
      })
      .catch(e => {
        console.error(e);
      });
  }
  ...
}
```

![Our custom js size](/content/images/odoo-todo-js.png)

We end up with a much smaller js bundle. Granted that our is simpler than the main Odoo's web client, which requires much more to be packed in its Javascript bundle. But that is exactly the reason why we should slim it down when building our own custom app. 
