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