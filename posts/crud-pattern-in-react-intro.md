.. title: CRUD Pattern in React Intro
.. slug: crud-pattern-in-react-intro
.. date: 2020-03-12 17:02:28 UTC+05:30
.. tags:
.. category:
.. link:
.. description:
.. status: draft
.. type: text
.. summary: React.js based front-end powered by REST API can be a typical architecture in a Dashboard like applications. A Dashboard application usually consists of multiple pages with a similar structure where each page has a collection of objects and an interface to create or update an object using **CRUD** actions like **Create**, **Retrieve**, **Update** or **Delete**. In this post, we'll explore the use of CRUD pattern in a Dashboard application that implements these actions using an example app.

# Overview

I worked on a Dashboard application in my last project where front-end was written in `React.js` and the API was powered by `Django REST Framework (DRF)`. When I joined the project, some initial functionality was already in place. When I worked on fixing some issues or adding new features to the code, similar-looking code repeated at different places without a proper abstraction. Thus reasoning about the code was becoming a bit difficult. However, it was quite apparent that each page utilizes similar functionality ( essentially CRUD actions). Perhaps it made sense to think of this functionality as a pattern, which may or may not be abstracted out as a library component, but if we could structure such code in a certain way, it becomes easier to reason about and hence maintain the same code.

While 'thinking' about what such a structure or pattern could be, following can be noted. Each page required a functionality to have - a) A Summary view of a collection of objects b) A detailed view of an individual object and c) the detailed view itself can be used for updating an existing object or creating a new object, where each such object represented an entity served by a REST API.

We can implement above functionality in `React.js` as -

1. A Table UI component that shows collection of objects and triggers actions on selected object(s). Objects and actions are passed by parent component as props.
2. A Form UI component that creates a new object or updates an existing object. It will implement and handle functionality for input validation, field value change, error and API error handling by itself. The existing object is passed by parent component as props.
3. A Base component that wraps Table and Form component. It will pass CRUD actions and retrieved objects as props to child components.

Here's an overview of CRUD pattern -

> Base component structure

```javascript
class Base extends React.Component {
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
  render(){
    return (
          <Form />
          <Table />
    )
  }
}
```

The Base component implements API action handlers. These handlers call the API action and handle the API response. It also renders Form and Table components and passes objects and action handlers as props to those components.

To demonstrate such a pattern we will be using a simple "Book Library"  that manages a collection of "Books" and "Members". This is a simple enough application to demonstrate some of the ideas that are used in implementation of the pattern we call as 'React CRUD Pattern." We'll also be using react-redux for state maintenance.

Let's look at the library App Structure -

1. Dashboard: Home page of the library app that shows the total number of books and members.
2. Books - Base component with BookForm and BookTable as child components. Each Book object has `title, author, isbn and year` fields.
3. Members: Base component with MemberForm and MemberTable as child components.Each Member object has `name, phone, email, address, city, state, zip` fields.

We'll check the `Books` component into more details to demonstrate the pattern -

1.  `BookTable` - UI component that renders book data (collection of objects) and triggers update or delete action on selected book object. Book data and actions are passed by parent component
2.  `BookForm` - UI component that creates a new book object or updates an existing book object. An existing book object is passed by parent component. It does the form validation, field value change, error and API error handling by itself.
3.  Books - Base component that wraps `BookForm` and `BookTable` component. passes CRUD actions and book objects to child component

> Books component structure

```javascript
class Books extends React.Component {
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
```

As can be seen above, the `Books` component, is actually a concrete implementation of the abstract `Component` described earlier. There are other details about the Book components that we are not covering in this post. The actual implementation of this component with other boilerplate can be seen in the source code of the library application on following URLs.

- [front-end (React)](https://github.com/hyphenOs/library-frontend)
- [back-end (Django REST)](https://github.com/hyphenOs/library-backend)

We will be looking at implementation details of the Form component in a subsequent post, which covers an interesting way of handling 'Form Validation' and 'API Response' validation etc and how this can be all wrapped inside the Form component without the functionality leaking out.

If you have queries about this post, please create an issue in one of the mentioned repos. Comments are disabled for the post as of now.

