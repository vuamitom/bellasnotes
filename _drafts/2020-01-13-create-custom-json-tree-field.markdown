---
layout: post
title:  "Create custom tree field widget for json data with React"
date:   2020-01-13 00:12:33 +0700
tags: [odoo,custom field, reactjs, preact]
---

Joining tables in SQL is costly. Consider a database in which we store survey question templates. The normalized way would be to have a `template` table and a `question` table, with each question having a foreign key to the template that owns it. So whenever there is a need to render a survey, Odoo would need to query database twice, once for the `template` table and once for the `question` table. Sometimes, instead of the usual one to many relationship, we can optimize for read operations by lumping all related records into a json text field. The approach would suffer from low write throughput due to the need to lock the `template` table just to add a question. In reality, survey template is normally created once and read many times, that does not sound too bad a trade off. So in this post, we will look into rendering a json field as tree view in Odoo. 

Again, this is straight forward to do with Odoo. First we implement a custom widget which would take json data and render them as tree view. Secondly, we specifiy the field that should use that widget for rendering. 

### 1. Create a custom widget. 

All custom field widget in Odoo extends the `AbstractField` widget and override neccessary method. 

```javascript
odoo.define('beolla_survey.fields', function(require) {
"use strict"

var registry = require('web.field_registry');
var AbstractField = require('web.AbstractField');
var TemplateView = require('beolla_exam.template_view'); 
var JsonTreeField = AbstractField.extend({
    events: _.extend({}, AbstractField.prototype.events, {
        'click': '_onClick'
    }),
    //--------------------------------------------------------------------------
    // Private
    //--------------------------------------------------------------------------

    /**
     * Display button
     * @override
     * @private
     */
    _render: function () {
        // if (this.value) {
        //     var className = this.attrs.icon || 'fa-globe';

        //     this.$el.html("<span role='img'/>");
        //     this.$el.addClass("fa "+ className);
        //     this.$el.attr('title', this.value);
        //     this.$el.attr('aria-label', this.value);
        // }
        TemplateView.render(TemplateView.tableView, this.$el[0]);
    },

    //--------------------------------------------------------------------------
    // Handlers
    //--------------------------------------------------------------------------

    /**
     * Open link button
     *
     * @private
     * @param {MouseEvent} event
     */
    _onClick: function (event) {
        event.stopPropagation();
        window.open(this.value, '_blank');
    },

});

registry.add('json_tree', JsonTreeField);
})
```