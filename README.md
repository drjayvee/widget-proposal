# Considerations #
Client-side web development practices fall along a continuum based on where between the the server and client the logic is implemented. Two common approaches are server-side rendering with progressive enhancement (PE) on the client, and client-side rendering. The latter is particularly useful for single-page applications.

## Server-side rendering with PE ##
This is the traditional approach to web development. Markup is rendered on the server and sent to the client in response to an HTTP request. Compared to the client-side rendering approach, the upsides of this approach are:

### Fast (initial) page load / display ###
Since no little to no javascript is required to display content, the browser can start displaying the content while it is still loading. Little to no flickering (caused by browser reflow) happens because the markup does not change.

The user can interact with the content during load.

This is especially beneficient to mobile users, who have a slow connection over which to download javascript files.

### Lightweight ###
Since less javascript needs to run, the application is less demanding on the client.

This is again beneficient to mobile users, since their devices have less processing power and run on a battery.

### Graceful degradation ###
Finally, the page content degrades gracefully in case some bits of javascript do not load at all or contain errors.

Again, mobile users benefit because their connections can be unstable.

### Future-proof ? ###
Since client-side technology evolves so fast and will continue to in the future, implementing view rendering on the stable server stack might be a way to future-proof applications.

### References ###
[Christian Heilmann: Overboard.js](https://youtu.be/Bvs5-AilZyY?t=33m57s)

# Initialization #
Users must be able to initialize widgets in multiple ways which fall along a continuum:
* programmatically (no existing markup)
* progressive enhancement (existing markup + scripting)
* declaratively (no scripting)

To ease progressive enhancement and (custom) styling, markup should be minimal.

Widgets should strive to modify existing markup as little as possible to keep styling and behavior.

## Programmatic ##
```js
new Widget.Button({disabled: true}).renderTo(document.querySelector('.container'))
```

## Progressive enhancement (PE) ##
This is the difficult one, because the library author has less control over the markup.

```html
<button class="widget-button">Go</button>  <!-- class is already set for styling -->
```
```js
new Button({
  srcNode: document.querySelector('button'),
  label: 'Stay'
});
```

The trick here is to sensibly parse the existing markup to extract property values (such as a button's label, checked state, etc). Properties specified programmatically override any parsed from markup.

Ideally, the user can write markup that is already styled appropriately on page load (i.e. before js is run) and is "stealthily" enhanced (i.e. quickly, without flashing and reflow operations).

Maybe PE is only sensible for some widgets?

## Declarative ##
```html
<div data-widget-type="ToggleButton" data-properties="checked: true, label: 'Go'"></div>
```
(Alternatively, `data-checked="true" data-label="Go` ?)

Instance is retrievable via `Widget.getByNode()`.

# Markup #
Markup should be kept simple (no tag soups) and should use native solutions where possible.

For example, the markup for a checked and disabled toggle button could be:
```html
<button class="widget widget-button widget-button-toggle" disabled checked>Label</button>
```
By using the native `button` element and setting its `checked` and `disabled` attributes, PE works seamlessly, query selection (for styling and scripting) is easy, and the UI is accessible (especially when enhanced with ARIA).

To keep the DOM in sync with the instance's properties, two-way bindings have to be set up (at least for a subset of the properties).

# API #
After initialization, the user must not make changes to the markup, and should not query markup to determine state. Instead, she has to call methods on the widget instance and / or register event handlers.

For example:
```js
// get the instance
let bt = Widget.getByNode(document.getElementById('myButton'));

// call methods
bt.disable()                         // alternatively: set('disabled', true) ?
  .setLabel('Can\'t touch this!');   // alternatively: set('label', '...') ?

// register event handlers
on(bt, 'click', event => { bt.enable(); });
```

# Example #
![screenshot](https://github.com/drjayvee/widget-proposal/blob/master/img/dd-expanded.png)

In this page, the user can manage permissions for a set of user groups, which are listed in a dropdown. If no groups have been selected (as is always the case the first time the page is loaded), the server will render the dropdown expanded.

The button which updates the page is always rendered disabled in order to force the user to change the selection. Clicking the button without first changing the selection will have no effect (but instead reload the same page). Therefore, a client-side script will listen for changes and enable the button if any group is selected.

Note that this feature does not degrade gracefully. If the javascript that enables the button does not work (fails to load, contains an error), the user will not be able to use the form at all!

## Server-side code ##
Here is (a part of) the server-side template ([twig](http://twig.sensiolabs.org/)):
```html
<form action="index.php" method="GET">
	<input type="hidden" name="mod" value="manageAccess">
	<div class="dropdown {% if selectedUserGroups is empty %}dropdown-expanded{% endif %}">
		<button class="yui3-button dropdown-label" type="button">User groups</button>
		<div class="dropdown-content">
			<ul class="dense searchable max-height">
			{% for id, name in usergroups %}
				<li><label><input type="checkbox" {% if id in selectedUserGroups %}checked{% endif %} name="userGroup[]" value="{{ id }}"> {{ name }}</label></li>
			{% endfor %}
			</ul>
			
			<div class="dropdown-footer">
				<button class="yui3-button action" disabled>Display</button>
			</div>
		</div>
	</div>
</form>
```

## Client-side code ##
On the client, a number of things happen:

### Dropdown state ###
The dropdown is displayed as expanded (even before the page finishes loading and any javascript has run) because the `dropdown-expanded` class is already present.

As soon as the javascript for the dropdown loads, it adds behavior to the dropdown.
* First, the `button.dropdown-label` is enhanced by the `ToggleButton` widget. Since the dropdown is expanded, the `ToggleButton` is initialized as "pressed". The markup does not change.
* Next comes the most important features: the `dropdown-expanded` class is toggled when the ToggleButton is clicked (i.e., fires a `toggled` event). The script detects the expanded class and initializes its own state accordingly. Markup is not changed.

### "Display" button ###
As mentioned earlier, an event listener is added to the list to detect changes to the selection and enable / disable the button.

If instead the button were never rendered disabled by the server (which degrades gracefully), no javascript would be required at all.

### List search ###
Since the `ul` has the `searchable` class, javascript will add a search input and autocomplete functionality on page load.

If javascript fails, no harm is done (although the user loses some comfort if there are many options).
