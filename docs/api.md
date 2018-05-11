# API

UDA has a single API that provides unified data access:

    handle = getIdamAPI(signal, source); // C
    str    = getData(signal, source)     ; IDL
    ip     = client.get(signal, source)  # Python

## C API

To build a C code using the C API you can compile using:

    gcc $(pkg-config --cflags uda-client) example.c $(pkg-config --libs uda-client) -o example
    
An example using the C API is:    
    
    #include <stdio.h>
    
    #include <uda.h>
    
    void print_int_array(const int* data, int len, int max)
    {
        printf("[");
        int i;
        const char* delim = "";
        for (i = 0; i < len; ++i) {
            if (i > max) {
                printf("%s...", delim);
                break;
            }
            printf("%s%d", delim, data[i]);
            delim = ", ";
        }
        printf("]\n");
    }
    
    void print_float_array(const float* data, int len, int max)
    {
        printf("[");
        int i;
        const char* delim = "";
        for (i = 0; i < len; ++i) {
            if (i > max) {
                printf("%s...", delim);
                break;
            }
            printf("%s%g", delim, data[i]);
            delim = ", ";
        }
        printf("]\n");
    }
    
    int main()
    {
        putIdamServerHost("idam3.mast.ccfe.ac.uk");
        putIdamServerPort(56565);
    
        printf("host: %s\n", getIdamServerHost());
        printf("port: %d\n", getIdamServerPort());
    
        int handle = idamGetAPI("ip", "18299");
    
        if (handle < 0) {
            // UDA error
            fprintf(stderr, "error: %s\n", getIdamError(handle));
            return handle;
        }
    
        printf("label: %s\n", getIdamDataLabel(handle));
        printf("units: %s\n", getIdamDataUnits(handle));
    
        int rank = getIdamRank(handle);
        printf("rank: %d\n", rank);
    
        int shape[rank];
    
        int i;
        for (i = 0; i < rank; ++i) {
            shape[i] = getIdamDimNum(handle, i);
        }
    
        printf("shape: ");
        print_int_array(shape, rank, 10);
    
        printf("dimensions:\n");
        for (i = 0; i < rank; ++i) {
            printf("  number: %d\n", i);
            printf("  label: %s\n", getIdamDimLabel(handle, i));
            printf("  units: %s\n", getIdamDimUnits(handle, i));
            printf("  data: ");
            int dim_data_type = getIdamDimType(handle, i);
            if (dim_data_type != UDA_TYPE_FLOAT) {
                fprintf(stderr, "unexpected data type for dim %d: %d\n", i, dim_data_type);
                return -1;
            }
    
            char* dim_data = getIdamDimData(handle, i);
            print_float_array((float*)dim_data, shape[i], 8);
        }
    
        int data_len = getIdamDataNum(handle);
        printf("data length: %d\n", data_len);
    
        char* data = getIdamData(handle);
        printf("data: ");
        print_float_array((float*)data, data_len, 8);
    
        return 0;
    }

## C++ API

## Fortran API

## IDL API

## Python API

An example of using the Python API is:

    # For Python 2:
    from __future__ import print_function
    
    # Import module
    import pyuda
    
    # Create a client instance
    client = pyuda.Client()
    
    # Retrieve data
    ip = client.get('ip', 18299)
    
    # Examine returned data object
    print(ip.label)
    print(ip.units)
    print(ip.data)
    print(ip.time.label)
    print(ip.time.units)
    print(ip.time.data)