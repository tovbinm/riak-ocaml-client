riak-ocaml-client
=====

**© 2012 Dave Parfitt**

riak-ocaml-client is a Riak 1.2 Protobuffs-only client for OCaml 3.12.1. Future versions may support all HTTP operations.

Pretty docs [here](http://metadave.github.com/riak-ocaml-client/).

##Dependencies

* [ocamlfind](http://projects.camlcity.org/projects/findlib.html)
* [Piqi](http://piqi.org/) 
* [http://code.google.com/p/protobuf/](Protobuffs)
   * On OSX, `brew install protobuf` if you are using Homebrew
* [http://ounit.forge.ocamlcore.org/](OUnit)

## Installation

```
./configure
make 
make install

# To test, run this:
./configure --enable-tests
export RIAK_OCAML_TEST_IP="127.0.0.1"
export RIAK_OCAML_TEST_PORT=8087
make test
```
  **Note**: Testing requires a running instance of Riak 1.2+. By default, it tries to connect to Riak on 127.0.0.1, port 8081. To override these values, simply export the following two environment variables:
  
* RIAK_OCAML_TEST_IP
* RIAK_OCAML_TEST_PORT

## Tutorial

###Hello world

The following program makes a connection to Riak and sends a ping message. 

```
open Riak
open Sys
open Unix

let client() =
    let conn = riak_connect_with_defaults "127.0.0.1" 8087 in
    let _ = match riak_ping conn with
        | true  -> print_endline("Pong")
        | false -> print_endline("Error")
    in
    riak_disconnect conn;
    exit 0;;

handle_unix_error client ();;

```


Compile this example with the following:

```
   	ocamlfind ocamlc -o foo -package Unix -package oUnit -package \
   	    piqi.runtime -package riak -linkpkg foo.ml

```

**Note**: Change the IP/port to the value defined in the Riak app.config `pb_port` and `pb_ip`.

**Note**: If compiling this example from the same directory as the Riak Ocaml Client source, you will see an error like this:
`findlib: [WARNING] Interface Riak.cmi occurs in several directories: ., /usr/local/lib/ocaml/site-lib/riak`

## Development Guide
 
### A note on types

Throughout the docs, you will find the following types. Almost all are strings:
 
```
type riak_bucket = string
type riak_key = string
type riak_client_id = string
type riak_mr_query = string
type riak_mr_content_type = Riak_MR_Json | Riak_MR_Erlang
type riak_2i_name = string
type riak_2i_range_min = string
type riak_2i_range_max = string
type riak_search_query = string
type riak_search_index = string
type riak_node_id = string
type riak_version = string
type riak_vclock = string
```

See ./src/Riak.mli for the complete interface.
 
### Connect/Disconnect

```
val riak_connection_defaults : riak_connection_options

val riak_connect_with_defaults : string -> int -> riak_connection

val riak_connect : string -> int -> riak_connection_options -> riak_connection

val riak_disconnect : riak_connection -> unit

```

To connect using default connection properties: 

```
    let conn = riak_connect_with_defaults "127.0.0.1" 8081
```   
   
  * **Note**: Pooling of connections isn't implemented, but feel free to roll your own.
   
#####Default connection properties:
The following defaults are used when calling `riak_connect_with_defaults`.

* disable Nagle's algorithm for better performance
* try an operation 3 times before an exception is thrown
* throw an exception if siblings are encountered    

To override these values:

```
	let options = 
	    { riak_connection_defaults with riak_conn_retries=5 } in
	let conn = riak_connect "127.0.0.1" 8081 options in
	...
```

To disconnect:    

    riak_disconnect conn

### Ping

```
	val riak_ping : riak_connection -> bool
```

**Example**

```
	match riak_ping conn with
    	| true -> ()
    	| false -> assert_failure("Can't connect to Riak")
```

### Client ID

```
val riak_get_client_id : riak_connection -> riak_client_id

val riak_set_client_id : riak_connection -> riak_client_id -> unit
```

**Example**

```
	let test_client_id = "foo" in
	let _ = riak_set_client_id conn test_client_id in
	let client_id = riak_get_client_id conn in
	...
```

### Server Info

```
val riak_get_server_info : riak_connection -> riak_node_id * riak_version
```

**Example**

```
	let (node, version) = riak_get_server_info conn in
```

### Get

```
val riak_get :
  riak_connection -> 
  riak_bucket -> 
  riak_key -> 
  riak_get_option list -> 
  riak_object option
```

**Example**	

```
let result = riak_get conn "my_bucket" "my_key" [Get_basic_quorum false; Get_head true] in
…
```

**type riak_get_option =**

- **Get_r** of *riak_tunable_cap*
  
  Read quorum. How many replicas need to agree when retrieving the object. Default is defined per bucket. See the Tunable CAP Options section below.
      
- **Get_pr** of *riak_tunable_cap*
   
	Primary read quorum. How many primary replicas need to be available when retrieving the object. Default is defined per bucket. See the Tunable CAP Options section below.

- **Get_basic_quorum** of *bool*

	Whether to return early in some failure cases (eg. when r=1 and you get 2 errors and a success basic_quorum=true would return an error). Default is defined per bucket.

- **Get_notfound_ok** of *bool*
  
	Whether to treat notfounds as successful reads for the purposes of R (default is defined per the bucket). Default is defined per bucket.
 
- **Get_if_modified** of *string*

	When a vclock is supplied as this option only return the object if the vclocks don't match
 
- **Get_head** of *bool*

	Return the object with the value(s) set as empty - allows you to get the metadata without a potentially large value

- **Get_deleted_vclock** of *bool*
  
	Return the tombstone's vclock, if applicable
 
### Put

```
val riak_put :
  riak_connection ->
  riak_bucket ->
  riak_key option ->
  string ->
  riak_put_option list -> 
  riak_object list

val riak_put_raw :
  riak_connection ->
  riak_bucket ->
  riak_key option ->
  string ->
  riak_put_option list -> 
  riak_vclock option -> 
  riak_object list
```
If you plan on inserting new key/values, use riak_put_raw. If you aren't sure if your key/value is new, use riak_put. riak_put will try and fetch the vclock before updating to limit sibling explosion

**Example**



```
 let newkey = "foo" in
 let newval = "bar" in
 riak_put conn bucket (Some newkey) newval [Put_return_body true]
```

```
 let newkey = "foo" in
 let newval = "bar" in
 let existing_vclock = (*Some vclock *)
 riak_put_raw conn bucket (Some newkey) newval [Put_return_body true] existing_vclock
```

type riak_put_option =

- **Put_w** of *riak_tunable_cap*

    Write quorum. How many replicas to write to before returning a successful response. Default is defined per bucket. See the Tunable CAP Options section below.
    
- **Put_dw** of *riak_tunable_cap*

	How many replicas to commit to durable storage before returning a successful response. Default is defined per bucket. See the Tunable CAP Options section below.

- **Put_return_body** of *bool*

	 Whether to return the contents of the stored object. Defaults to false.

- **Put_pw** of *riak_tunable_cap*

	How many primary nodes must be up when the write is attempted. Default is defined per bucket. See the Tunable CAP Options section below.
	
- **Put_if_not_modified** of *bool*

	Update the value only if the vclock in the supplied object matches the one in the database.

- **Put_if_none_match** of *bool*

	Store the value only if this bucket/key combination are not already defined.

- **Put_return_head** of *bool*

	Like *return_body" except that the value(s) in the object are blank to avoid returning potentially large value(s).

### Delete

```
val riak_del :
  riak_connection ->
  riak_bucket ->
  riak_key -> riak_del_option list -> unit
```

**Example**

```
	riak_del conn bucket "del_test" [] 
```

type riak_del_option =

- **Del_rw** of *riak_tunable_cap*

	How many replicas to delete before returning a successful response. Default is defined per bucket. See the Tunable CAP Options section below.

- **Del_vclock** of *string*

	Opaque vector clock provided by an earlier Get request. Use to prevent deleting of objects that have been modified since the last get request.
 
- **Del_r** of *riak_tunable_cap*

	Read quorum. How many replicas need to agree when retrieving the object. Default is defined per bucket. See the Tunable CAP Options section below.
 
- **Del_w** of *riak_tunable_cap*

	Write quorum. How many replicas to write to before returning a successful response. Default is defined per bucket. See the Tunable CAP Options section below.
 
- **Del_pr** of *riak_tunable_cap*

	Primary read quorum. How many primary replicas need to be available when retrieving the object. Default is defined per bucket. See the Tunable CAP Options section below.
 
- **Del_pw** of *riak_tunable_cap*

	How many primary nodes must be up when the write is attempted. Default is defined per bucket. See the Tunable CAP Options section below.
 
- **Del_dw** of *riak_tunable_cap*

	How many replicas to commit to durable storage before returning a successful response. Default is defined per bucket. See the Tunable CAP Options section below.



### Tunable CAP Options

 type riak_tunable_cap =

- **Riak_value_one**
- **Riak_value_quorum**
- **Riak_value_all**
- **Riak_value_default**
- **Riak_value of Riak_kv_piqi.uint32**


### List Buckets

```
val riak_list_buckets : riak_connection -> riak_bucket list
```

**Example**

```
 let buckets = riak_list_buckets conn in
```

### List Keys

```
val riak_list_keys : riak_connection -> riak_bucket -> riak_key list
```

**Example**

```
 let keys = riak_list_keys conn "mybucket" in
```

### Get Bucket Props (limited)
At the moment, Riak Protobuffs only implement 2 bucket properties, 

  * n_val
  * allow_mult
  
  
```
val riak_get_bucket : 
	riak_connection -> 
	riak_bucket -> 
	int32 option * bool option
```

**Example**

```
 let (n, multi) = riak_get_bucket conn bucket in
      (match n with
        | Some nval -> assert_bool "Valid bucket n value" (nval > 0l)
        | None -> assert_failure "Unexpected default N value");
      (match multi with
        | Some multival -> assert_equal false multival
        | None -> assert_failure "Unexpected default multi value")
```

### Set Bucket Props (limited)
At the moment, Riak Protobuffs only implement 2 bucket properties, 

  * n_val
  * allow_mult

```
val riak_set_bucket : 
    riak_connection -> 
    riak_bucket -> 
    int32 option -> 
    bool option -> 
    unit
```

**Example**

```
  let n_val = 2l in
  let allow_mult = (Some true) in
  riak_set_bucket conn bucket n_val allow_mult
```

### Map/Reduce

```
val riak_mapred :
  riak_connection ->
  riak_mr_query ->
  riak_mr_content_type ->
  (string option * int32 option) list

```

**Example**

See src/test.ml for an example.

### Index Query

Secondary index (2i) exact match query:

```
val riak_index_eq :
  riak_connection ->
  riak_bucket ->
  riak_2i_name ->
  riak_key option -> string list
```

Secondary index (2i) range query:

```
val riak_index_range :
  riak_connection ->
  riak_bucket ->
  riak_2i_name ->
  riak_2i_range_min option ->
  riak_2i_range_max option -> string list
```

### Riak Search

Good luck. This probably needs a little bit of cleanup. Please don't punch me Ryan.

```
val riak_search_query :
  riak_connection ->
  string ->
  string ->
  riak_search_option list ->
  (string * string option) list list *
  Riak_search_piqi.Riak_search_piqi.float32 option *
  Riak_search_piqi.Riak_search_piqi.uint32 option
```


type riak_search_option =

- **Search_rows** of *Riak_kv_piqi.uint32*

	 Specify the maximum number of results to return. Default is 10.
	 
- **Search_start** of *Riak_kv_piqi.uint32*

	 Specify the starting result of the query. Useful for paging. Default is 0.
	 
- **Search_sort** of *string*

	Sort on the specified field name. Default is “none”, which causes the results to be sorted in descending order by score.
	
- **Search_filter** of *string*

	Filters the search by an additional query scoped to inline fields.
	
- **Search_df** of *string*

	Use the provided field as the default. Overrides the “default_field” setting in the schema file.
	
- **Search_op** of *string*

	Allowed settings are either “and” or “or”. Overrides the “default_op” setting in the schema file. Default is “or”.
	
- **Search_fl** of *string list*

	Return fields limit (for ids only, generally).

- **Search_presort** of *string*

	Presort (key / score)

---

## Contributing

* Please report all bugs and feature requests via Github Issues.
* Friendly pull requests accepted. Please create a new branch for your features (checkout -b my_branch).

##TODO
    * test search, index
    * better error handling
	* Next version: support HTTP operations, or better yet, implement all HTTP ops as PB messages in Riak

** © 2012 Dave Parfitt **

Portions of the documentation are ** © 2012 Basho Technologies **

