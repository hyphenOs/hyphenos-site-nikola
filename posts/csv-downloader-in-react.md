.. title: CSV Downloader in React
.. slug: csv-downloader-in-react
.. date: 2020-06-25 13:05:28 UTC+05:30
.. tags: ReactJS
.. link:
.. url_type: full_path
.. description:
.. status:
.. type: text
.. summary: Downloading the presented data (Table data) as a file is a very general & frequent use case. CSV is most widely used data format for data transfer. We will see how can we use the books data from the "Library App" (JSON) to create & serve downloadable csv file. This functionality can be achieved on the back-end (Django REST) itself using 'Content-Disposition' header but front-end approach has some advantages that makes it preferred choice for CSV Download. We'll use CSVDownloader component that converts the JSON data into csv string and serve it as downloadable csv file using Blob (a binary object).


# The use-case or why csv?

Data is main element of any application. We won't be using many of these apps without data. Data transfer (between computers or apps) increases the usability even further as the same data can be used for different purposes.

eg. We have the ***Library*** app that shows library books & members. We see all the books in table format. But, wouldn't it be great if we can export the books data in some 'standard data type'. Data type which is 'widely used' & simple to convert, so it can be transferred & used by many computers, applications?

CSV (Comma Separated Values) is widely used data type for data exchange between computers. It stores data in plain text using comma as separator. We can convert & download our books data (JSON) as csv file so it can be used for different purposes by different computers, applications.

# CSV Download Approaches

We know the use cases of csv & we have decided to implement it. But, there are two approaches for downloading csv.

- Back-end Approach
- Front-end Approach

Let's check both of them & see why we prefer front-end approach over Back-end approach

# Back-end Approach

We are using Django REST on the back-end in the ***Library*** app. We are using  python's 'csv' module for parsing & writing csv data. 'Content-Disposition' header is used to indicate the nature of content, whether an 'inline'(displayed in browser) or an 'attachment' (downloaded). 'Content-Type' header indicates media type of the resource.

eg. Content-Disposition: "attachment;filename='export.csv'"
    Content-Type: "text/csv"

This way we are able to download csv file on the back-end itself. But, there are some advantages on front-end that makes it a preferred way for downloading csv.

# Front-end Approach

## Declarative Approach of React

On the front-end, we can use the already available JSON data to create CSV file. We use 'Declarative Approach' in React. What it means is, we simply 'declare' the intention without mentioning the implementation. Let's see how we used the declarative approach of React for CSV download functionality

1. Create a csv string from the JSON data using getCSVString()
2. Create a Blob from the csvString using getCSVBlob()
3. Serve the blob using serveCSVBlob

```javascript
  downloadCSV = () => {
    const arr = this.props.data;
    const csvString = this.getCSVString(arr);
    const csvBlob = this.getCSVBlob(csvString);
    this.serveCSVBlob(csvBlob);
  };
```

<!-- ## Why download csv on front-end? or Use cases -->





### Data is already available

When we open the book section of library we see all the books data in table format. Now, we want to download the 'same data' as csv. This is the most general use case for a csv file. We already have the data available. Instead of making another API call for getting the data in csv format we can use the available data to create a csv string that can be served as downloadable file using Blob. This is a very good advantage of preferring front-end approach. While for the back-end approach we will have to make another API call to get the csv file.


On the front-end, we get greater amount of flexibility in terms of data mapping, filtering, sorting. These functions can be performed as per user selections.


2. Show save as dialog:
   Many browsers do not show the save as dialog & hence we cannot set another filename

3. Ability to map, sort, filter
   We can filter some unwanted fields or sort them or map them to some other values

------
# Finish
------

# Ideas scribbled

There are two approaches to download csv file
We will check, how json data is used to be served as csv file in react. This way, we can get the csv file without additional API call & can also have sorting, mapping, filtering applied when required.


we started on the backend to  get the data as attachment using 'Content-type: text/csv' & 'Content-Disposition: attachment; filename.csv' format.


1. dialog on browser

2. json to csv

3. mapping and sorting

4. filtering/ excluding fields

5.


we already have the data now we want to export it as csv. Instead of writing a new api and getting it from backend which does not always show the save as dialog, we can process the already avail

CSV Download is a much required functionality in an app. Like we have a all data of book

whenever we are having



CSV is very common data type used for data exchange since 1972 when IBM Fortran first supported it. Numerous standards exist for it.
