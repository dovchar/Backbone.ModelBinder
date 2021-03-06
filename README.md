Special thanks to Derick Bailey for providing unit tests and the predecessor to this plugin.

This simple, powerful plugin helps you automatically synchronize your Backbone Views and Models.

Your Backbone apps already synchronize your models and views but you usually have to write a lot of boiler plate code to make it happen - this plugin helps you eliminate that boiler plate code.

The core of the plugin is a simple javascript class named the `Backbone.ModelBinder`.

`ModelBinders` will live inside of your Backbone javascript files - typically a Backbone.View.

Usually your html will not require any modification - this is different than most of the other model binders out there such as `Knockout` that require modification to your html.

## Simple but powerful
The ModelBinder uses the same jQuery event binding mechanism that Backbone relies on to handle events on Views so it should be pretty easy to understand.

The `ModelBinder` should be flexible enough to handle most situations you'll encounter including:

* Deeply nested models and views
* Partial view binding (only some html elements are bound while others are ignored by the binder)
* Easy formatting and type conversion
* Binding a model's attribute to multiple html elements
* Binding to any html attribute (Color, size, font etc.)
* Dynamic re-binding when swapping models


<br>
## Typical boilerplate code - the ModelBinder helps you get rid of most of this
**(Skip to the next section if your proficient with Backbone)**

A typical Backbone app will have code that looks something like this...

````
TypicalView = Backbone.View.extend({
    initialize: function{
        this.model.on('change:address', this._onModelAddressChange, this);
    },

    _onModelAddressChange: function(){
        this.$('#address').val(this.model.get('address'));
    }
});
````

In the ex. above the view is registering for when the address changes and will update the view appropriately.
In some smaller apps registering for model changes might not always be necessary.

<br>
In most apps the event handlers aren't so fine grained - typically if any model attribute changes the whole view is re-rendered as shown below. This could be wasteful for some situations...

````
TypicalView = Backbone.View.extend({
    initialize: function{
        this.model.on('change', this.render, this);
    },

    render: function(){
        // Entire view
    }
});
````

<br>
More commonly, if a form is updated we need to update our models...

````
TypicalView = Backbone.View.extend({
    events: {
        'change #address', '_onAddressChanged'
    },

    _onAddressChanged: function(){
        this.model.set({address: this.$('#address')});
    }
});
````

In the example above, we use Backbone's event block to register for when a element with an id="address" changes.
In the `_onAddressChanged` handler we update the Model's address attribute with the new value in the view element.

Some applications don't copy values from the View to the Model until the user clicks on a submit button and then perform a simple copy from all field elements to the Model attributes.


<br>
## Simple Example of the ModelBinder
ModelBinder has a no argument constructor and 2 public functions - `bind()` and `unbind()`.
An example of how to use `bind()` is shown below.

````
// Snippet from the html template file - used to create your template() used in the render() function
<input type="text" name="address"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },

    render: function(){
        this.$el.append(this.template());
        this.modelBinder.bind(this.model, this.el);
    }
});
````

The `bind()` function takes 3 parameters.  The 3rd parameter is optional.

* The Backbone Model your binding to.
* An html `rootElement` that should contain all of the other elements your binding to - probably a form, div or span element.
* A 3rd optional parameter called the bindingsHash is also possible - this is reviewed in the next section.

The `bind()` function finds all of the child elements under `rootElement` that have a `name` attribute defined.
**It then binds those child elements to Models attributes with the same `name`.**

For many simple views that that have elements with a `name` attribute that matches a Model's attribute name, this technique is sufficient.
Otherwise, you'll need to define the `bindingsHash` discussed in the next section.

Note that the ModelBinder does not care about Backbone Views, although you'll typically create ModelBinders inside of Views.


<br>
## The Bindings Hash

The `ModelBinder.bind()` function can take a `bindingsHash` as a 3rd optional parameter.
The `bindingsHash` uses a very similar format as the Backbone View events block.
It relies on jQuery selectors to locate which html elements to bind to.
Here is how the previous simple example will look like with a `bindingsHash` to achieve the exact same binding behavior.

````
// Snippet from the template file
<input type="text" name="address"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },

    render: function(){
        this.$el.append(this.template());

        var bindingsHash = {address: '[name=address]'};
        this.modelBinder.bind(this.model, this.el, bindingsHash);
    }
````

<br>
## Basic Bindings Hash syntax
The `bindingsHash` follows this basic structure:

````
    bindingsHash: {

        // Basic syntax
        'modelAttributeName' : 'jQuerySelector',

        // If your binding to an html element with name="address"
        'address'            : '[name=address]',

        // If your binding to an html element with id="phone"
        'phone'              : '#phone'
    }
````

The `bindingsHash` can take any jQuery selector to locate which html element(s) to bind to - just like the Backbone View events block.

<br>
The `bindingsHash` can also define multiple html selectors with an array as shown below.

````
// Snippet from the template file
<span name="pageTitle"/>
<input type="text" name="address"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },
    render: function(){
        this.$el.append(this.template());
        var bindingsHash = {address: [ '[name=address]', '[name=pageTitle]' ]};
        this.modelBinder.bind(this.model, this.el, bindingsHash);
    }
````

In the example above, address will be bound to both the input element and the span with the name pageTitle.
Both elements will be updated with the Model's address attribute changes.

<br><br>
## Bindings Hash syntax - Converters
You can also define `converters` with your bindings.
`Converters` are just functions that allow you to keep your views formatted differently than your Model attributes or perform type conversion.

All previous examples just defined a jQuery selector without explicitly naming it `'selector'` but if you pass in multiple options you must specify the `selector` with a name.
The example below shows a `converter` doing simple formatting.

````
  <input name="phoneNumber"/>

  // from inside a View.render() function
  var binder = new Backbone.ModelBinder();

  // This converter function can be defined anywhere, for simplicity it's just defined inline
  var phoneConverter = function(direction, value){
    if (direction === Backbone.ModelBinder.Constants.ModelToView) {
      if (value.length == 7){
        return value.substring(0, 3) + '-' + value.substring(3, 7);
      }
      else{
        return value;
      }
    }
    else {
      return value.replace(/[^0-9]/g, '');
    }
  };

  var bindingsHash = {phoneNumber: [{selector: '[name=phoneNumber]', converter: phoneConverter}]}
  binder.bind(this.model, this.el, bindingsHash);
````

A `Converter` is simply a function that takes a `direction` and a `value` as parameters and should return a converted value.
The direction will either be ModelToView or ViewToModel.  This allows your Model's attributes to remain in a pristine state but the view to format them appropriately.


<br><br>
You can also use `Converters` for more advanced operations like easily selecting a nested Model.

````
  <select name="nestedModel">
    <option value="">Please Select Something</option>
    <% _.each(nestedModelChoices, function (modelChoice) { %>
      <option value="<%= modelChoice.id %>"><%= modelChoice.description %></option>
    <% }); %>
  </select>

  // From inside a View.render() function
  // An example of what might be passed to the template function
  var nestedModelChoices = [{id: 1, description: 'This is One'}, {id: 2, description: 'This is Two'}];

  var binder = new Backbone.ModelBinder();

  var bindingsHash = {nestedModel: { selector: '[name=nestedModel]',
                                     converter: new Backbone.ModelBinder.CollectionConverter(nestedModelChoices).convert} }
  binder.bind(this.model, this.el, bindingsHash);
````

Here, the `converter` is leveraging the Backbone.ModelBinder.CollectionConverter - this converts Backbone Models to ids.
The select element's values are defined with the possible Model's ids.
The net result is that the nested Model will be whatever the user selected in the view with little effort.


<br><br>
## Bindings Hash syntax - Binding to any html attribute with `elAttribute`

All previous example bound to the text of the html elements but you can also bind to attributes like Color, Enabled, Size etc.

````
// Snippet from the template file
<input type="text" name="address"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },
    render: function(){
        this.$el.append(this.template());

        var bindingsHash = {isAddressEnabled: {selector: '[name=address]',  elAttribute: 'enabled'}};
        this.modelBinder.bind(this.model, this.el, bindings);
    }
````

In the example above, we bound the Model.isAddressEnabled property to the address element's enabled attribute.
You could also extend this to html element colors, sizes, the sky is the limit!


<br><br>
## Exposing the Power of jQuery Selectors, Selecting by Classes etc.
Binding definitions simply use jQuery.  You can select based off of a class attribute or anything else you'd like.

````
// Snippet from the template file
<input type="text" class="partOne" name="address"/>
<input type="text" class="partOne" name="phone"/>
<input type="text" class="partOne" name="fax"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },
    render: function(){
        this.$el.append(this.template());
        var bindingsHash = {isPartOneEnabled: {selector: '[class~=partOne]',  elAttribute: 'enabled'}};
        this.modelBinder.bind(this.model, this.el, bindingsHash);
    }
````

In this example, all 3 html elements enabled attribute are bound to the Model's isPartOneEnabled attribute.
This is because the jQuery selector '[class~=partOne]' returned all 3 elements.



<br><br>
## ModelBinder Scoping Rules and Name Conflicts
<br>
The ModelBinder.bind() function takes a root html element as an input parameter.
All bound html elements will need to exist under this root element.
Sometimes you'll have fields that might share the same name on the same page like the field 'identifier' shown below.

````
  // Snippet from the template file
  <span id="personFields">
    <input type="text" name="address"/>
    <input type="text" name="identifier"/>
  </span>
  <span id="invoiceFields">
    <input type="text" name="invoiceNumber"/>
    <input type="text" name="identifier"/>
  </span>

  SomeView = Backbone.View.extend({
      initialize: function(){
          this.personInfoBinder = new Backbone.ModelBinder();
          this.invoiceBinder = new Backbone.ModelBinder();
      },
      render: function(){
          this.$el.append(this.template());
          this.personInfoBinder.bind(this.personModel, this.$('#personFields'));
          this.invoiceBinder.bind(this.invoiceModel, this.$('#invoiceFields'));
      }
````

In this example, each binder takes a different root element that each have an element where name="identifier".
Since the elements are under their own scope there is no conflict.

<br><br>
## Multiple ModelBinders in a Single View
You can use as many model binders as you want to in a view.
In this example, the personInfoBinder binds appropriate elements to the personModel and the invoiceBinder binds the correct elements to the invoiceNumber.
The next example shows how to make this work in an easier way.

````
 // Snippet from the template file
 <input type="text" name="address"/>
 <input type="text" name="phone"/>
 <input type="text" name="invoiceNumber"/>

 SomeView = Backbone.View.extend({
     initialize: function(){
         this.personInfoBinder = new Backbone.ModelBinder();
         this.invoiceBinder = new Backbone.ModelBinder();
     },
     render: function(){
         this.$el.append(this.template());
         this.personInfoBinder.bind(this.personModel, this.el, {address: '[name=address]', phone: '[name=phone]' });
         this.invoiceBinder.bind(this.invoiceModel, this.el, {invoiceNumber: '[name=invoiceNumber]');
     }
````


<br>
Model values are copied to the view on when bind() is invoked. 
In the example below, the address html element will have the value of '1313 Mockingbird Lane' right after the .bind() function is invoked.
View values are not copied to model attributes at bind() time.  The appropriate place to initialize models is with the Backbone Model defaults block.

````
<input type="text" name="address"/>

SomeView = Backbone.View.extend({
    initialize: function(){
        this.modelBinder = new Backbone.ModelBinder();
    },
    render: function(){
        this.$el.append(this.template());
        this.model.set({address: '1313 Mockingbird Lane'});
        this.modelBinder.bind(this.model, this.el);
    }
````

<br>
You can call bind multiple times with different models.  
Calling bind will automatically internally call the model binder unbind() function to unbind the previous model.



<br>
When a view closes you should call the ModelBinder.unbind() function.





# Legal Info (MIT License)

Copyright (c) 2012 Bart Wood

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.