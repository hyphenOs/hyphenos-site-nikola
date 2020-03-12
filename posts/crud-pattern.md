.. title: CRUD Pattern
.. slug: crud-pattern
.. date: 2020-03-12 17:02:28 UTC+05:30
.. tags:
.. category:
.. link:
.. description:
.. status: draft
.. type: text

# CRUD Pattern

React.js based frontend powered by REST API can be a typical architecture in Dashboard like applications. In my last project, I was working on such a Dashboard application where front-end was written in React.js and the API was powered by Django REST framework. The application consisted of multiple pages with a similar structure. Each page would look like a table of records which can be **Created**, **Retrieved**, **Updated** or **Deleted**, the common **CRUD** pattern. When I joined the project, the prototype was already in place and the code had become complex to understand. After several sessions of work on different parts of the codebase, it was apparent that each page had similar functionality(CRUD actions). There seemed a possibility of abstraction of this functionality into a common component that would provide a neatly structured code that is easier to understand and maintain.

While 'thinking' about the structure of the abstract component, a conclusion was formed that each such component required -

- A Table UI that will render a short view of the retrieved object. it will have an interface to Update or Delete the object.
- A Form UI that will show details of the object in an editable view. It will provide UI to either Create a new object or Update the existing object.

In this blog post, we will go through the implementation of the CRUD pattern using a simple Library App. We'll look in detail how the functionality can be split into multiple components. We'll check the Form component that handles all of its required functionality (onChangeHandler, formValidator etc.) by itself. While the eventual implementation has undergone several iterations, we will also check a couple of false starts which we started with and refactored the components properly later on.

We'll structure our pattern in the following way -

1. A Base component (a wrapper) that wraps a Form component, a Table component and a set of actions for API calls. It will pass action handler functions(calls API action and also handles API response) and records(data) as props to either Form or Table component. It will also pass API error received from the backend to the Form component.
2. A Table component wrapped by Base component. It will receive records (table data) from the Base component as props. It will pass the selected record as an argument to action handler function passed by the Base component for further action(Update, Delete).
3. A Form component wrapped by Base component. If the existing record is to be updated, the selected record passed by Table to Base component via action handler function is received as props. A common `onChangeHandler` handles changes for all input fields. It will handle all validation specific functionality by itself with the help of `formValidator` function. `formValidator` uses `apiToFormFieldIDs` object that contains field validation attributes. The backend validation errors if any, will be passed by the Base component to the Form component for error handling using `apiErrorsToFormFields`. Either `formValidator` or `apiErrorsToFormFields` set field-specific helpertext and error attributes using `fieldHelperText` and `fieldError` functions respectively. We will look at one interesting way of achieving it.

> Base Component Structure

```javascript
class Base extends React.Component {
  componentDidMount() {
    retrieveAPIAction();
  }
  add(){
    addAPIAction()
  }
  edit(){
    editAPIAction()
  }
  delete(){
    deleteAPIAction()
  }
  render(){
    <Form />
    <Table />
  }
}
```

Let's look at the Library App Structure that has _Books_ and _Members_ sections with _Book_ and _Member_ objects respectively.

1. Dashboard: Home page of the library app that shows the total numbers of books and members.
2. Book: It has `title, author, isbn and year` fields.
3. Member: It has `name, phone, email, address, city, state, zip` fields.

We'll be using Book Model for a demonstration from here onwards.

```javascript
const initialState = {
  open: false,
  bookData: {},
  error: null
};

class Books extends React.Component {
  state = { ...initialState };

  componentDidMount() {
    this.props.getBooksAPIAction();
  }

  addBook = bookData => {
    addBookAPIAction(bookData);
  };

  editBook = (bookId, bookData) => {
    editBookAPIAction(bookId, bookData);
  };

  deleteBook = bookId => {
    deleteBookAPIAction(bookId);
  };

  render() {
    return (
      <div>
        <BookForm
          formData={this.state.bookData}
          addBook={this.addBook}
          editBook={this.editBook}
          error={this.state.error}
        />

        <BookTable
          books={this.props.books}
          showEditForm={this.openForm}
          performDelete={this.deleteBook}
        />
      </div>
    );
  }
}

const mapStateToProps = ({ books }) => {
  return { books };
};

const mapDispatchToProps = { getBooksAPIAction };

export default connect(mapStateToProps, mapDispatchToProps)(Books);
```

Books section will have the following components -

- Book List View :
  - A Table component that Shows all the records in table format. triggers Update and Delete.
- Book Details View:
  - A Form component to input, validate and submit mentioned Book fields.
  - The same form will be used to edit, validate and submit existing book record.
  - API error passed by Base component will be used to set helperText and error of the form field.

BookTable component is kind of self-explanatory so we'll not cover it in detail.

Let's look at the BookForm component in more detail.
Typically a Form component has following actions -

- onChangeHandler - Handles changes for every field and sets it in a formData and changedFields object for Create and Update actions respectively.
- Field-based validation - Whether certain fields are numeric or should follow certain text structure(eg. email).
- API Validation results - This could be errors reported by API that map to a field.

Let's look at their implementation -

> **onChangeHandler**

Initially, while working on a form with several fields, I used to create an `onChangeHandler` for every field, like onChangeTitle, onChangeAuthor,onChangeISBN and so on. It was necessary to create a single common
`onChangeHandler` to handle change for every field which will reduce the duplicate code and will be easier to maintain.
So, we set an "id" prop on every field (eg. id="title', id="isbn") and then, we recieve {id, value} props from the event using destructuring(ES6). Since ES6 has abled the support for dynamic key(variable as Object Key), we use variable id as key (eg. [id]:value => "title":value) and value as value in
formData(Create) and changedFields(Update) objects.

```javascript
onChangeHandler = e => {
  const { id, value } = e.target;
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
```

> **apiToFormFieldIDs** object

- It defines attributes used for field validation.
- The key (or object attribute) is API field name and value is also an object whose keys are
  - `key` - ID of the form input field used to attach helperText or error to the field
  - `required` - Validates whether this field is required or not
  - `editable` - Validates whether this field is editable or not after creation
  - `customValidator` - A function for custom-defined validations for the user input.
    - eg. validate if input for 'year' field is numeric only and within valid range
    - It will validate the value with defined conditions and will return a formError object

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
  }
};
```

Let's look at form submission in little more details -

- `formValidator` function is used for form field validation
- When a validator for a field fails No API call is made
- API error is mapped to field helperText using field ID.

> **Form Validator**

- Form Validator function receives `apiToFormFieldIDs` object and userInput object(set by `onChangeHandler`).
- It is called by onSubmit function that is invoked when user submits the form.
- attributes from `apiToFormFieldIDs` are used for validations
  - `editable` will bypass the field validation on edit form, if false
  - `required` will validate whether input is empty if true
  - `customValidator` if defined, will validate input using custom validations and return formError object if input is invalid
- The object returned by `customValidator` is then set in formErrors for the field
- If formError object us not empty then its errors are set to respective field using `fieldError` and `fieldHelperText` functions and no API call is made.
- error object will be ({error:true, helperText:"invalid input"} for invalid value and {error:false, helperText: {}for valid value)
- The form is submitted if the formErrors object is empty

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

> **apiErrorsToFormFields** function

- The API call is made only after successful validation (empty formErrors object)
- If the API response is an error, that error is passed to `apiErrorsToFormFields` function in Form component
- `apiErrorsToFormFields` checks for an errorData object and sets it into apiErrors object
- apiErrors object is similar to formErrors object.
- The apiErrors object is passed to `fieldError` and `fieldHelperText` functions, just like `formError` object
- `fieldError` and `fieldHelperText` functions will set error and helperText on the field respectively.

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

> **fieldError** and **fieldHelperText** functions

- These functions receive a field id as an argument ("title", "isbn") to match with the key in formErrors, apiErrors objects.
- If field id matches the key, then the error and helperText is set for that field.
- If the error attribute is true, the label will be displayed in an error state.
- The helperText attribute will show the error text for the field.
- Default values are false and "" for `fieldError` and `fieldHelperText` respectively.

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

Points to remember -

1. The BaseComponent should implement an action handler function for each API action and pass it to Form/Table as props. When the function is triggered by Form/Table it shall call the API Action.
2. A common `onChangeHandler` can be used to set value for many number of input fields
3. `apiToFormFieldIDs` object can be used to define field attributes and custom validations
4. API error is passed by BaseComponent to Form component which uses this error to set field helperText and error.
5. API error handling, field onChangeHandler and Form validations are handled by Form itself.

This was the CRUD Pattern applicable in many cases. We will carry our journey forward in the next blog post.

source code:

frontend (React): [https://github.com/hyphenOs/library-frontend](https://github.com/hyphenOs/library-frontend)

backend (Django REST): [https://github.com/hyphenOs/library-backend](https://github.com/hyphenOs/library-backend)
