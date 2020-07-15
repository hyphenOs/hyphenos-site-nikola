.. title: CRUD Pattern in React - Implementation
.. slug: crud-pattern-in-react-implementation
.. date: 2020-07-15 12:33:18 UTC+05:30
.. tags: ReactJS
.. author: Mayur Borse
.. link:
.. description:
.. type: text
.. status:
.. summary: This is continuation post of our <strong>CRUD Pattern</strong> series. We introduced the pattern with the <em>Books</em> (parent) component in the <a href="/blog/2020/crud-pattern-in-react-introduction">previous post</a>. This post provides the implementation details of <em>BookForm</em> and <em>BookTable</em> (child) components.

# Overview

In the <a href="/blog/2020/crud-pattern-in-react-introduction">previous post</a>, we checked the structure of `Books` component. The `Books` component implements CRUD actions and passes these actions and data (retrieved from API) to its child components `BookTable` and `BookForm`. In this post, We will check the implementation details of `BookTable` and `BookForm` components.

# `BookTable` Component - Implementation

A table view is generally used to provide summary of the data. It may also provide an interface (e.g. button) to trigger _Update_ and _Delete_ actions on the selected data record. The `BookTable` component presents book entities in a summary view in table format. It triggers _Update_ and _Delete_ actions using `showEditForm` and `performDelete` callback functions passed as `props` by the `Books` component (parent). It also passes the data required for the actions (book object, book id) back to the parent component.

React supports Unidirectional/One-Way Data Flow only. It means the data is passed from parent to child as `props` which are immutable arguments. Our actions (update, delete) are defined in the `Books` component (parent), but we need the data (selected book id or object) from `BookTable` component (child) to perform the actions. We can achieve this by explicitly invoking Inverse Data Flow from `BookTable` component to `Books` component.

## Inverse Data Flow

- A callback function (`openForm()` or `delete()`) defined in the parent (`Books`) component is passed as props to the child (`BookTable`) component.
- The function is invoked by the child component for an event. e.g. to trigger update or delete action (function invoke) on button click (event).
- The child component can pass data (book id or book object) as an argument to the callback function. As the callback function is defined in the parent, the data is made available to the parent component, thus achieving _Inverse Data Flow_.
- The passed data can then be used by the parent component to update the internal state which may potentially re-render the page.

`BookTable` component is fairly straightforward and we have covered its usage to demonstrate the *Inverse Data Flow* concept in this post. Its entire implementation is available [here](https://github.com/hyphenOs/library-frontend/blob/master/src/pages/books/components/BookTable.js)

Next, we will look at the `BookForm` component implementation.

# `BookForm` Component - Implementation

A form component creates or updates an entity on the server by performing API actions. In order to do this, it needs to fulfill following requirements:

- Validate form and display validation errors (if any) related to specific field
- Display API response errors (if any) related to specific field on the UI

We can fulfill these requirements by using a `formFieldAttributes` object that defines validation rules for each field. We will check its details in the next section.

## `formFieldAttributes` Object

`formFieldAttributes` object is a collection of _objects_ where each such _object_ defines validation rules as attributes specific to individual form field. These objects wrap field validation logic and are used to map errors. One _key_ advantage of using this object is - If in future a new field needs to be added to the form or a field is to be changed from 'optional' to 'required', one simply has to update this object and all the code that performs validation etc. (see below `formValidator` function) is not required to be touched. This helps a lot in maintaining the code.

```javascript
const formFieldAttributes = {
  title: {          // value of key ("title") is same as this (convention)
    key: "title",   // Should be unique as it is used as field 'id'
    required: true, // Whether an empty field is allowed or not
    customValidator: value =>
      value.length > 100 // Custom validation logic
        ? { helperText: "enter less than 100 characters", error: true } // Object with error
        : { helperText: "", error: false } // Object without error
  },
  ...
}

// Using 'key' as argument to fieldError() and fieldHelperText()
// to map error and helperText
<form id="bookForm">
    <TextField
        id={formFieldAttributes.title.key}
        error={this.fieldError(formFieldAttributes.title.key)}
        helperText={this.fieldHelperText(formFieldAttributes.title.key)}
        ...
    />
...
</form>

```

1. `key`: Used to generate and map error to the respective field
2. `required`: Defines whether an empty field is allowed or not
3. `customValidator()`: Provides custom validation logic for each field. Returns an error object in `{key: { error: true, helperText:"text" }}` format.


`formValidator()` function uses this object to validate the form and generate errors (if any). We will check its functioning in the next section.

## Form Validation (`formValidator()` Function)

`formValidator()` function is the central place for performing all the validations before the API call is made. Validation results are collected in `formErrors` object and errors (if any) present in `formErrors` object are mapped to field. It performs these actions using `formFieldAttributes` object discussed above. The function looks like -

```javascript
const formValidator = (formFieldAttributes, userInput, isEditForm) => {
  let formErrors = {};
  for (let field in formFieldAttributes) {
    let fieldObj = formFieldAttributes[field];
    if (isEditForm && !fieldObj.editable) {
      continue;
    }
    let fieldKey = fieldObj.key;

    let value = userInput[fieldKey];

    let fieldEmpty = !value;

    if (fieldEmpty && fieldObj.required) {
      formErrors[fieldKey] = {
        error: true,
        helperText: "this field should be non-empty"
      };
    } else if (fieldObj.customValidator && !fieldEmpty) {
      let customValidator = fieldObj.customValidator;
      let formError = customValidator(value);
      if (formError.error) formErrors[fieldKey] = formError;
    }
  }
  return { formErrors, errorCount: Object.keys(formErrors).length };
};
```

In fact, we have used the `formValidator` as a common component across several forms in our implementation and works quite well here. It's possible depending upon the complexity of validation logic, this component cannot be made a common component across several forms, in such cases, it should be part of the `Form` component, to keep the functionality well contained.

In the next section, we will see how errors returned from API response are similarly mapped to UI fields.

## API Errors Reporting (`apiErrorsToFormField()` Function)

Collecting field specific errors received from API response is done by `apiErrorsToFormFields()` function in very similar manner to that of `formValidator()`. For every field error, an error object (as mentioned in previous section) is added in `apiErrors` object. Thus, using the same object (`formFieldAttributes`) one can map both client-side and server-side errors to UI fields in a consistent manner.

In the next section, we will see how the errors from `formErrors` and `apiErrors` are mapped to the related field.

## Mapping Errors (`fieldError()` and `FieldHelperText()` Helper Functions)

`formErrors` and `apiErrors` (which are similar objects) are used by these helper functions to display the field specific errors on UI. To identify the field to which the error is mapped, the `key` attribute (from `formFieldAttributes` object) is used. The code that performs this is fairly straight forward and code for `fieldError` helper function is shown below -

```javascript
  fieldError = id => {
    let apiErrors = this.apiErrorsToFormFields();
    let errors = { ...apiErrors, ...this.state.formErrors };
    let field = errors[id];
    return field ? field.error : false;
  };
```
# Conclusion

We checked how to achieve Inverse Data Flow in the `BookTable` component and a self-contained form component that does
validation and error mapping internally, without leaking this functionality outside the component. So to conclude  -

1. Think of Dashboard applications as **CRUD** for different objects.
2. Structure individual component as a Parent Component defining actions and wrapping UI components.
3. UI Components should be self-contained and their logic should not be leaking out (e.g. Validation logic should be within a Form component where it is 'required'.)

Feel free to check the source code here:

- [front-end (React)](https://github.com/hyphenOs/library-frontend)
- [back-end (Django REST)](https://github.com/hyphenOs/library-backend)
