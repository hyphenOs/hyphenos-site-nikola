.. title: CRUD Pattern in React - Implementation
.. slug: crud-pattern-in-react-implementation
.. date: 2020-07-08 12:33:18 UTC+05:30
.. tags: ReactJS
.. category: Mayur Borse
.. link:
.. description:
.. type: text
.. status:
.. summary: This is a continuation post of our <strong>CRUD Pattern</strong> series. We introduced the pattern with the <em>Books</em> (parent) component in the previous post <a href="/blog/crud-pattern-in-react-introduction">here</a>. This post will detail the implementation details of <em>BookForm</em> and <em>BookTable</em> (child) components.

# Recap

In the previous post <a href="/blog/crud-pattern-in-react-introduction">here</a>, we checked the structure of `Books` component that implements CRUD actions and passes these actions and data (retrieved from API) to its child components `BookTable` and `BookForm`. In this post, We will check implementation details of `BookTable` and `BookForm` components.

# BookTable Component - Implementation

A table view is generally used to provide summary of the data. It may also provide an interface (eg. button) to trigger _Update_ and _Delete_ actions on the selected data record. The `BookTable` component presents book entities in a summary view in table format. It triggers _Update_ and _Delete_ actions using `showEditForm` and `performDelete` callback functions, passed as `props` by the `Books` component (parent). It also passes the data required for the actions (book object, book id) back to the parent component.

React supports **Unidirectional/One-Way Data Flow** only. It means, the data is passed from parent to child as `props` which are immutable arguments and not the other way around. Our actions (update, delete) are defined in the `Books` component (parent), but we need the data (selected book id or object) from `BookTable` component (child) to perform the actions. It is not possible to pass data back in _Unidirectional Data Flow_. So how do we _pass back_ data from child to parent component? We are able to achieve it by having an **Inverse Data Flow**.

## Inverse Data Flow

- A callback function defined in the parent component is passed as props to the child component.
- The function is invoked by the child component for an event.
- eg. to trigger update or delete action (function invoke) on button click (event).
- The child component can pass data as an argument to the callback function. As the callback function is defined in the parent, the data is made available to the parent for further action.
- As data is passed in the _inverse_ direction (from child to parent), **Inverse Data Flow** is achieved.
- The passed data can then be used by the parent component to call an API action or update the state and re-render the page or any other purpose.

# BookForm Component - Implementation

A form component creates or updates an entity on the server by performing API actions. Before doing this, it needs to fulfill requirements like field validations, generating and mapping client side errors to respective fields and mapping errors from API response to respective fields. We can fulfill these requirements by using a `formFieldAttributes` object that defines validation rules for each field. We will check its details in the next section.