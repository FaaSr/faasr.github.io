# R APIs

### faasr_get_file

Usage: `faasr_get_file(local_file, remote_file, server_name="", local_folder=".", remote_folder=".")`

This function gets (i.e. downloads) a file from an S3 bucket to be used by the FaaSr function.

`server_name` is a string with name of the S3 bucket to use; it must match a name declared in the workflow configuration JSON file.
This is an optional argument; if not provided, the default S3 server specified as `DefaultDataStore` in the workflow configuration JSON file is used.

`remote_folder` is string with the name of the remote folder where the file is to be downloaded from. This is an optional argument that defaults to `"."`

`remote_file` is a string with the name for the file to be downloaded from the S3 bucket. This is a required argument.

`local_folder` is a string with the name of the local folder where the file to be downloaded is stored. This is an optional argument that defaults to `"."`

`local_file` is a string with the name for the file downloaded from the S3 bucket. This is a required argument.

Examples:

```r
faasr_get_file(remote_folder="myfolder", remote_file="myinput1.csv", local_file="input1.csv")
faasr_get_file(server_name="My_Minio_Bucket", remote_file="myinput2.csv", local_file="input2.csv")
```

### faasr_put_file

Usage: `faasr_put_file(local_file, remote_file, server_name="", local_folder=".", remote_folder=".")`

This function puts (i.e. uploads) a file from the local FaaSr function to an S3 bucket.

`server_name` is a string with name of the S3 bucket to use; it must match a name declared in the workflow configuration JSON file.
This is an optional argument; if not provided, the default S3 server specified as `DefaultDataStore` in the workflow configuration JSON file is used.

`local_folder` is a string with the name of the local folder where the file to be uploaded is stored. This is an optional argument that defaults to `"."`

`local_file` is a string with the name for the file to be uploaded to the S3 bucket. This is a required argument.

`remote_folder` is string with the name of the remote folder where the file is to be uploaded to. This is an optional argument that defaults to `"."`

`remote_file` is a string with the name for the file to be uploaded to the S3 bucket. This is a required argument.


Examples:

```r
faasr_put_file(local_file="output.csv", remote_folder="myfolder", remote_file="myoutput.csv")
faasr_put_file(server_name="My_Minio_Bucket", local_file="output.csv", remote_file="myoutput.csv")
```

### faasr_get_folder_list

Usage: `folderlist <- faasr_get_folder_list(server_name="", prefix="")`

This function returns a list with the contents of a folder in the S3 bucket. 

`server_name` is a string with name of the S3 bucket to use; it must match a name declared in the workflow configuration JSON file.
This is an optional argument; if not provided, the default S3 server specified as `DefaultDataStore` in the workflow configuration JSON file is used.

`prefix` is a string with the prefix of the folder in the S3 bucket. This is an optional argument that defaults to `""`

Examples:

```r
mylist1 <- faasr_get_folder_list(server_name="My_Minio_Bucket", prefix="myfolder")
mylist2 <- faasr_get_folder_list(server_name="My_Minio_Bucket", prefix="myfolder/mysubfolder")
```

### faasr_delete_file

Usage: `faasr_delete_file(remote_file, server_name="", remote_folder="")`

This function deletes a file from the S3 bucket.

`server_name` is a string with name of the S3 bucket to use; it must match a name declared in the workflow configuration JSON file.
This is an optional argument; if not provided, the default S3 server specified as `DefaultDataStore` in the workflow configuration JSON file is used.

`remote_folder` is string with the name of the remote folder where the file is to be deleted from. This is an optional argument that defaults to `""`

`remote_file` is a string with the name for the file to be deleted from the S3 bucket. This is a required argument.

Examples:

```r
faasr_delete_file(remote_folder="myfolder", remote_file="myoutput.csv")
faasr_delete_file(server_name="My_Minio_Bucket", remote_file="myoutput.csv")
```

### faasr_arrow_s3_bucket

Usage: `faasr_arrow_s3_bucket(server_name, faasr_prefix)`

This function configures an S3 bucket to use with Apache Arrow.

`server_name` is a string with name of the S3 bucket to use; it must match a name declared in the workflow configuration JSON file.
This is an optional argument; if not provided, the default S3 server specified as `DefaultDataStore` in the workflow configuration JSON file is used.

`faasr_prefix` is a string with the prefix of the folder in the S3 bucket. This is an optional argument that defaults to `""`

It returns a list that is subsequently used with the Arrow package.

Examples:

```r
mys3 <- faasr_arrow_s3_bucket()
myothers3 <- faasr_arrow_s3_bucket(server_name="My_Minio_Bucket", faasr_prefix="myfolder")
frame_input1 <- arrow::read_csv_arrow(mys3$path(file.path(folder, input1)))
frame_input2 <- arrow::read_csv_arrow(mys3$path(file.path(folder, input2)))
arrow::write_csv_arrow(frame_output, mys3$path(file.path(folder, output)))
```

### faasr_log

Usage: `faasr_log(log_message)`

This function writes a log message to a file in the S3 bucket, to help with debugging.
The default S3 server for logs is `DefaultDataStore` as specified in the workflow configuration JSON file.
This default can be overridden with `LoggingDataStore` in  the workflow configuration JSON file.

`log_message` is a string with the message to be logged.

Example:

```r
log_msg <- paste0('Function compute_sum finished; output written to ', folder, '/', output, ' in default S3 bucket')
faasr_log(log_msg)
```

### faasr_rank

Usage: `faasr_rank()`

Only applicable for workflows with functions that are triggered for concurrent execution using rank.
This returns this function's invocation rank as a list with two values:

`list$max_rank` - an integer that determines to the maximum number of invocations (e.g. N)
`list$rank` - an integer that determines this function's invocation rank (e.g. a number ranging from 1 to N)

Example:

In a workflow, suppose you have declared that a function `start` invokes a successor function `compute` with rank N=10, where each instance of `compute` works to produce an output file `myoutput_i.csv`, where i is a number from 1 to 10.

At runtime, `start` invokes 10 instances of `compute`; `faasr_rank()` determines the value of `i` for each instance of `compute`, which can be used to construct the file name:

```r
rank_list <- faasr_rank()
my_rank <- rank_list$rank
max_rank <- rank_list$max_rank
local_file <- paste0("myinput_", my_rank, ".csv")
write.csv(my_data, local_file, row.names=FALSE)

```

### faasr_invocation_id

Usage: `faasr_invocation_id()`

Returns this action's invocation ID.

Example:

```r
invocation_id <- faasr_invocation_id()
```

### faasr_get_s3_creds

Usage: `faasr_get_s3_creds(server_name="")`

Returns the credentials and connection details for an S3 data store as a list, so you can construct your own S3 client when the standard get/put file APIs are not sufficient. (For Apache Arrow specifically, use `faasr_arrow_s3_bucket` above instead.)

`server_name` is a string with the name of the S3 data store; if not provided, the `DefaultDataStore` specified in the workflow configuration JSON file is used.

Example:

```r
creds <- faasr_get_s3_creds()
```

### faasr_secret

Usage: `faasr_secret(secret_name)`

Returns the value of a named secret from your workflow's secret store. This is useful for passing credentials such as third-party API keys to your function.

`secret_name` is a string with the name of the secret to retrieve, as configured in your workflow.

Example:

```r
api_key <- faasr_secret("OpenWeather_API_Key")
```

### faasr_return

Usage: `faasr_return(return_value)`

Returns a value from your user function to FaaSr. This is the mechanism used for [conditional invocation]: return `TRUE` or `FALSE` to determine which successor action(s) are invoked.

`return_value` is the value (typically `TRUE` or `FALSE`) returned to FaaSr.

Example:

```r
faasr_return(TRUE)
```

### faasr_exit

Usage: `faasr_exit(message=NULL, error=TRUE, traceback=NULL)`

Terminates the current function early and reports back to FaaSr, optionally with a message.

`message` is a string with the message to report. `error` indicates whether this is an error exit (defaults to `TRUE`). `traceback` is an optional traceback string.

Example:

```r
faasr_exit(message="Input file missing", error=TRUE)
```

[conditional invocation]: conditional.md
