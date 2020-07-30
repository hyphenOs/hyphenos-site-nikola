.. title: CRUD Pattern in React - Introduction
.. slug: crud-pattern-in-react-introduction
.. date: 2020-07-01 07:02:28 UTC+05:30
.. tags: ReactJS
.. author: Mayur Borse
.. link:
.. description:
.. status:
.. type: text
.. summary: <strong>React.js</strong> based front-end powered by <strong>REST API</strong> can be a typical architecture in a Dashboard like applications. A Dashboard application usually consists of multiple pages with a similar structure where each page has a collection of objects and an interface to create or update an object using <em>CRUD</em> actions like <em>Create</em>, <em>Retrieve</em>, <em>Update</em> or <em>Delete</em>. In this post, we'll explore the use of <strong>CRUD Pattern</strong> in a Dashboard application.

# Background

I worked on a Dashboard application in my last project where front-end was written in `React.js` and the API was powered by `Django REST Framework (DRF)`. When I joined the project, some initial functionality was already in place. While working on fixing some issues or adding new features to the code, it was observed that similar-looking code was repeated at different places without proper abstraction. Thus reasoning about the code was becoming cumbersome. It was quite apparent that each page utilized similar functionality (essentially *CRUD* actions). It made sense to think of this functionality as a pattern, which can be reused across multiple components (pages). We refer it as a *"CRUD Pattern in React"*. Subsequently, abstracting out the functionality as a pattern substantially improved the readability and maintainability of the code.

# Overview

Typically, in a Dashboard like application, we need a functionality, that consists of -

1. Summary view of a collection of objects
2. Detailed view of an individual object
3. Ability to create new/update existing objects

Detailed view itself can be used for updating an existing object or creating a new object, where each such object represented by an entity served by REST API.

We can implement above functionality in `React.js` as -

1. Table UI component that shows collection of objects and triggers actions on selected object(s). Objects and actions are passed by parent component as props.
2. Form UI component that creates a new object or updates an existing object. It implements and handles functionality for input validation, field value change, error and API error handling by itself.

Both the above components will be wrapped in a `Base` component. The `Base` component will pass CRUD actions and objects (retrieved data) as `props` to these child components. Broadly, structure of the component could look something like:


```javascript
class Base extends React.Component {

  // Actions
  create = () => {
    createAPIAction();
  }
  retrieve() => {
    retrieveAPIAction();
  }
  update() => {
    updateAPIAction();
  }
  delete() => {
    deleteAPIAction();
  }

  // Elements
  render(){
    return (
          <Table {...props}/> // Table component (1. above) with actions & data as props
          <Form {...props} />  // Form component (2. above) with actions & data as props
    )
  }
}
```

To demonstrate the implementation of the ***CRUD Pattern*** using the `Base` component, we will be using a ***Library*** app that manages collection of ***Books*** and ***Members***

The ***Library*** App will have following pages (each being a component)

1. `Dashboard`: Home page of the library app that shows the total number of books and members.
2. `Books` - Base component with `BookForm` and `BookTable` as child components. Each `Book` object has `title`, `author`, `isbn` and `year` fields.
3. `Members`: Base component with `MemberForm` and `MemberTable` as child components.Each `Member` object has `name`, `phone`, `email`, `address`, `city`, `state` and `zip` fields.

Both sections (`Books` & `Member`) have similar implementation that uses *CRUD Pattern*. We'll use the `Books` component to demonstrate this.

1.  `BookTable` - UI component that renders book data (collection of objects) and triggers `update` or `delete` action on selected book object. Book data and actions are passed by parent component.
2.  `BookForm` - UI component used to  `create` a new book object or `update` an existing book object. An existing book object is passed by parent component. Form specific validations, handling API errors etc. is all contained within this component and the details are opaque to other components.
3.  `Books` - Base component that wraps `BookForm` and `BookTable` component. passes ***CRUD*** actions and book objects to child components.


We will check the details of `Books` component in this post. `BookTable` & `BookForm` implementations will be detailed in the next post.

# `Books` Component

We have defined all the CRUD API Actions in a separate file `booksActions.js`. These actions are dispatched to store in `mapDispatchToProps` object using `connect` which makes them available to the component as `props`.

We have used both local state & redux store just for demonstration purpose. Local state is good option for simple apps. Redux store is useful when the app becomes complex. The decision should be based upon the requirement & complexity.


## `initialState` - Set or Reset the State

`Books` component retrieves the books data from redux store. It also has local state. We are using an `initialState` object to set or reset this `state`. While closing a form or performing any action where we need to take care that all the state values are getting cleared, it becomes laborious to keep track of all the state values. `initialState` solves this problem. For this particular component, the entire state is contained within `initialState` object, but it is possible to have additional values in the `state` object.

```javascript
const initialState = { // set or reset state using this object
  open: false,
  bookData: {},
  error: null,
};

class Books extends React.Component {
  state = { ...initialState }; // Setting state to initial values
  ...
  closeForm = () => {
    this.setState({ ...initialState }); // Resetting state to initial values
  };
```

As can be seen above, `closeForm` will just reset the `state` to `initialState`. This becomes quite convenient, rather than having to remember which all values need to be reset. Note: We are making a *copy* of `initialState` object above.

## CRUD Actions

While retrieving & presenting the data in the table, we need to ensure the data is retrieved when the component is mounted. React has a lifecycle method `componentDidMount` which is invoked once the component is mounted. We use this method to `retrieve` the data from the server using `retrieveBooksAPIACtion`.
The retrieved data is made available to the component as `props` by `mapStateToProps` function using `connect`.

```javascript
  componentDidMount() {
    this.retrieve(); // Retrieve books when the component is mounted
  }

  retrieve = () => { // Retrieve
    this.props.retrieveBooksAPIAction(); // Retrieves the data from the server and dispatches it to reducer
  };
  ...
const mapStateToProps = ({ books }) => { // Book data retrieved from redux store made available as 'books`
  return { books };
};

const mapDispatchToProps = {
  createBookAPIAction,
  retrieveBooksAPIAction,
  updateBookAPIAction,
  deleteBookAPIAction,
};
```

`create`, `update` & `delete` actions have similar approach. In this application they are not dispatching any data through redux store so strictly speaking, mapping them above to `dispatch` is not required this is shown here for consistency.

## CRUD Elements

We are passing `update` & `delete` actions to `BookTable` & `create` action to `BookForm`. We are also passing `error` from the API Response to `BookForm` for mapping errors to related fields. We will check their implementation details in the subsequent post, which covers an interesting way of handling 'Form Validation' and 'API Response' validation etc and how this can be all wrapped inside the Form component without the functionality leaking out.

```javascript
render(){
  return (
    ...
        {/* FORM (Details View) */}
        {this.state.open ? (
          <BookForm
            open={this.state.open}
            closeForm={this.closeForm}
            formData={this.state.bookData}
            create={this.create}
            update={this.update}
            error={this.state.error}
          />
        ) : null}

        {/* TABLE (List View) */}
        <h2>Books</h2>
        <BookTable
          books={this.props.books}
          openForm={(bookData)=>this.openForm(bookData)}
          delete={this.delete}
        />
    ...
  )
}
```
<br/>

# Conclusion

As can be seen above, the `Books` component is actually a concrete implementation of the base component described earlier. There are other details about the Book components that we are not covering in this post. The actual implementation of this component with other boilerplate can be seen in the source code of the library application on following URLs.


- [front-end (React)](https://github.com/hyphenOs/library-frontend){:target="\_blank"}
- [back-end (Django REST)](https://github.com/hyphenOs/library-backend){:target="\_blank"}

*PS* In a future post, we will describe the implementation of this pattern using functional components and React Hooks.
