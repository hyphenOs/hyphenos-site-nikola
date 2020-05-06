.. title: CRUD Pattern in React - Implementation
.. slug: crud-pattern-in-react-implementation
.. date: 2020-05-07 12:57:04 UTC+05:30
.. tags: CRUD, Design Pattern, Form Validation
.. category: React
.. link:
.. description:
.. type: text
.. status: draft
.. type: text
.. summary: In this post, we will see the implementation details of <em>BookTable</em> and <em>BookForm</em> components. We will explore how to handle field change, field validation, form errors, API errors in <em>BookForm</em> component. We will be using an apiToFormFieldIDs object that holds validation properties for every field. We will also explore how to achieve <strong>Inverse Data Flow</strong> in <em>BookTable</em> by passing data back to parent component Books.


**Recap:** `Books` is a Base component that handles **CRUD** actions and also renders `BookTable` and `BookForm` components. It passes actions and data to the child components. We have introduced it in the previous post that can be found [here](/blog/crud-pattern-in-react-intro/).

# 1. BookTable

In general, A Table view is used to provide a summary of the data. It can also have an interface (eg. button) to trigger *Update* or *Delete* action on selected data record. The `BookTable` uses `Table` component to present book entities in a summary view. It triggers *Update* or *Delete* action using `showEditForm` and `performDelete` callback functions, passed as `props` by the `Books` component (parent). It also passes the data required for the actions (book object, book id) back to the parent component.

React supports **Unidirectional/One-Way Data Flow** only. It means, the data is passed from parent to child as `props` which are immutable arguments and not the other way around. Our actions (update, delete) are defined in the `Books` component (parent), but we need the data (selected book id or object) from `BookTable` component (child) to perform the actions. It is not possible to pass data back in *Unidirectional Data Flow*. So how do we *pass back* data from child to parent component? We are able to achieve it by having an **Inverse Data Flow**.

> ## Inverse Data Flow

- A callback function defined in parent component is passed as props to the child component.
- The function is invoked by the child component for an event.
- eg. to trigger update or delete action (function invoke) on button click (event).
- The child component can pass data as an argument to the callback function. As the callback function is defined in parent, the data is made available to the parent for further action.
- As data is passed in the *Inverse* direction (from child to parent), it is called an **Inverse Data Flow**.
- The passed data can then be used by parent component to call an API action or update the state and re-render the page or any other purpose.

# 2. BookForm

In general, a form view is used to get user input. Ideally, a form should handle all the internal actions like field value change, field value validation, set field-specific errors and API errors received in the response by itself. The `BookForm` component handles all the mentioned actions by itself. API actions to create and update book record are passed by parent `Books` component. We are using an object called `apiToFormFieldIDs` to define validatioin properties for form fields. We will be implementing following functions to perform mentioned actions:


- `onChangeHandler()`: Common function to handle value change for every field
- `apiToFormFieldsIDs`: Object used by `formValidator()` function to validate field values against. It holds field validation properties (required, customValidator() etc.)
- `formValidator()`: Validates field values against properties mentioned in `apiToFormFieldIDs`
- `formErrors`, `APIErrors`: Hold errors for the fields. They use field `id` as key and an object with properties `error` and `helperText`  as value
- `apiErrorsToFormFields()`: Generates `apiErrors` object used to map API response errors to respective field by `fieldError()` and `fieldHelperText()`
- `fieldError()`, `fieldHelperText()`: Set `error`, `helperText` prop of field if received field `id` is present in `formError` or `APIError` object


 Let's check implementation details of mentioned actions

> ## onChangeHandler()

Initially, We had a book form with three input fields `title`, `author` and `isbn`. We created a handleChange() function for each field to handle value change like handleChangeTitle(), handleChangeAuthor() and so on. We decided to add one more field for `year` later on, that caused us to add another function handleChangeYear() to handle `year` value change.

For each additional field in future, we will have to implement additional handleChange function. So for n number of input fields, we will end up implementing n number of handleChange functions  ( `O(n)` code complexity ). It will be a poor implementation decision, if all the handleChange functions are doing exactly same thing (set field value in state in this case). What if we can achieve same functionality with a single common handler function ( `O(1)` code complexity ) ?

Since ES6, support for Computed Property Name or Dynamic Object key has been added. Using this feature, we can implement a single `onChangeHandler()` function to handle field value change for any number of fields. We can use the `id` property of the input field as *key* and field value as *value* in formData (Create) and changedFields (Update) objects of state (eg. [id]: value => ['title']: value).

`onChangeHandler()` function follows **Open-Closed (Open for extension, Closed for modification)** principle from **SOLID** principles as no modifications will be required in `onChangeHandler()` for newly added field.

```javascript

onChangeHandler = e => {
  const { id, value } = e .target;
  this.setState({
    ...this.state,
    formData: {
      ...this.state.formData,
      [id]: value
    },
    changedFields: {
      ...this.state.changedFields,
      [id]: value
    }
  });
};


<TextField id="title" onChange={this.onChangeHandler}/>
<TextField id="author" onChange={this.onChangeHandler}/>
<TextField id="isbn" onChange={this.onChangeHandler}/>

// Added later on
<TextField id="year" onChange={this.onChangeHandler}/>

```

> ## apiToFormFieldIDs

It uses API field name as key and an object holding field validation properties as value. These properties are used by `formValidator()` to validate a field. We define field properties `key`, `required`, `editable`, `customValidator()` in this object.

  - `key`: field `id` used to map error or helperText from `formErrors` and `APIErrors` to the field.
  - `required`: Whether the field is required or not. if false, `formValidator()` will not generate error for empty field.
  - `editable` -  Whether this field is editable or not after creation.
  - `customValidator()` - A function to provide field specific validation logic. (like email field value has valid email format or phoneNumber field contains numbers only). It returns error object containing `error` and `helperText` properties



```javascript

const apiToFormFieldIDs = {
  title: {
    key: "title",
    editable: true,
    required: true,
    customValidator: value =>
      value.length > 100
        ? { helperText: "enter less than 100 characters", error: true }
        : { helperText: "", error: false }
  },
  author: { ... },
  ...
}

```

> ## `formValidator()`

It is invoked by onSubmit function when user submits the form. It receives `formData` object containing objects (each having field `id`, `value` as key, value) as a parameter along with `apiToFormFieldIDs` object that contains validation properties for every field. It validates every field against validation properties and generates an error object with `error` (`Boolean`) and `helperText` (`String`) properties. It also receives an error object from `customValidator()`. The error object is then added to `formErrors` object. `formErrors` object is returned along with `errorCount` after all fields are validated.

API call is made only if `formErrors` is empty (errorCount === 0) else, errors from `formErrors` are mapped to respective field by `fieldError()` and `fieldHelperText()` functions

```javascript

const formValidator = (apiToFormFieldIDs, userInput, isEditForm) => {
  let formErrors = {};
  for (let field in apiToFormFieldIDs) {
    if (isEditForm && !apiToFormFieldIDs[field].editable) {
      continue;
    }
    let value = userInput[field];

    let fieldEmpty = !value;
    if (fieldEmpty && apiToFormFieldIDs[field].required) {
      formErrors[field] = {
        error: true,
        helperText: "this field should be non-empty"
      };
    } else if (apiToFormFieldIDs[field].customValidator && !fieldEmpty) {
      let customValidator = apiToFormFieldIDs[field].customValidator;
      let formError = customValidator(value);
      if (formError.error) formErrors[field] = formError;
    }
  }

  return { formErrors, errorCount: Object.keys(formErrors).length };
};

export default formValidator;


```

> ## `formErrors` and `APIErrors`

These objects are generated by `formValidator()` and `apiErrorsToFormFields()` functions respectively. They contain objects with field `id` as key and another object with `error` and `helperText` properties as value. `fieldError()` and `fieldHelperText()` functionsthen use the field `id` to map the `error` and `helperText` properties to respective field.

> ## `apiErrorsToFormFields()`

Once the form is validated and all errors are resolved by the user, we submit the form. We are not done  yet though. There might be errors on server side also. Just like formErrors, server side errors should also be mapped to the respective field. We use `apiToErrorsToFormFields()` for the exact thing. We receive API response error as props from `Books` component. It is then set in state using `getDerivedStateFromProps()` and used to generate `apiErrors` object.

```javascript

apiErrorsToFormFields = () => {
  const { error } = this.state;
  let apiErrors = {};
  if (error && error.response && error.response.data) {
    let errorData = error.response.data;
    for (let apiKey in errorData) {
      apiErrors[apiKey] = {
        helperText: errorData[apiKey].join(""),
        error: true
      };
    }
  }
  return apiErrors;
};

```

> ## `fieldError()` and `fieldHelperText()`

These functions receive field `id` as parameter. apiErrors are received from `apiErrorsToFormFields()` a new local object named *errors* is created by copying the values of `formErrors` and `apiErrors` using *spread* operator. If the field `id` is present in *errors* object, `error` and `helperText` is set and returned for the field. default values of `error` and `helperText` are `false` and "" respectively.

```javascript

fieldError = id => {
  let apiErrors = this.apiErrorsToFormFields();
  let errors = { ...apiErrors, ...this.state.formErrors };
  let field = errors[id];
  return field ? field.error : false;
};

fieldHelperText = id => {
  let apiErrors = this.apiErrorsToFormFields();
  let errors = { ...apiErrors, ...this.state.formErrors };
  let field = errors[id];
  return field ? field.helperText : "";
};

```

This post explored how to achieve ***Inverse Data Flow*** using the `BookTable` component and we also checked different functions, objects used by `BookForm` component to handle field change, field validation, form errors and API errors by itself. We will explore another React pattern in the next post. Till then Good Bye!


Check out the source code for all the implementation details.

- [front-end (React)](https://github.com/hyphenOs/library-frontend)
- [back-end (Django REST)](https://github.com/hyphenOs/library-backend)

Feel free to create an issue in mentioned repos if you have any queries, questions or suggestions as comments are disabled for the post for now.
