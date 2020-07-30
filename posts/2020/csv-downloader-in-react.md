.. title: CSV Downloading Functionality as a React Component
.. slug: csv-downloader-in-react
.. date: 2020-07-30 10:17:20 UTC+05:30
.. tags: ReactJS
.. author: Abhijit Gadgil
.. link:
.. description:
.. type: text
.. status:
.. summary: A common requirement in many dashboard front-end applications is to download the data that is presented in a table as a `CSV` file. In this blog post, we discuss an implementation of a React component that can be used to achieve this.


# Overview

In many dashboard applications, it's a common requirement to download the API data as a `CSV` file. In this blog post we discuss an implementation of a React component that can be used to achieve this. We also present additional benefits of implementing such component in the front-end rather than relying on the APIs alone to provide data as `CSV`.

# Motivation

Why simply an API Based approach won't work?

- When we have access to the API endpoints, we can add the download feature using the `Content-Disposition` HTTP header with `attachment` attribute. We'll need to define a separate API end-point rather than our usual `Json` based API end-point (or at-least some kind of a separate `Renderer` for the API request.). While this approach should work in theory, if your front-end is making requests using any standard components like `axios`, the requests are _Ajax_ requests and the browser does not honor the `Content-Disposition` header in such cases. Which, kind of makes sense because in an _Ajax_ request, a user being at the end of the request is not a typical case. This would mean we need to create an explicit user action by creating an `anchor` element in the front-end, outside the normal request flow. While this can be made to work, it's a little cumbersome.

- If the API calls you are making are not something you have a control over (_ie_ you only get the data as `Json`), there is no choice but to take care of this in the front-end.

We'll also see a couple of other benefits of doing this in the front-end.

# Main Idea

The main challenge we face, with the _Ajax_ requests is that - we have to 'trigger' download request as a user initiated request, rather than it being a simple _Ajax_ request. There is a neat trick that can be used using the [`Blob` object](https://developer.mozilla.org/en-US/docs/Web/API/Blob){:target="\_blank"} as shown on the MDN site. So essentially -

1. Convert the data returned by API as a String.
2. Create a `Blob` object using this string.
3. Create and Serve Url for the Blob object using [`URL.createObjectURL`](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL){:target="\_blank"} method.
4. Make the link downloadable, also by giving a file name if required.

This is shown in the code below.

```javascript

// https://github.com/hyphenOs/library-frontend/blob/master/src/common/components/CSVDownloader.js

class CSVDownloader extends React.Component {

  downloadCSV = () => {
    const arr = this.props.data;
    const csvString = this.getCSVString(arr);
    const csvBlob = this.getCSVBlob(csvString);
    this.serveCSVBlob(csvBlob);
  };

...

  // Reference: https://developer.mozilla.org/en-US/docs/Web/API/Blob
  serveCSVBlob = csvBlob => {
    const url = URL.createObjectURL(csvBlob);

    const link = document.createElement("a");
    link.href = url;
    link.setAttribute("download", "CSVData.csv");
    document.body.appendChild(link);
    link.click();
  };
...

}
```

Once we decide to process the data on the client side, there are other additional features that can be implemented on the client side. Next we'll look at a couple of such features -

# Additional Features

Other than simply translating the data from `Json` to `CSV`, it's possible to perform some other transformations on the data before the data is converted into `CSV`.

1. For instance, often we have an attribute, the API returned values for that attribute can be a `Boolean` value, say `true/false` or `0/1`, but we may want to have a more readable and meaningful value like `Open/Close` when we save the data as `CSV`. This is essentially mapping one value of the attribute to another string. This can be achieved by passing a user defined function that performs such a mapping, we call such a function as a `mapper`. So what a `mapper` essentially does is converts a set of attributes from their database (or API) values into more readable values.

2. Similarly, it's possible to have custom sorting based on some other attribute of the data rather than the default attribute which is selected by the API. This can also be achieved by a user defined function that sorts the given data using the user defined function. We call such a function `sorter` function.

These are just a couple of examples, other things like filtering out certain column from the `CSV` etc. is possible. Generally the code will look something like -

```

<CSVDownloader
    data={data}
    mapper={mapper}
    sorter={sorter}
/>

...

// Passed functions are applied to transform data
if (mapper && sorter) {
	downloadCSV(mapper(sorter(data)));
} else if (mapper) {
	downloadCSV(mapper(data));
} else if (sorter) {
	downloadCSV(sorter(data));
} else {
	downloadCSV(data);
}

```

# Wrapping up

We have seen, rather than taking an API based approach alone, implementing a functionality like downloading `CSV` in the front-end, not only allows us to work with our own as well as third party API, but also, allows us to implement additional features. Also, one more benefit of writing such a component is, this component can be reused across several parts of the application, if there are many parts that require this functionality. With some additional effort, it's quite possible to make this component as a completely reusable component completely decoupled from the application logic.

A full source code for an example implementation of this component is available at -
- [CSVDownloader React Component](https://github.com/hyphenOs/library-frontend/blob/master/src/common/components/CSVDownloader.js){:target="\_blank"}

# Thanks

[Mayur Borse](/authors/mayur-borse/): Author of the code mentioned in the blog post and an early draft.
