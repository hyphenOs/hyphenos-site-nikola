.. title: CRUD Pattern in React - Implementation
.. slug: crud-pattern-in-react-implementation
.. date: 2020-07-08 12:33:18 UTC+05:30
.. tags: ReactJS
.. category: Mayur Borse
.. link:
.. description:
.. type: text
.. status:
.. summary: This is a continuation post of our <strong>CRUD Pattern</strong> series. We introduced the pattern with the <em>Books</em> (parent) component in the <a href="/blog/crud-pattern-in-react-introduction">previous post</a>. This post provides the implementation details of <em>BookForm</em> and <em>BookTable</em> (child) components.

# Overview

In the <a href="/blog/crud-pattern-in-react-introduction">previous post</a>, we checked the structure of `Books` component that implements CRUD actions and passes these actions and data (retrieved from API) to its child components `BookTable` and `BookForm`. In this post, We will check implementation details of `BookTable` and `BookForm` components.

# BookTable Component - Implementation

A table view is generally used to provide summary of the data. It may also provide an interface (eg. button) to trigger _Update_ and _Delete_ actions on the selected data record. The `BookTable` component presents book entities in a summary view in table format. It triggers _Update_ and _Delete_ actions using `showEditForm` and `performDelete` callback functions, passed as `props` by the `Books` component (parent). It also passes the data required for the actions (book object, book id) back to the parent component.

React supports Unidirectional/One-Way Data Flow only. It means, the data is passed from parent to child as `props` which are immutable arguments. Our actions (update, delete) are defined in the `Books` component (parent), but we need the data (selected book id or object) from `BookTable` component (child) to perform the actions. We can achieve this by explicitly mapping Inverse Data Flow from `BookTable` component to `Books` component.

## Inverse Data Flow

- A callback function (`openForm()` or `delete()`) defined in the parent (`Books`) component is passed as props to the child (`BookTable`) component.
- The function is invoked by the child component for an event .
- eg. to trigger update or delete action (function invoke) on button click (event).
- The child component can pass data (book id or book object) as an argument to the callback function. As the callback function is defined in the parent, the data is made available to the parent for further action.
- As data is passed in the _inverse_ direction (from child to parent), Inverse Data Flow is achieved.
- The passed data can then be used by the parent component to update internal state which may potentially re-render the page.

# BookForm Component - Implementation

A form component creates or updates an entity on the server by performing API actions. In order to do this, it needs to fulfill following requirements:

- Validate form and display validation errors (if any) of specific field
- Display API response errors (if any) of specific field on the UI

We can fulfill these requirements by using a `formFieldAttributes` object that defines validation rules for each field. We will check its details in the next section.

## `formFieldAttributes` object

It defines custom validation rules for every field in `{fieldName: { attr1: value1, ..., attrN: valueN }}` format. The rules can be specified as attributes as following :

1. `key`: Used to generate and map error to the respective field
2. `required`: Defines whether an empty field is allowed or not
3. `customValidator()`: Provides custom validation logic for each field. Returns an error object in `{key: { error: true, helperText:"text" }}` format.

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

`formValidator()` function uses this object to validate form and generate errors (if any). We will check its functioning in the next section.

## Form Validation (`formValidator()` function)

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

Form validation is performed by `formValidator()` function (just before an API call is made). This function checks every field value against the attributes mentioned in `formFieldAttributes` object. For every field error, it adds an error object as `{key(fieldName): { error: true/false, helperText: "text"/"" }}` in `formErrors` object. Error object for every field might also be received from `customValidator()`. Finally, this function returns `formErrors` object. The caller will then map errors to field using this `formErrors` object.

In the next section, we will see how errors returned from API response are similary mapped to UI fields.

## API Errors Reporting (`apiErrorsToFormField()` function)

Collecting field specific errors received from API response is done by `apiErrorsToFormFields()` function in very similar manner to that of `formValidator()`. For every field error, an error object (as mentioned in previous section) is added in `apiErrors` object. An error generating pattern can be clearly observed here.

In the next section, we will see how the errors from `formErrors` and `apiErrors` are mapped to the related field.

## Mapping errors to field (`fieldError()` and `FieldHelperText()`)

The errors from `formErrors` and `apiErrors` are combined and mapped to related field by `fieldError()` and `fieldHelperText()` functions. An `id/key` argument is used to check if the field has an error or helperText. Errors from `apiErrors` are mapped only when there are no errors in `formErrors` object.

# Conclusion

We checked how to achieve Inverse Data Flow in the `BookTable` component and a self contained form component that does
validation and error mapping internally. This was the second post in "CRUD pattern" series. We will check another React post soon.

Feel free to check the source code here:

- [front-end (React)](https://github.com/hyphenOs/library-frontend)
- [back-end (Django REST)](https://github.com/hyphenOs/library-backend)
