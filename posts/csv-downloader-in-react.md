.. title: CSV Downloader in React
.. slug: csv-downloader-in-react
.. author: Mayur Borse
.. date: 2020-06-01 17:15:36 UTC+05:30
.. tags: CSV,
.. category: React
.. link:
.. status: draft
.. description:
.. type: text
.. summary: We are going to learn a new pattern in our React series. It's called "CSVDownloader Pattern". This pattern implements a CSVDownloader Component that serves the data (array of objects) as a downloadable CSV file using Blob. CSV is a data exchange format to move tabular data between programs. for example, a user may need to transfer information from a database to a spreadsheet. The database information can be exported as "CSV" which then can be imported by a spreadsheet program.

 While working on a project, a requirement to implement a CSVDownloader that will export the data in a CSV file was occurred, which ended up in the form of this pattern. This feature was to be implemented for more than one component. So instead of creating CSVDownloader for each page, we created a common **`CSVDownloader`** component that receives data as input and exports it as a downloadable CSV file.  We'll use **Library App** sections *Books* and *Members* for demonstration of this pattern.

Suppose, our library app has 1000 book records and we want to export the data so it can be used and viewed in different places like a spreadsheet. How can we export the data in a simple yet accurate form? CSV file format can be a solution to this problem. Let's introduce CSV first

#### What is CSV?

CSV (comma-separated values) is a file format which stores tabular data in plain text format using comma to separate values. In CSV file, each line is a data record and each data record has one or more fields separated by comma.

> #### *"The IBM Fortran first supported CSV files in 1972"*

```javascript
CSV Structure
// First row consists of column headers. Other rows represent data

A,B,C // Headers A, B and C
1,2,3 // Row1 with A=1, B=2 and C=3
4,5,6 // Row2 with A=4, B=5 and C=6
```

In our *library* app we have two sections named Books and Members. Both will pass data(array of objects) to CSVDownloader

```javascript
<Books>
    <CSVDownloader data={this.props.books}>
</Books>
<Members>
    <CSVDownloader data={this.props.Members}>
</Members>
```

***

## ***`CSVDownloader`*** Implementation

CSVDownloader Component contains a button that triggers download process on `onClick` by calling `downloadCSV()` function.

```javascript
<Button onClick={this.downloadCSV}>
    <CloudDownloadRounded />
    CSV
</Button>
```

CSVDownloader Component has the following elements

- `downloadCSV()`
    - `getCSVString()`
    - `getCSVBlob()`
    - `serveCSVBlob()`

Let's check how we export CSV from `data` using the above functions.

### ***`downloadCSV()`*** function

```javascript

downloadCSV = () => {
const arr = this.props.data;
const csvString = this.getCSVString(arr);
const csvBlob = this.getCSVBlob(csvString);
this.serveCSVBlob(csvBlob);
};

```

`downloadCSV()` gets `this.props.data` (array of objects) passed by parent component (Books or Members component) as props and passes it to `getCSVString()` function which returns it in string form (`csvString`). `csvString` is then passed to `getCSVBlob()` function  to get a Blob which is then passed to `serveCSVBlob()` function. `serveCSVBlob()` serves it in browser for download purpose.

***

### `getCSVString()` function

```javascript
getCSVString = arr => { // arr=[{'A': 1, 'B': 2, 'C': 3}, {'A': 4, 'B': 5, 'C': 6}]
const csvArray = [Object.keys(arr[0])].concat(arr); // csvArray = [['A', 'B', 'C',], arr]

return csvArray
    .map(row => {
    return Object.values(row); // [['A', 'B', 'C'], [1, 2, 3], [4, 5, 6]]
    })
    .join("\n"); // returns string "A, B, C \n 1, 2, 3 \n 4, 5, 6"
};
```

`getCSVString()` first creates a `csvArray` containing array of csv headers and the `arr` object itself.

`map` method then returns a new array with an array of headers and subsequent arrays of values from each object by calling provided function on every element of csvArray. In short, we get an array containing an array of headers and arrays of records.

`join("\n")` function creates and returns `csvString` by concatenating all the elements in csvArray with new-line `\n`. This way, we get the array data in CSV format.

***

### ***`getCSVBlob()`*** function

```javascript
getCSVBlob = csvString => {
return new Blob([csvString], { type: "text/csv" });
};
```

It creates and returns ***Blob*** object of `csvString` with the help of constructor. It also passes `type: "text/csv" as MIME-type

Well, but what exactly is ***Blob***?

### ***Blob***

***Blob*** (Binary Large Object) is a data type which holds *"binary data with type"*. We can convert our files, images into binary data and store them as Blob. Blob represents a file-like object of immutable, raw data. Being immutable, we can't change the content of blob but we can create a new Blob using its slice method.

```
    new Blob(blobParts, options); //Blob constructor

    Example
    new Blob(["id,year\n1,2020"], {type: "text/csv"} )

    blobParts is an array of ArrayBuffer/String/Blob

    ArrayBuffer is an opaque representation of bytes in memory
    just as, Blob is an opaque representation of chunk of bytes/data

    options :
      1. type: MIME-type, eg. "text/csv"
      2. endings: whether to transform the end-of-line
         according to current OS. eg. "\r\n or \n"
          - transparent: do nothing
          - native: transform
```

***

### `serveCSVBlob()` function

```javascript
serveCSVBlob = csvBlob => {
const url = URL.createObjectURL(csvBlob);

const link = document.createElement("a");
link.href = url;
link.setAttribute("download", "CSVData.csv");
document.body.appendChild(link);
link.click();
};
```

`serveCSVBlob()` receives csvBlob as input and passes it to `URL.createObjectURL()` function to serve blob as URL. Let's see how we do it.

#### Serving Blob as URL

`URL.createObjectURL(blob)` returns a unique url in the form `blob:<origin>/<uuid>`.
The generated URL is valid only for the current document, while it's open and can be referenced into `<img>` and `<a>`.
The generated URL is then set as `href` of `anchor` HTML element which is created using `createElement` function.
 The anchor element with its *download* attribute prompts the user to save the linked url instead of navigating it. So, we'll set the *download* attribute with the value of *CSVData.csv* as filename using `setAttribute`. The anchor element (link) will then be appended to document body using `appendChild`.
 Finally, we will simulate the mouse click on the anchor element(link) to download the **CSV** file

 So, this was the ***CSVDownloader Pattern***. I hope it was a fine learning experience for you. We will meet again with another pattern in our next post.
Till then,

 Good Bye!

source code:

[Front-end (React)](https://github.com/hyphenOs/library-frontend)

[Back-end (Django REST)](https://github.com/hyphenOs/library-backend)
