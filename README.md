# omero2pandas

A convenience package to download data from OMERO.tables into Pandas dataframes.

# Installation

omero2pandas can be installed with pip on Python 3.6+:
```
pip install omero2pandas
```
omero2pandas also supports authentication using tokens generated by `omero-user-token`. 
Compatible versions can be installed as follows:
```
pip install omero2pandas[token]
```
See the [omero-user-token documentation](https://github.com/glencoesoftware/omero-user-token) for more information.


# Usage

```python
import omero2pandas
df = omero2pandas.read_table(file_id=402)
df.head()
```

Tables can be referenced based on their OriginalFile's ID or their Annotation's ID. 
These can be easily obtained by hovering over the relevant table in OMERO.web, which shows a tooltip with these IDs.

To avoid loading data directly into a dataframe, you can also download directly into a CSV:

```python
import omero2pandas
omero2pandas.download_table("/path/to/output.csv", file_id=2, chunk_size=1000)
```

`chunk_size` can be specified when both reading and downloading tables. It determines 
how many rows are loaded from the server in a single operation.

### Supplying credentials

Multiple modes of connecting to OMERO are supported. If you're already familiar with omero-py, you can supply a premade client:

```python
import omero
import omero2pandas
my_client = omero.client(host="myserver", port=4064)
df = omero2pandas.read_table(file_id=402, omero_connector=my_client)
df.head()
```

Alternatively, your connection and login details can be provided via arguments:

```python
import omero2pandas
df = omero2pandas.read_table(file_id=402, server="omero.mysite.com", port=4064,
                             username="myuser", password="mypass")
df.head()
```

If you have `omero_user_token` installed, an existing token will be automatically detected and used to connect:

```python
import omero2pandas
df = omero2pandas.read_table(file_id=402)
df.head()
```

You can also generate the connection object separately using the built-in wrapper:
```python
import omero2pandas
connector = omero2pandas.connect_to_omero(server="myserver", port=4064)
# User will be prompted for any missing connection info. 

df = omero2pandas.read_table(file_id=402, omero_connector=connector)
df.head()
```

When prompting for missing connection information, the package automatically detects whether 
omero2pandas is running in a Jupyter environment. If so, you'll get a login widget to complete details.
Otherwise a CLI interface will be provided.

This behaviour can be disabled by supplying `interactive=False` to the connect call.

## Reading data

Several utility methods are provided for working with OMERO.tables. These all support the full range of connection modes.

Fetch the names of the columns in a table:
```python
import omero2pandas
columns = omero2pandas.get_table_columns(annotation_id=142)
# Returns a list of column names
```

Fetch the dimensions of a table:
```python
import omero2pandas
num_rows, num_cols = omero2pandas.get_table_size(annotation_id=12)
# Returns a tuple containing row and column count.
```


You can read out specific rows and/or columns
```python
import omero2pandas
my_dataframe = omero2pandas.read_table(file_id=10, 
                                       column_names=['object', 'intensity'],
                                       rows=range(0, 100, 10))
my_dataframe.head()
# Returns object and intensity columns for every 10th row in the table
```

Returned dataframes also come with a pandas index column, representing the original row numbers from the OMERO.table.

## Writing data

Pandas dataframes can also be written back as new OMERO.tables.
N.b. It is currently not possible to modify a table on the server.

Connection handling works just as it does with downloading, you can 
provide credentials, a token or a connection object.

To upload data, the user needs to specify which OMERO object(s) the table
will be associated with. This can be achieved with the `parent_id` and 
`parent_type` arguments. Supported objects are Dataset, 
Well, Plate, Project, Screen and Image.

```python
import pandas
import omero2pandas
my_data = pandas.read_csv("/path/to/my_data.csv")
ann_id = omero2pandas.upload_table(my_data, "Name for table", 
                                   parent_id=142, parent_type="Image")
# Returns the annotation ID of the uploaded FileAnnotation object
```

Once uploaded, the table will be accessible on OMERO.web under the file 
annotations panel of the parent object. Using unique table names is advised.

### Linking to multiple objects

To link to multiple objects, you can supply a list of `(<type>, <id>)`
tuples to the `links` parameter. The resulting table's FileAnnotation 
will be linked to all objects in the `links` parameter (plus 
`parent_type`:`parent_id` if provided).


```python
import omero2pandas
ann_id = omero2pandas.upload_table(
    "/path/to/my.csv", "My table", 
    links=[("Image", 101), ("Dataset", 2), ("Roi", 1923)])
# Uploads with Annotation links to Image 101, Dataset 2 and ROI 1923 
```

Links allow OMERO.web to display the resulting table as 
an annotation associated with those objects.


### Large Tables
The first argument to `upload_table` can be a pandas dataframe or a path to a 
.csv file containing the table data. In the latter case the table will be read 
in chunks corresponding to the `chunk_size` argument. This will allow you to 
upload tables which are too large to load into system memory.

```python
import omero2pandas
ann_id = omero2pandas.upload_table("/path/to/my.csv", "My table", 
                                   142, chunk_size=100)
# Reads and uploads the file to Image 142, loading 100 lines at a time 
```

The `chunk_size` argument sets how many rows to send with each call to the server. 
If not specified, omero2pandas will attempt to automatically optimise chunk 
size to send ~2 million table cells per call (up to a max of 50,000 
rows per message for narrow tables).




# Advanced Usage

This package also contains utility functions for managing an OMERO connection.

`omero2pandas.connect_to_omero()` takes many of the arguments from the other functions and returns an `OMEROConnection` object.

The `OMEROConnection` handles your OMERO login and session, cleaning everything up automatically on exit. This has some accessory methods to access useful API calls:

```python
import omero2pandas
connector = omero2pandas.OMEROConnection()
connector.connect()
client = connector.get_client()
blitz = connector.get_gateway()
```
When a client is active within the `OMEROConnection` object, calls to this wrapper class will also be forwarded directly to the client object.

OMEROConnection objects can also be used as a context manager:
```python
import omero2pandas
with omero2pandas.OMEROConnection(server='my.server', port=4064, 
                                  username='test.user',) as connector:
    blitz = connector.get_gateway()
    image = blitz.getObject('Image', id=100)
    # Continue using the standard OMERO API.
```

The context manager will handle session creation and cleanup automatically.

### Connection Management

omero2pandas keeps track of any active connector objects and shuts them down 
safely when Python exits. Deleting all references to a connector will also 
handle closing the connection to OMERO gracefully. You can also call 
`connector.shutdown()` to close a connection manually.

By default omero2pandas also keeps active connections alive by pinging the 
server once per minute (otherwise the session may timeout and require 
reconnecting). This can be disabled as follows 

```python
omero2pandas.connect_to_omero(keep_alive=False)
```

N.b. omero2pandas uses a different system from the native OMERO API's
`client.enableKeepAlive` function, using both is unnecessary.

### Querying tables

You can also supply [PyTables condition syntax](https://www.pytables.org/usersguide/condition_syntax.html) to the `read_table` and `download_table` functions.
Returned tables will only include rows which pass this filter.

**Basic syntax**

Select rows representing objects with area greater than 20:
```python
omero2pandas.read_table(file_id=10, query='(area>20)')
```

**Multiple conditions**

Select rows representing objects with an even ID number lower than 50:
```python
omero2pandas.read_table(file_id=10, query='(id%2==0) & (id<50)')
```

**Complex conditions** 

Select rows representing objects which originated from an ROI named 'Nucleus':
```python
omero2pandas.read_table(file_id=10, query='x!="Nucleus"', variables={'x': omero.rtypes.rstring('Roi Name')})
```

N.b. Column names containing spaces aren't supported by the native syntax, but can be supplied as variables which are provided by the `variables` parameter.

The variables map needs to be a dictionary mapping string variables to [OMERO rtypes](https://omero.readthedocs.io/en/v5.6.9/developers/GettingStarted/AdvancedClientDevelopment.html#rtypes) objects rather than raw Python objects. 
These should match the relevant column type. Mapped variables are substituted into the query during processing.

A `variables` map usually isn't needed for simple queries. The basic condition string should automatically get converted to a meaningful type, but when this fails 
replacing tricky elements with a variable may help.
