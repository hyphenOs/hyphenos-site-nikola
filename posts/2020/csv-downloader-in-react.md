.. title: CSV Downloader in React
.. slug: csv-downloader-in-react
.. date: 2020-07-17 13:17:20 UTC+05:30
.. tags: ReactJS
.. link:
.. description:
.. type: text
.. status:
.. summary: This post describes requirement and implementation details of CSVDownloader component in React. This component converts and serves provided data as blob and is usable across different pages.

# Overview

In our day to day life as user, we frequently need to download data presented on the UI so it could be ported and used for other applications. CSV format is widely used for this purpose being simple and compatible. A CSVDownloader component in React can be used across pages to convert and download the data. There was a requirement to download data presented on UI in csv format across multiple pages. After many trials and errors I moved on the client side implementation when the `content-disposition` header didn't work with AJAX request. I'm writing this post so those who are solving similar problems can have all details at one place as I didn't find a single source where I can find all the implementation details. A client side implementation was preferred for this feature instead of having partial back-end and partial front-end implementation. This gave a benefit to have additional features in the implementation.

# Motivation

While thinking about the implementation approach we will come across two scenarios.

- When we have access to create/modify the API endpoints, we can try to add the download feature using the `content-disposition` header with `attachment` attribute to indicate to browser that the data is to be downloaded. But when the request is of type AJAX (as majority of the requests are) it is not possible to prompt this action as the request is asynchronous in nature (as server doesn't expect user on receiving end). We use the `anchor` tag to download the data but then we need to provide the full url path.

- When we don't have access to create/modify the API endpoints, we simply have no option but to implement the feature on the client side. The additional benefit of this approach is we can transform the data if required by using additional features on the client-side.

# Core Idea

A requirement to download data presented on UI in common format like csv arises very frequently. We can provide a button which downloads the data in csv format on click. This functionality can be provided in React component which transforms the data into `blob` and serves it as csv file in browser. Additionally, it could also apply functions like sorting and mapping on the data. We are using `blob` which is a file-like object to store and serve the csv data. It is well described and documented on <a href="https://developer.mozilla.org/en-US/docs/Web/API/Blob">`MDN` blob page</a>

Downloading csv is done in three step.

1. convert provided data into string
2. Create a `blob` object using converted string
3. Create and serve url for the blob object using <a href="https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL">URL.createObjectURL()</a> method. In this step an `anchor` link of `blob` is created and the link click is simulated which shows the download popup.

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

# Additional Implementations

The use of client side approach provides ability to transform the data with help of additional features. It provides better experience as user is able to customize the output as necessary. We will check the implementation details of some of the optional features like mapping and sorting.

## Mapper

  Useful to map value to relevant string. eg. The data may have field like is_available: 0/1 or is_available: true/false. We can map this to more relevant value like is_available: "Yes/No". It will be easy for the user to relate with the value.

## Sorter

  Useful when the user wants to get the data sorted by selected column instead of default sort. eg. Default sort may be by id but user wants to sort it by name or data or any other column.

```
<CSVDownloader
    data={data}
    mapper={mapper}
    sorter={sorter}
/>
...
// Passed functions are applied to transform data
if (mapper && sorter) downloadCSV(mapper(sorter(data)));
else if (mapper) downloadCSV(mapper(data));
else if (sorter) downloadCSV(sorter(data));
else downloadCSV(data);
```

# Wrapping up

The client-side implementation gives us following benefits:

1. Data transformation can be done by the component using additional features like sorting, mapping etc. These features can be optionally provided with data.
2. This component is reusable, that can be used across several pages. With some effort, this can also be used across different applications.

We checked the use of `blob` object to download data on the client side. This approach provides the advantages of having additional features like sorting, maping for data transformation. The CSVDownloader component implementation can be found <a href="https://github.com/hyphenOs/library-frontend/blob/master/src/common/components/CSVDownloader.js">here</a>
