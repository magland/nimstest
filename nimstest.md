## Setting up a development environment for NIMS 2.0


We are using Ubuntu 14.04 (fresh install)

###Install the required packages:

```sh
> sudo apt-get install mongodb python-virtualenv python-dev
> sudo apt-get install git
```

###Checkout the NIMS:
```sh
> git clone https://github.com/scitran/nims.git
> cd nims
```

You will see five submodules that are empty for now. (internims, nimsapi, nimsdata, nimsproc, and nimsweb)
To checkout the submodules you can use commands like "git submodule init" and "git submodule update"
However, we will keep it simple and checkout each submodule separately:

```zh
> git clone https://github.com/scitran/internims.git internims
> git clone https://github.com/scitran/nimsapi.git nimsapi
> git clone https://github.com/scitran/nimsdata.git nimsdata
> git clone https://github.com/scitran/nimsproc.git nimsproc
> git clone https://github.com/scitran/nimsweb.git nimsweb
```

Let's switch to a development branch of nimsdata (this was necessary to remove runtime error)
```sh
> cd nimsdata
> git checkout ksh-nimsdata2
> cd ..
```

###Set up python

Start a python virtual environment:
```sh
> cd $HOME  (is this step necessary?)
> virtualenv webapp2
> source webapp2/bin/activate
> cd nims
```

Install the required python packages:
```sh
> pip install webapp2 webob paste pymongo markdown numpy pillow pydicom requests pycrypto (pycrypto will not be needed in updated version)
> pip install nibabel (not sure why this needed to be installed after the others)
```

####NOTE
For some reason, I couldn't get dcmstack to install...
So I had to break the program by removing all references to dcmstack in nimsdicom.py and nimsmydata.py


###Start the NIMS API
```sh
> PYTHONPATH=. nimsapi/api.py development.ini --db_uri mongodb://localhost/nims --store_path=/tmp
```

Now point your browser to http://localhost:8080/nimsapi

If you want to serve from a different port and/or serve to the outside world, then edit the bottom of nimsapi/api.py
```python
paste.httpserver.serve(app, host='0.0.0.0', port='8081')
```
  
The problem is that we don't have any data in the database.
To initialize some data in the database we need to run nimsapi/bootstrap.py
```sh
> cd nimsapi
> cp bootstrap.json.sample bootstrap.json ## Note, I recommend putting this sample file into the repository
```

At this point you can edit bootstrap.json if needed
```sh
> ./bootstrap.py dbinit mongodb://localhost/nims -j bootstrap.json
```

Now, at least we have a user in the system. So start up the web server again. Refresh the web page, and enter a user name in the edit box (you can use u1@stanford.edu). Click the button "Generate Custom Links". Notice that the URL has been appended so that the current user is hard-coded. This is okay (and very useful) for development.

Now click on some of the links (e.g., nimsapi/users) to see how the API returns json responses for the various queries.

For development, it is recommended that you use the chrome web browser and install the following plugin: JSONView
Now you can click on links that appear in the query results. Very cool!

Unfortunately we still don't have any actual data in the system. That will be the next step
  
