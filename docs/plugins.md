# Plugins

## UDA plugin architecture

Calls to UDA are made using a call similar to

    get("signal", "source")
    
The default use of this is to try and read the given `signal` from the given `source`, possibly using the
[meta-data plugin](#MetadataPlugin) to map the signal and source to a data archive specific values.

The extension to this syntax which allows for direct plugin calls is:

    get("MYPLUGIN::myfunction(argument1=value1, argument2=value2, ...)", "")

This syntax will instead directly look for an available plugin called `MYPLUGIN` and call the `myfunction` function in
this plugin. The arguments provided are given to the plugin as a list of key-value pairs.

## Developing a new plugin

### Creating the plugin

To start developing a new plugin copy the plugin from the UDA/source/plugins/template directory.

You will find two source files in this directory templatePlugin.c and templatePlugin.cpp. These can be used to develop 
C or C++ plugins respectively.

Once you have copied the template plugin rename the files to be more appropriate for your plugin, and delete the
templatePlugin file for the language you are not going to use (i.e. delete templatePlugin.c if you plan to use C++).

### Seting up the plugin build

The next step is to modify the CMakeLists.txt. A `uda_plugin` macro has been provided to make it easy to configure and
compile a UDA plugin; you will just need to set the macro fields accordingly.

The arguments to the `uda_plugin` macro are:

|Name                   |Required   |Description|
|---                    |---        |---|
|NAME                   |yes        | The name that will be used in UDA to call the plugin |
|LIBNAME                |yes        | The name of the shared library that will be built |
|ENTRY_FUNC             |yes        | The name of the function that is called to access the plugin |
|DESCRIPTION            |yes        | A short description of your plugin that will be printed by UDA |
|EXAMPLE                |yes        | An example of basic functionality provided by the plugin |
|CONFIG_FILE            |no         | The name of the environment configuration file that will be sourced at runtime (see [Config Files](#ConfigFiles))|
|SOURCES                |yes        | The source files to compile to build the plugin |
|EXTRA_INCLUDE_DIRS     |no         | Any additional directories that need to be added to the include paths at compile time |
|EXTRA_LINK_DIRS        |no         | Any additional directories that need to be added to the library search paths at link time |
|EXTRA_LINK_LIBS        |no         | Any additional libraries that need to be used at link time |
|EXTRA_DEFINITIONS      |no         | Any plugin specific macro definitions that need to be defined |
|EXTRA_INSTALL_FILES    |no         | Any addtional files that need to be be copied into the install directory along with the plugin |

For example:

    uda_plugin(
      NAME TEMPLATEPLUGIN
      ENTRY_FUNC templatePlugin
      DESCRIPTION "Standardised Plugin Template"
      EXAMPLE "TEMPLATEPLUGIN::functionName()"
      LIBNAME template_plugin
      SOURCES templatePlugin.cpp
      EXTRA_INCLUDE_DIRS
        ${LIBXML2_INCLUDE_DIR}
      EXTRA_LINK_LIBS
        ${LIBXML2_LIBRARIES}
    )

This will generate a plugin library `template_plugin.so` (on Linux) and can be called in UDA using `TEMPLATEPLUGIN::`.

### <a name="ConfigFiles"></a> Config Files

The plugin configuration files can be used to set up any environmental variables that are required by the plugin. The
configuration file should have the name `<CONFIG_FILE>.in` where `<CONFIG_FILE>` is the name specified in the `uda_plugin`
macro.

**Note**: the configuration file as specified by `<CONFIG_FILE>` must have the `.cfg` extension in order for UDA to
find it.

During `cmake` configuration the `<CONFIG_FILE>.in` will be used to generate a `<CONFIG_FILE>` in the build directory.
When the plugin is installed this generated configuration file will be copied into the UDA plugins configuration directory
and will be automatically loaded at run-time.

The `cmake` step allows for the `<CONFIG_FILE>.in` to have configuration specific variables. You can specified variables
for `cmake` to replace using the `@VARNAME@` sytnax. I.e. `export MY_PLUGIN_FILE=@CMAKE_INSTALL_DIR@/share/my_file.txt`
would get replaced with `export MY_PLUGIN_FILE=/usr/local/share/my_file.txt` if UDA is being installed into `/usr/local`.
 

### Writing a plugin

The entry function (as specified in `ENTRY_FUNC` in the build configuration) to your plugin must look like

    int entryFunc(IDAM_PLUGIN_INTERFACE* idam_plugin_interface)
    {
        ...
    }

The basic headers you will need to include to allow this to build are:

    #include <plugins/udaPlugin.h>

Other headers your are likely to need include:

    #include <clientserver/errorLog.h>
    #include <clientserver/initStructs.h>
    #include <clientserver/stringUtils.h>
    #include <clientserver/udaTypes.h>
    #include <logging/logging.h>
    
The `IDAM_PLUGIN_INTERFACE` contains the following data:

| Name | Type | Description |
| --- | --- | --- |
| interfaceVersion | unsigned short | The version of the plugin interface being passed. This is currently 1 -- if the version is higher than you expect then the plugin should report and error and exit. |
| pluginVersion | unsigned short | An out field to assign the version of your plugin that has been called. |
| housekeeping | unsigned short | A flag set to 1 if the plugin is being run in housekeeping mode -- in this mode you should tidy up an resources and exit. |
| changePlugin | unsigned short | An out field to set whether another plugin should be called after this one has finished. |
| dbgout | FILE* | The debug stream which can be written to (might be NULL -- in which case debug messages are being ignored). |
| errout | FILE* | The error stream which can be written to (might be NULL -- in which case errors are being ignored). |
| data_block | DATA_BLOCK* | The main data structure that should be populated. |
| request_block | REQUEST_BLOCK* | The structure that details the request being made to the plugin. |
| environment | const ENVIRONMENT* | A structure containing details about the environment in which the server has been run -- environmental variables etc. |
| pluginList | const PLUGINLIST* | The list of available plugins that can be called directly. |

The `request_block` field of the `IDAM_PLUGIN_INTERFACE` structure is how the function being called is specified. You should
check the value of the `request_block->function` element and respond accordingly.

For example

    REQUEST_BLOCK* request_block = idam_plugin_interface->request_block;
    
    if (STR_IEQUALS(request_block->function, "help")) {
        err = do_help(idam_plugin_interface);
    }
    
Here we are responding the `help()` function call by calling a `do_help` function. The `STR_IEQUALS` is a helper macro
defined in `udaPlugins.h` which tests for case-insensitive string equality.

Inside the function to handle a specific plugin function call you need to do the following:

1. Extract the function arguments from the `request_block->nameValueList` structure.
2. Perform the appropriate calculation.
3. Add the data to return to the `data_block` structure and return 0, or log an error and return \< 0.


#### Extracting function arguments

There are help methods and macros to extract the function arguments.

To extract a required value you can use `FIND_REQUIRED_<TYPE>_VALUE(NAME_VALUE_LIST, VARIABLE)`
(where `<TYPE>` is one of `INT`, `SHORT`, `CHAR`, `FLOAT` or `STRING`). When using this macro the `VARIABLE` is used as
both the name of variable to write the data but also the variable name to look for in the name-value list. I.e.

    int shot = 0;
    FIND_REQUIRED_INT_VALUE(request_block->nameValueList, shot);
    
Will look for an argument that has been passed to the plugin as `shot=999` as well as write this value to the `shot`
C variable.

If the argument is not required you can use the `FIND_<TYPE>_VALUE(NAME_VALUE_LIST, VARIABLE)` macro which works in the
same way as the `FIND_REQUIRED_<TYPE>_VALUE` macro except no error is thrown if the variable is not found.

Arrays can also be passed as UDA plugin arguments using `;` separated lists, i.e. `PLUGIN::func(mylist=1;2;3;4)` passes
the list `[1, 2, 3, 4]` as the `mylist` argument. The macros `FIND_REQUIRED_<TYPE>_ARRAY` and `FIND_<TYPE>_ARRAY` are
available to extract these list arguments. These macros require an integer variable to exist with the name
`n<VARIABLE>` which is used to write the length of the list found. I.e.

    int* indices = NULL;
    size_t nindices = 0;
    FIND_REQUIRED_INT_ARRAY(request_block->nameValueList, indices);

Will extract values from `indices=1;2;3` and set the `indices` to be equal to the array of these values, with `nindices`
equal to the number of values found.

#### Setting the return data

The data to be returned by the plugin needs to be set using the `data_block` field on the `IDAM_PLUGIN_INTERFACE` structure.
The important fields on the `DATA_BLOCK` structure that need to be set are:

| Name | Type | Description |
| --- | --- | --- |
| rank | unsigned int | The number of dimensions of the data -- for scalar data set this to 0 and `dims` to `NULL`. |
| data_type | int | The type of the data -- should be one of values of the `UDA_TYPE` enum in `udaTypes.h`. |
| data_n | int | The number of elements of data being returned. |
| data | char* | The actual data returned. |
| dims | DIMS* | An array of `DIMS` structures specifying the dimensions of the data. |

If `rank` \> 0 then you also need to set the `dims` field to an array describing the dimensions of the data. The important
fields of the `DIMS` structure that need to be set are:

| Name | Type | Description |
| --- | --- | --- |
| data_type | int | The type of the dimension data -- should be one of values of the `UDA_TYPE` enum in `udaTypes.h`. |
| dim_n | int | The number of elements dimension data being returned. |
| dim | char* | The actual dimension data returned. |

You can also set a dimension to be automatically generated by setting `compressed=1`, `dim0` to the starting value and
`diff` to be the dimension stride. In this case `dim` can be `NULL`. I.e.

    DIMS dim;
    dim.compressed = 1;
    dim.dim_n = 100;
    dim.dim0 = 0;
    dim.diff = 1;
    dim.dim = NULL;
    
Would generate the dimension `[0, 1, 2, 3, ..., 99]`.

There are also helper functions to allow easy returning of common types of data. These are:

    setReturnData<TYPE>Array(DATA_BLOCK* data_block, float* values, size_t rank, const size_t* shape, const char* description);
    setReturnData<TYPE>Scalar(DATA_BLOCK* data_block, float value, const char* description);
    setReturnDataString(DATA_BLOCK* data_block, const char* value, const char* description);
            
Where `<TYPE>` is one of `Float`, `Double`, `Int`, `Long` and `Short`.

In these helper functions `description` is an optional description to add to the returned data and can be `NULL`.


## Writing plugin tests

*TODO*

## <a name="SpecialPlugins"></a> Special plugins

There a few special plugins in UDA which are used to provide some special syntax support. These plugins are optional
but if provided can allow for powerful functionality to be tailored to a specific experiment and data archive.

The special plugins are:

1. Meta-data plugin: A plugin used to provide signal aliasing and querying, usually using a meta data database.
2. Provenance plugin: A plugin used to provide data access provenance.
3. File reader plugins: Plugins which are used to read different file formats. Some of these are provided by default,
 such as the HDF5 and NetCDF plugins. 

### <a name="MetadataPlugin"></a> Meta-data plugin

### Provenance plugin

### File reader plugins