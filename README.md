# NextForm

## What is NextForm?

NextForm is an application that allows you to easily create dynamic, ephemeral, UI-based forms in ServiceNow.
This is an exended version of what David Lindgren created. The two are similar but not swappable

Previously, to create a form displayed in the UI of ServiceNow you would either need to create a table, or create a record producer. Sometimes these methods are perfectly fine; other times they can be more complex than needed. For example:

- If you create a table, you get all the overhead and complexity in your application that comes with them, such as ACLs, views, related lists, UI actions, and so on.
- If you create a record producer, you can only use it to create new records. They are not designed to support opening existing records and editing those values.

NextForm allows you to dynamically define a form that will only ever exist in the UI. The values for the form can be pre-populated with data sourced from the ServiceNow instance, and once completed the values can also be persisted into storage on the ServiceNow instance. However, how the form is generated and how it is processed is completely left up to the developer leveraging NextForm.

![NextForm Overview](images/nextform-overview.png)

NextForm's power comes from its Controller; it contains all the front-end logic to drive the experience. Using its [Preset](https://docs.servicenow.com/bundle/tokyo-application-development/page/administer/ui-builder/concept/presets.html) you can easily attach it to and configure a Form component. Simply add the Form component to your UI Builder page and you'll be offered to connect it to a NextForm with the prompt shown below:

![NextForm "Preset" prompt](images/form-component-preset.png)

In the **NextForm Sys ID** field, insert the **Sys ID** of a record in the **NextForms** [`x_snc_nf_nextform`] table and click **Use**. A NextForm has both a **Generator** and a **Processor**.

## Generators

Generators create the form that will be displayed by NextForm. They must return an `x_snc_nf.NextForm` object.

The NextForm object has helper functions to define your form. Each function returns a reference to itself, so you can use method chaining to define forms in an expressive manner.

When the form is loaded via the REST API, an optional `input` object may be passed to the generator script as the scoped variable `input` (for example, context from the host page).

### The NextForm Generator API

#### NextForm

##### `new NextForm()`

Creates a `NextForm`. There are no parameters required.

##### `NextForm.addScreen()`

Add a screen to the NextForm. There are no parameters required.

##### `NextForm.addRow()`

Adds a vertical row to the most recently created screen. Columns are placed inside rows. There are no parameters required.

##### `NextForm.addColumn()`

Adds a horizontal column to the most recently created row. Fields are placed inside columns. There are no parameters required.

##### `NextForm.addField(x_snc_nf.NextField)`

Adds a field to the most recently created column. Accepts a `NextField` as its only parameter.

#### NextField

##### `new NextField(type, label, name, options)`

Creates a `NextField`. Accepts three mandatory parameters and one optional object:

###### type

Mandatory. The field type as a string. See [Supported field types](#supported-field-types) below. Unknown types fall back to `string`.

###### label

Mandatory. The label for the field.

###### name

Mandatory. The internal name of the field. This must be unique across the entire NextForm.

###### options

Optional. An object with field properties. Common options:

| Option | Applies to | Description |
|--------|------------|-------------|
| `value` | Data fields | Initial stored value |
| `displayValue` | Data fields | Initial display value |
| `mandatory` | Data fields | Whether the field is required (also settable via `nfValidation`) |
| `readonly` | Data fields | Whether the field is read-only (also settable via `nfValidation`) |
| `maxLength` | `string`, `integer`, `decimal`, `html` | Maximum length |
| `fieldHint` | Data fields | Hint text shown on the field |
| `fieldLayout` / `layout` | Data fields | Layout hint (for example `{ layout: 'vertical' }`) |
| `fieldRegex` | Data fields | Array of regex rules for the Form component (see below) |
| `choices` | `choice` | Array of `{ displayValue, value }` entries |
| `reference` | `reference`, `list` | Reference table name |
| `referenceQualifier` | `reference` | Encoded query or qualifier for the reference picker |
| `useReferenceQualifier` | `reference` | Qualifier mode (for example `'simple'`) |
| `currencyCodes` | `currency` | Allowed currency codes |
| `nfValidation` | Data fields | Client-side validation rules (see [nfValidation](#nfvalidation) below) |

When the same property appears in both `options` and `options.nfValidation`, **`nfValidation` wins** for `maxLength`, `minLength`, `readonly`, and `mandatory`.

###### fieldRegex

Optional array of pattern rules passed through to the Form component:

```javascript
fieldRegex: [{
    pattern: '^[A-Za-z]*$',
    placeholder: '',
    guidanceText: 'Value does not match required pattern',
    type: 'suggestion'
}]
```

You can also supply `nfValidation.regex` as a string; NextField converts it into a `fieldRegex` entry when one is not already set.

### Supported field types

| Type | Description |
|------|-------------|
| `string` | Single-line text (default for unknown types) |
| `integer` | Whole number |
| `decimal` | Decimal number |
| `boolean` | True/false |
| `choice` | Dropdown from a supplied `choices` list |
| `date` | Date (`glide_date`) |
| `dateTime` | Date and time (`glide_date_time`) |
| `time` | Time of day (`glide_time`) |
| `currency` | Currency amount |
| `html` | HTML content (supports `maxLength`, default 8000) |
| `url` | URL |
| `reference` | Reference to a record (supports `reference`, `referenceQualifier`) |
| `list` | List of references (`glide_list`; supports `reference`) |
| `label` | Read-only text display; **not submitted** (`isDataField: false`) |
| `annotation` | Informational annotation; **not submitted** (`isDataField: false`) |

### nfValidation

Pass client-side validation rules in `options.nfValidation`. NextField normalizes these onto the field JSON sent to the UI. The NextForm controller runs validation on load, on field change, and before processing; invalid fields block **Process** (`canProcess` stays false until all fields pass).

Supported properties:

| Property | Description |
|----------|-------------|
| `mandatory` | When `true`, the field must have a value |
| `minLength` | Minimum string length |
| `maxLength` | Maximum string length |
| `allowedValues` | Array of permitted string values (for non-choice fields) |
| `type` | Override the validation type (for example validate a string field as `integer`) |
| `regex` | Pattern string; converted to `fieldRegex` on the field |
| `readOnly` / `readonly` | When `true`, the field is read-only and skipped by validation |

When `readOnly` is `true`, other validation properties are ignored.

Validation behavior by type (when `nfValidation.type` is set, or inferred from the field type):

- **string** (default): mandatory, `allowedValues`, `minLength`, `maxLength`
- **integer**: must match `/^-?\d+$/`
- **decimal**: must match a decimal number pattern
- **boolean**: must be one of `true`, `false`, `1`, `0`, `y`, `n` (case-insensitive)
- **date** / **glide_date** / **glide_date_time**: must parse as a valid date
- **choice**: value must appear in the field's `choices` list

Regex matching on the client is handled by the Form component's built-in `fieldRegex` evaluation, not by a separate `nfValidation.regex` check.

### Choice lists

Choice fields require a `choices` array on the field options:

```javascript
choices: [{
    displayValue: 'Under 18',
    value: 'under_18'
}, {
    displayValue: '18 and Over',
    value: 'over_18'
}],
value: 'under_18',
displayValue: 'Under 18'
```

Each entry needs `displayValue` (shown in the UI) and `value` (stored/submitted). Set `value` and `displayValue` on the field options to pre-select an option.

Processors can replace the choice list at runtime by returning `choices` in a field's `newValues` patch (see [Processors](#processors)).

### Messages

Field messages are an array of `{ message, type }` objects on the field. The controller and Form component use them to show inline feedback.

**Client validation** sets messages automatically when a field fails `nfValidation` checks. For example:

```javascript
[{ message: 'Field is mandatory', type: 'error' }]
```

Common client validation messages include:

- `Field is mandatory`
- `Must be at least N characters long`
- `Exceeds maximum length: N`
- `Value not permitted`
- `Value must be an integer` / `Value must be a number` / `Value must be a boolean` / `Value must be a valid date`
- `No valid choices are configured for this field` (missing or empty `choices`)

When validation passes, `messages` is cleared from the field.

**Processor patches** can also set `messages` on fields via `newValues` (for example server-side validation errors after submit). The same `{ message, type }` shape applies; `type: 'error'` is typical.

### Example Generator Script

This example matches the default Generator script template and demonstrates several field types, `nfValidation`, `fieldRegex`, and choice lists:

```javascript
(function generateNextFormObject() {

    return new x_snc_nf.NextForm()
        .addScreen()
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField(
            'string',
            'Comment',
            'comment', {
                value: '',
                displayValue: '',
                maxLength: 1000,
                fieldRegex: [{
                    pattern: '^[A-Za-z]*$',
                    placeholder: '',
                    guidanceText: 'Value does not match required pattern',
                    type: 'suggestion'
                }],
                nfValidation: {
                    readonly: false,
                    maxLength: 40,
                    minLength: 10,
                    mandatory: true,
                    allowedValues: ['value 1', 'value 2']
                }
            }
        ))
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('integer', 'Quantity', 'quantity'))
        .addField(new x_snc_nf.NextField('choice', 'Age group', 'age', {
            choices: [{
                displayValue: 'Under 18',
                value: 'under_18'
            }, {
                displayValue: '18 and Over',
                value: 'over_18'
            }],
            value: 'under_18',
            displayValue: 'Under 18'
        }))
        .addColumn()
        .addField(new x_snc_nf.NextField('dateTime', 'When did you last sleep?', 'sleep'))
        .addField(new x_snc_nf.NextField('boolean', 'Receive marketing?', 'marketing_allowed'))
        .addScreen()
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('annotation', 'Note', 'note_hint', {
            displayValue: 'All fields marked mandatory must be completed before submit.'
        }));

})();
```

## Processors

Processors handle the user response for a form as required by the form developer.

The `nfData` object is keyed by field name. Each entry submitted from the client is typically:

```javascript
{
    name: 'comment',
    value: 'some value',
    displayValue: 'Some Value'
}
```

Only data fields are included (`isDataField !== false`). When the host page sets **Process edited only**, only fields whose `value` differs from `originalValue` are sent.

Processors receive `metadata` as a second argument when the host page supplies it via the NextForm controller property.

### Processor return contract

Return an object with this shape:

```javascript
{
    success: true,      // or false
    message: '',        // optional human-readable message (form-level)
    newValues: {        // field name -> patch object
        comment: { value: 'Updated text' },
        age: {
            value: 'over_18',
            displayValue: '18 and Over',
            mandatory: true,
            readonly: false,
            messages: [{ message: 'Please confirm your age group', type: 'error' }],
            choices: [{ displayValue: 'Under 18', value: 'under_18' }]
        }
    }
}
```

Each entry in `newValues` may only include **`value`**, **`displayValue`**, **`mandatory`**, **`readonly`**, **`messages`**, and **`choices`**. `NFAPIHelper` strips any other properties before the REST response is returned.

Return `newValues` when the processor transforms field state. If `newValues` is **omitted** or an **empty object** (`{}`), `NFAPIHelper` fills it from the `nfData` submitted to the API (normalizing to the allowed patch keys). Build `newValues` manually when you need a subset of fields or non-default patches.

After a successful process, the controller merges each allowed patch property onto matching fields (including falsy `value`s such as `0`, `false`, and `""`) and updates `originalValue` when `value` is present in the patch.

When `success` is `false`, the controller records the result in `lastProcessResult` and does not apply field patches.

### Example Processor Script

This example matches the default Processor script template:

```javascript
(function process(nfData, metadata) {

    gs.info('NextForm Processed! ' + JSON.stringify(nfData));

    // Persist, validate, or transform — then return patches or echo nfData:
    return {
        success: true,
        message: '',
        newValues: nfData
    };

})(nfData, metadata);
```

Example returning server-side validation messages:

```javascript
(function process(nfData, metadata) {

    if (gs.nil(nfData.comment?.value)) {
        return {
            success: false,
            message: 'Comment is required',
            newValues: {
                comment: {
                    messages: [{ message: 'Comment is required', type: 'error' }]
                }
            }
        };
    }

    // ... persist data ...

    return {
        success: true,
        message: 'Saved successfully'
    };

})(nfData, metadata);
```

## NextForm Controller

The NextForm Controller is automatically added to the UI Builder page when the NextForm Preset is applied to the Form component. You can find it in the **Data Resources** panel.

### Output Properties

The NextForm Controller manages the state of the form, and outputs properties to allow UI Builder page developers to build a UI that controls the form.

| Property            | Type    | Description                                                                                                                   |
|---------------------|---------|-------------------------------------------------------------------------------------------------------------------------------|
| nextFormData        | JSON    | Layout data from the Generator (e.g. `screens`); the `fields` map is not duplicated here—it is exposed only via the `fields` output. |
| fields              | JSON    | All the fields of the NextForm in their JSON representation. Fed into the fields property of the Form component.              |
| currentSections     | JSON    | The current screen of the NextForm in JSON representation. Fed into the sections property of the Form component.              |
| currentScreenNumber | Integer | The 1-indexed (starting at 1) number of the current screen. Used for UI display of the current screen.                        |
| currentScreenIndex  | Integer | The 0-indexed (starting at 0) index of the current screen. Used for programmatic access to the current screen.                |
| screenCount         | Integer | The total number of screens.                                                                                                  |
| canGoFirst          | Boolean | Whether it is possible to change to the first screen (e.g. true if on second screen, false if already on the first screen)     |
| canGoLast           | Boolean | Whether it is possible to change to the last screen (e.g. true if on second-last screen, false if already on the last screen) |
| canGoNext           | Boolean | Whether it's possible to change to the next screen. True if not already on the last screen.                                   |
| canGoPrevious       | Boolean | Whether it's possible to change to the previous screen. True if not already on the first screen.                              |
| canGoIndex          | Boolean | Whether it's possible to change screens. True if there is more than 1 screen.                                                 |
| isLoading           | Boolean | Whether NextForm is doing any kind of loading operation (e.g. initial load, processing)                                       |
| canReset            | Boolean | Whether the NextForm can be reset to its default values.                                                                      |
| canProcess          | Boolean | Whether the NextForm can be processed. True on the last screen when all fields pass validation and the form has not already been processed. |
| isLoaded            | Boolean | Whether the initial load of the NextForm has completed.                                                                       |
| isProcessed         | Boolean | Whether the form has already been processed.                                                                                  |
| lastProcessResult   | JSON    | Last API process result: `{ success, message, newValues }`. Updated after each process completes or fails.                    |

### Handled Events

| Name                           | Description                                                                      | Parameters                                                                                                                                |
|--------------------------------|----------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| [Screen] Change Current Screen | Navigate to a specified screen number (1-indexed)                                | Screen number [screen_number]: the 1-indexed number of the screen to navigate to.                                                         |
| [NextForm] Process             | Submits the field values of the NextForm to the Processing script on the server. |                                                                                                                                           |
| [Field] Set Value              | Sets a specified field's value and display value.                                | Field [field]: The name of the field to set the value of Value [value]: The new value Display value [displayValue]: The new display value |
| [NextForm] Reload              | Reset the NextForm's fields to have all their default values.                    |                                                                                                                                           |
| [Screen] Go To First Screen    | Navigate to the first screen (if not already on it)                              |                                                                                                                                           |
| [Screen] Go To Last Screen     | Navigate to the last screen (if not already on it)                               |                                                                                                                                           |
| [Screen] Go To Next Screen     | Navigate to the next screen (if possible)                                        |                                                                                                                                           |
| [Screen] Go To Previous Screen | Navigate to the previous screen (if possible)                                    |                                                                                                                                           |

### Dispatched Events

| Name                            | Description                                      | Payload |
|---------------------------------|--------------------------------------------------|---------|
| [NextForm] Processing completed | When the processing of a NextForm has completed. | None; read **`lastProcessResult`** (`{ success, message, newValues }`) |
| [NextForm] Field value changed  | When the value of a field has changed.           |         |
| [NextForm] Processing started   | When the processing of a NextForm has started.   |         |
| [NextForm] Loading completed    | Once the initial load of the form has completed  |         |
