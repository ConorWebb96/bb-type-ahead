# Bb-type-a-head
This is a readme for the type a head field custom plugin.

# Description
A component used for filtering to query tables, search through results, select and then save selected items back to the db.

## Configuration

### General Section
This has your main elements; your table select, types and fields.\
<img width="270" alt="General settings" src="https://github.com/ConorWebb96/bb-typehead/assets/126772285/c62f41c6-e3d4-47f3-89df-1feea5143686">
* The **Table** field is populates the selectable fields which can be binded to. **REQUIRED**
* **Type**, there are 3 different types, these change which fields can be selected. **REQUIRED**
* **Field**, this is used to select the field you wish to target/save back to the database after selecting. **REQUIRED**

### Configuration Section
This section is all related to functionality within the component itself.\
<img width="275" alt="Screenshot 2023-06-02 at 12 05 06" src="https://github.com/ConorWebb96/bb-typehead/assets/126772285/072d9baf-95fd-4f40-b703-b7afe2cb6332">
* **Limit**, this limits the number of results you can get back, the default being 25 and the max being 1000.
* **Type**, works similarily to the one within general however theres only 2 instead of 3. **REQUIRED**
  * The standard field type search isn't case senative.
  * Array type only works for exact searches **including case**. (Currently you can't search for more than one option using this.)  
* **Search Field**, selectable which you wish to search against. **REQUIRED**
* **Label**, field used to for visible select options in the frontend.
* **Value**, field ussed for the hidden values for binding/saving back to the database.

##Demo
### Fields - Single select
![String type a head example](https://github.com/ConorWebb96/bb-typehead/assets/126772285/edc49b0a-2dd5-48da-808c-9d6e30ff1841)

### Arrays - Multi Select
![Array demo for type a head](https://github.com/ConorWebb96/bb-typehead/assets/126772285/43895e08-1a7a-4c0b-8e01-437e2608086e)

### Relationship Multi-select
![Relationship search and assign example](https://github.com/ConorWebb96/bb-typehead/assets/126772285/b94b7fa6-02fe-496f-9bf4-546f74339e97)

## Disclaimer
You can search on arrays e.g. multi-select inputs within Budibase. However you can only search one of the values within these. In the future I may expand this to search for multiple valeus within these field types.
