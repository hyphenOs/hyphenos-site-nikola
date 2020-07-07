# CRUD Pattern Implementation - Approach

## Recap (Books component) and brief overview (BooksTable, BookForm)
## BookTable Implementation and how to achieve Inverse data flow
## BookForm component overview

BookForm component is used to create or update a book entity. It needs to implement validation mechanism to ensure the security and sanctity of the user input data. The mechanism validates user input, generates field specific errors (from client as well as server side) and maps generated errors to respective field. This way, it
1. Ensures the submitted data is in the required format.
2. Informs and lets the user correct invalid data entries (if any) right away which results in a better user experience and performance as API calls are made only when the data is valid.
3. Also informs and lets the user correct errors from the API response (server-side).

The validation process is accomplished in ordered way. Firstly, the rules for validation are defined in a `formFieldAttributes` object in the form of attributes. Then, the input data is validated against these attributes by `formValidator()` function which generates and returns `formError` object. The errors from API response are generated in similar manner by `apiErrorsToFormFields()` function as `apiErrors` object. `formError` and `apiErrors` are collection of error objects where each object is in {fieldId: {error: true/false, helperText: "error text"}} format. Lastly, the generated errors are then mapped to the related field using its `id` attribute by `fieldError()` and `fieldHelperText()` functions. A `key` attribute of each field from `formFieldAttributes` object is assigned as `id` to respective field. This attribute is used to generate error on both sides and map generated errors to respective field.

Here, we are following the same pattern for generating and mapping errors from client-side and server-side. Now, we will check the above mentioned actions in detail


### `apiToFormFieldIDs` object to specify validation rules and customise field attributes
- Uses `key` attribute to generate and map errors to related field by usning `key` as `id` for the respective field
- Validation props like `required`, `editable` are used to bypass validation field validation
- `customValidator` is used to define custom validation rules for the field. Returns {error, helperText} error object


### `Form Validator` - Validate form fields as per specified rules and Generate `formError` object
### `apiErrorsToFormFields` - generate `apiError` object using API response errors
### `fieldError` and `fieldHelperText` - map errors from `formErrors` and `apiErrors` to related field
