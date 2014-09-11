## Setting up a development environment for NIMS 2.0

We are using Ubuntu 14.04 (fresh install)

###Install the required packages:

```sh
sudo apt-get install mongodb python-virtualenv python-dev
sudo apt-get install git
```

###Checkout the NIMS:
```sh
git clone https://github.com/scitran/nims.git
cd nims
```

You will see five submodules that are empty for now. (internims, nimsapi, nimsdata, nimsproc, and nimsweb)
To checkout the submodules you can use commands like "git submodule init" and "git submodule update"
However, we will keep it simple and checkout each submodule separately:

```sh
git clone https://github.com/scitran/internims.git internims
git clone https://github.com/scitran/nimsapi.git nimsapi
git clone https://github.com/scitran/nimsdata.git nimsdata
git clone https://github.com/scitran/nimsproc.git nimsproc
git clone https://github.com/scitran/nimsweb.git nimsweb
```

Let's switch to a development branch of nimsdata (this was necessary to remove runtime error)
```sh
cd nimsdata
git checkout ksh-nimsdata2
cd ..
```

###Set up python

Start a python virtual environment:
```sh
cd $HOME
virtualenv webapp2
source webapp2/bin/activate
cd nims
```

Each time you log in you should re-run "source webapp2/bin/active" to enter the python virtual environment.

Install the required python packages:
```sh
pip install webapp2 webob paste markdown pillow pydicom requests pymongo pycrypto (note: pycrypto will not be needed in the updated version)
pip install numpy
pip install pydicom nibabel
```

Note that nibabel requires numpy to be installed first.

Install dcmstack:
```sh
wget https://github.com/moloney/dcmstack/archive/v0.6.2.tar.gz
easy_install v0.6.2.tar.gz
rm v0.6.2.tar.gz
```

Install ipython:
```sh
pip install ipython
```


###Start the NIMS API
```sh
PYTHONPATH=. nimsapi/api.py development.ini --db_uri mongodb://localhost/nims --store_path=/tmp
```

Now point your browser to http://localhost:8080/nimsapi

If you want to serve from a different port and/or serve to the outside world, then edit the bottom of nimsapi/api.py
```python
paste.httpserver.serve(app, host='0.0.0.0', port='8081')
```

(+++ Consider adding the port and host as command-line options?)
  
The problem is that we don't have any data in the database.
To initialize some data in the database we need to run nimsapi/bootstrap.py
```sh
cd nimsapi
cp bootstrap.json.sample bootstrap.json
```

At this point you can edit bootstrap.json if needed
```sh
./bootstrap.py dbinit mongodb://localhost/nims -j bootstrap.json
```

Now we have a user in the system. So start up the web server again. Refresh the web page, and enter a user name in the edit box (you can use u1@stanford.edu). Click the button "Generate Custom Links". Notice that the URL has been appended so that the current user is hard-coded. This is okay (and very useful) for development.

Now click on some of the links (e.g., nimsapi/users) to see how the API returns JSON responses for the various queries.

For development, it is recommended that you use the chrome web browser and install the following plugin: JSONView
Now you can click on links that appear in the query results. Very cool!

Unfortunately we still don't have any actual data in the system. That will be the next step

###Parse some DICOM files

Before importing DICOM data into NIMS, it is necessary to organize the files into a tarball with a specific format which includes a metadata.json. Fortunately there is a python script to help with this.

Checkout the nimsscripts project. From within the nims directory:
```sh
git clone https://github.com/scitran/nimsscripts.git
```

Copy a directory of dicoms to some path: '/path/to/dicom/files'

(+++Note that it would be nice to provide a link to some sample dicom files)

**IMPORTANT! The next command will move (not copy) your DICOM files. Be sure to use a copy!**

Create two directories, one for sorting and one for the tarballs and run the following command:

```sh
./nimsscripts/dicomsort.py tarsort /path/to/dicom/files /path/to/sort /path/to/tarballs
(+++ Check this command!)
```

**IMPORTANT AGAIN! This command will move (not copy) your DICOM files. Be sure to use a copy!**

This will create a number of tarballs in the /path/to/tarballs directory. There will be one .tgz file for each acquisition. Note that dicomsort.py will search the DICOM directory recursively, and it does not matter how the DICOM files are organized or named. Each tarball will contain a metadata.json file describing how the acquisition should be sorted during parsing (next step).

Let's check to see if things were sorted properly. Start ipython after setting the appropriate path:
```sh
PYTHONPATH=. ipython
```

You should be in the nims directory for this to work properly.

Run the following commands:

```python
import nimsdata
ds=nimsdata.parse('/path/to/a/tgz/file/from/previous/step')
print ds.nims_type
print 'project = '+ds.project
print 'group = '+ds.group
print 'subject = '+ds.nims_session_subject
print 'session = '+ds.nims_session_label
print 'acquisition label = '+ds.nims_epoch_label
print 'acquisition description = '+ds.nims_epoch_description
print 'acquisition type = '+ds.nims_epoch_type
```

This parses the dataset into the variable "ds" and prints a few relevant fields. You can inspect the other fields of "ds" by typing "ds." and then pressing \[TAB\].

(+++Talk about the default values for project and subject, and describe how to override these values in the dicomsort.py. Consider adding some command-line options to dicomsort allowing overriding these values without changing the script... or creating a config file to handle this)

(+++ Note that overwriting session_label requires a patch of nimsmrdata.py, so make sure that patch has been added to the repository)


###Import DICOM data into the NIMS database

Before importing, we need to add a new user and a new group to the database. From ipython:

```python
import pymongo
db = pymongo.MongoClient('mongodb://localhost/nims', tz_aware=True).get_default_database()
db.users.insert({'_id': 'user@email.address', 'email': 'user@email.address', 'firstname': 'John', 'lastname': 'Doe', 'superuser': True})
db.groups.insert({'_id': 'username', 'roles': [{u'access': u'read-write', u'share': True, u'uid': u'user@email.address'}]})
```

Now restart the web server and visit the web page (as above) to browse around the new user and new group. Note that you will need to login as 'user@email.address'.

Next we will import some DICOM data. Create a new directory 'path/to/DATA' where all imported data will reside, change to the nims directory, and issue:

```sh
PYTHONPATH=. nimsapi/bootstrap.py sort -q mongodb://localhost/nims /path/to/tarballs /path/to/DATA
```

**Note that this will move the tarballs rather than make a copy.**

Now return to the web page and browse around your new data!

