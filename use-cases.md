# Concrete Use cases

[toc]

The use cases on a high level are outlined in the [explainer](README.md).
This document is to list very specific, concrete use cases.

## Native functionality that could have been implemented via CA

- The `list` attribute of `<datalist>` for autocomplete
- The `title` attribute for tooltips
- Popover API
- Invokers

## Global attributes

- Add icon (`prefix="icon-name"` /`suffix="icon-name"` )
- Skeleton (e.g. `<p loading-placeholder="3 sentences">` )
- Tooltip (`tooltip="..."`)
- Data binding

```html
<input id="slider" type=range sync-with="slider-value">
<input id="slider-value" type=number sync-with="slider">
```

- Highlight matches within (e.g. `highlight="foobar"` )
- `removable`  (adds X button that removes the element and fires a suitable event)
- Conditional rendering (`show-if`  , `hide-if`  , `remove-if` etc)
- Emulate events for test framework (e.g. `click-after="0.5"` ) 
- LIterally all [htmx](https://htmx.org/docs/) attributes

## Form control attributes


- Persisting forms (e.g. in local storage)
- Custom form validation
- `required-trimmed`: Like `required`, but whitespace-only values don't count
- Show/hide password
- Checkboxes
	- Designate a checkbox as an aggregate checkbox for checkboxes with a given `name`  
	  - Or a `<progress>`  element
	- `indeterminate`  attribute
- Text inputs
	- Specific format, e.g. currency, measurement, 5-digit code, etc
- `<button>`  
	- `<button href>`  
	- Disable button if field is invalid (or form)

## Tables
- `<table sortable>` 
- `<td value>` to be used for filtering and sorting

## Code

- `<pre src>`  for loading and highlighting a file
- `<pre editable>` for code editors (or any other element)
- `<pre normalize-whitespace>`  

## Media elements

- HTMLMediaElement (`<audio>`  and `<video>` )
  - Custom toolbar
  - `start-at="0:05"`  
  - `lazy-src`  
- Images
  - Click to play (with separate `src`  for the static frame)

## Other

- `format` attribute on `<time>` , `<data>` for presenting data with custom formats