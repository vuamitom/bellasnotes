---
layout: post
title:  "Create custom tree field widget for json data with React"
date:   2020-01-20 21:12:33 +0700
tags: [odoo,custom field, reactjs, preact]
---

Joining tables in SQL is costly. Consider a database in which we store survey question templates. The normalized way would be to have a `template` table and a `question` table, with each question having a foreign key to the template that owns it. So whenever there is a need to render a survey, Odoo would need to query database twice, once for the `template` table and once for the `question` table. Sometimes, instead of the usual one to many relationship, we can optimize for read operations by lumping all related records into a json text field. The approach would suffer from low write throughput due to the need to lock the `template` table just to add a question. In reality, survey template is normally created once and read many times, that does not sound too bad a trade off. So in this post, we will look into rendering a json field as tree view in Odoo. 

In order to store survey data, we will use a single model as below. Survey's questions would be stored in a single json field `content`.

```python
class BeollaTemplate(models.Model):
    """Odoo model to store survey data"""
    _name = 'beolla_survey.template'
    name = fields.Char(required=True)
    # json data field
    content = fields.Text()
```

Again, this is straight forward to do with Odoo. First we implement a custom widget which would take json data and render them as tree view. Secondly, we specifiy the field that should use that widget for rendering. Conceptually, our widget would need to do two things, render the json data as html, and handle user's input to update that data accordingly.


### 1. Create a custom widget which display Json content using table view. 

All custom field widget in Odoo extends the `AbstractField` widget. To render the json data as table view, we need to override `renderReadonly` and `renderEdit`, which render the field in readonly mode and edit mode respectively. Here we can use any javascript view framework of choice (Reactjs, Preactjs or VueJs...). I prefer PreactJs due to its lightweightedness. Please find more information on how to integerate PreactJs with Odoo in the previous post on [running babel bundler](/run-babel-bundler-odoo/).

```javascript
odoo.define('beolla_survey.fields', function(require) {
"use strict"

var registry = require('web.field_registry');
var AbstractField = require('web.AbstractField');
var PreactViews = require('beolla_survey.preact_views'); 
var JsonTreeField = AbstractField.extend({

    /**
     * render in readonly mode
     * @override
     */    
    _renderReadonly: function () {
        this._renderTableView(true);
    },

    /**
     * render in edit mode
     * @override
     */
    _renderEdit: function() {
        this._renderTableView(false);
    },

    /**
     * Display a table view for json data     
     * @private
     */
    _renderTableView: function() {
        var questions = this.value? JSON.parse(this.value): [];
        var questionView = <QuestionList key={'question-list'} 
                                readonly={readonly}
                                create={this._createQuestion.bind(this)}
                                items={questions}/>        
        if (this.$el[0].children.length > 0) {
            // tell preact to update existing nodes
            preact.render(questionView, this.$el[0], ...this.$el[0].children);
        }
        else {
            preact.render(questionView, this.$el[0]);
        }
    }

});

// add the custom widget to Odoo's widget registry
registry.add('json_tree', JsonTreeField);
})
```
`<QuestionList/>` is a preact view class. Its aim is to replicate the look and feel of Odoo's table view. Please refer to the end of this post for [detailed implementation](#questionlist). Notice that we need to pass a `_createQuestion` method to the view, which will be responsible for adding new question to the template. 


### 2. Update field value with new question popup form. 

Similar to Odoo's built-in table view, our custom view has a `Add a question` button at the end, which shows a pop-up form to create a new question. Once user clicks save, question data will be updated to the model's `content` field by the `_saveQuestion` method.

```javascript
    /**
     * Show question form dialog when user click `add new question`
     */
    _createQuestion: function(evt) {
        evt.stopPropagation();
        evt.preventDefault();   
        // this.setState({activeQuestion: {id: 'new'}});
        new QuestionEditDialog(this, 
                {id: 'new'}, 
                this._saveQuestion.bind(this)
            ).open();
    },

    /**
     * Update question value 
     */
    _saveQuestion: function(val) {
        var items = this.value? JSON.parse(this.value): [];
        if (val.id === 'new')         
            items.push(val);
        else {
            for (var i = 0; i < items.length; i++) {
                if (items[i].id === val.id) {
                    items[i] = val;
                    break;
                }
            }
        }
        this.isDirty = true;
        if (!this.isDestroyed()) {
            return this._setValue(JSON.stringify(items));
        }
    }
```

`QuestionEditDialog` is basically a Odoo `Dialog` subclass to display the question edit form. Implementation of it is listed at the [end of this post](#questioneditdialog).

### 3. Using the widget

Last but not least, we will need to let Odoo know that it should use the custom widget `json_tree` to render `content` field. This is done by specifying the `widget` property of field in xml. 

```xml
<record id="view_beolla_survey_survey_template" model="ir.ui.view">
    <field name="name">beolla_survey.template.form</field>
    <field name="model">beolla_survey.template</field>
    <field name="arch" type="xml">
        <form string="Survey Template">
            <sheet>
                <div class="oe_button_box" name="button_box">
                </div>
                <div class="oe_title">
                  <label class="oe_edit_only" for="name" string="Template Name"/>
                  <h1><field name="name" placeholder="Template Name"/></h1>
                </div>                
                <notebook>
                  <page string="Questions">                   
                      <field name="content" widget="json_tree" />                    
                  </page>
                </notebook>
            </sheet>
        </form>
    </field>
</record>
```

And be sure to add your javascript file to Odoo's bundle. 

```xml
<template id="beolla_survey_json_tree_field" name="beolla_survey assets" inherit_id="web.assets_backend">
    <xpath expr="." position="inside">
        <script type="text/javascript"
                src="/beolla_survey/static/src/js/template.js"></script>
    </xpath>
</template>
```

And that is what we need to have a custom json based tree view.

### 4. Preact view classes

#### QuestionList

For more information on how to use custom js view frameworks in Odoo, please refer to the post. Though we used PreactJs, the same idea can be applied to ReactJs or VueJs. 


```javascript
class QuestionList extends preact.Component {
    render(props, state) {
        let questions = []
        for (let i = 0; i < props.items.length; i++) {
            let item = props.items[i];
            questions.push(<tr >
                <td>{'Question ' + (i + 1)}</td><td>{item.excerpt}</td>
            </tr>)
        }
        if (!props.readonly) {
            questions.push(<tr><td colspan={2}><a href='#' role='button' onClick={props.createQuestion}>Add a question</a></td></tr>)
        }
        if (questions.length < 4) {
            let empty = 4 - questions.length;
            for (let i = 0; i < empty; i++) {
                questions.push(<tr><td colspan={2}>&nbsp;</td></tr>)
            }
        }
        return (<div class='table-responsive'>
                <table class='o_list_view table table-sm table-hover table-striped o_list_view_ungrouped o_editable_list'>
                    <thead>
                        <tr><th class='o_column'>Order</th><th class='o_column'>Content</th></tr>
                    </thead>
                    <tbody class='ui-sortable'>                     
                        {questions}
                    </tbody>
                    <tfoot>
                        <tr><td></td><td></td></tr>
                    </tfoot>
        </table></div>);
    }
}

```

#### QuestionEditDialog

```javascript
var Dialog = require('web.Dialog');
var QuestionEditDialog = Dialog.extend({
    dialog_title: _t("Edit Question"),
    template: 'beolla_survey.QuestionEditDialog',

    /**
     * @override
     * @param {integer|string} channelID id of the channel,
     *      a string for static channels (e.g. 'mailbox_inbox').
     */
    init: function (parent, question, onSave) {
        this._question = question;
        this._onSave = onSave;

        this._super(parent, {
            size: 'medium',
            buttons: [{
                text: _t("Save"),
                close: true,
                classes: 'btn-primary',
                click: this._saveQuestion.bind(this),
            }, {
                text: _t('Cancel'),
                close: true
            }],
        });


    },
    /**
     * @override
     */
    start: function () {
        var self = this;
        // QuestionForm is a Preact View class
        // which is pretty simple to implement.
        preact.render(<QuestionForm question={this._question} ref={quesionForm=>this.quesionForm=quesionForm}/>, this.$el[0]);
        return this._super.apply(this, arguments);
    }

    _saveQuestion: function() {
        // get question
        
        var questionData = this.quesionForm.getQuestion();
        // console.log('Article data: ', outputData)
            // validate data
        let outputData = questionData.outputData,
            answers = questionData.answers;
        
        this._onSave({
            content: outputData,
            excerpt: excerpt,
            answers: answers,
            id: this._question.id
        })
        
    }
}
```

Source code of this demo will be uploaded after code cleaning up. 