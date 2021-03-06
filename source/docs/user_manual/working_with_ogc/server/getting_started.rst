.. only:: html

   |updatedisclaimer|

***************
Getting Started
***************

.. only:: html

   .. contents::
      :local:
      :depth: 2

Installation
============

.. index:: Debian, Ubuntu

At this point, we will give a short and simple installation how-to for
a minimal working configuration on Debian based systems. However, many other
distributions and OSs provide packages for QGIS Server.

Requirements and steps to install QGIS Server on a Debian based system are
provided in `QGIS installers page <https://qgis.org/en/site/forusers/alldownloads.html>`_.
Please refer to that section.


.. _`httpserver`:

HTTP Server configuration
=========================

.. index:: Apache

Apache
------

Configuration
^^^^^^^^^^^^^

Once QGIS Server is installed, let's configure the environment and enable it.

Install the Apache server in a separate virtual host listening on port ``80``.
Enable the rewrite module to pass HTTP BASIC auth headers:

.. code-block:: bash

 sudo a2enmod rewrite
 cat /etc/apache2/conf-available/qgis-server-port.conf
 Listen 80
 sudo a2enconf qgis-server-port

This is the virtual host configuration, stored in
:file:`/etc/apache2/sites-available/001-qgis-server.conf`:

.. code-block:: apache

   <VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/qgis-server-error.log
    CustomLog ${APACHE_LOG_DIR}/qgis-server-access.log combined

    # Longer timeout for WPS... default = 40
    FcgidIOTimeout 120
    FcgidInitialEnv LC_ALL "en_US.UTF-8"
    FcgidInitialEnv PYTHONIOENCODING UTF-8
    FcgidInitialEnv LANG "en_US.UTF-8"

    ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
    <Directory "/usr/lib/cgi-bin">
        AllowOverride All
        Options +ExecCGI -MultiViews +FollowSymLinks
	# for apache2 > 2.4
	Require all granted
        #Allow from all
    </Directory>
   </VirtualHost>


Moreover, you can use some :ref:`qgis-server-envvar` to configure QGIS Server.
With Apache as HTTP Server, you have to use ``FcgidInitialEnv`` to define these
variables as shown below:

.. code-block:: apache

    FcgidInitialEnv QGIS_DEBUG 1
    FcgidInitialEnv QGIS_SERVER_LOG_FILE /tmp/qgis-000.log
    FcgidInitialEnv QGIS_SERVER_LOG_LEVEL 0


Start
^^^^^

Now enable the virtual host and restart Apache:

.. code-block:: bash

 sudo a2ensite 001-qgis-server
 sudo service apache2 restart

QGIS Server is now available at http://localhost/cgi-bin/qgis-server.cgi.


.. index:: nginx, spawn-fcgi, fcgiwrap

NGINX
-----

You can also use QGIS Server with `NGINX <http://nginx.org/>`_. Unlike Apache,
NGINX does not automatically spawn a FastCGI process. Actually, you have to use
another component to start these processes.

To do that on Debian based systems, you may use **fcgiwrap** or **spawn-fcgi**
based on your preferences to run QGIS Server. In both case, you have to install
NGINX:

.. code-block:: bash

 sudo apt-get install nginx


fcgiwrap
^^^^^^^^

If you want to use fcgiwrap to run QGIS Server, you firstly have to install
the corresponding package:

.. code-block:: bash

 sudo apt-get install fcgiwrap

Then, introduce the following block in your NGINX server configuration:

.. code-block:: nginx
   :linenos:

     location /qgisserver {
         gzip           off;
         include        fastcgi_params;
         fastcgi_pass   unix:/var/run/fcgiwrap.socket;
         fastcgi_param  SCRIPT_FILENAME /usr/lib/cgi-bin/qgis_mapserv.fcgi;
     }

Finally, restart NGINX and fcgiwrap to take into account the new configuration:

.. code-block:: bash

 sudo service nginx restart
 sudo service fcgiwrap restart

QGIS Server is now available at http://localhost/qgisserver.

spawn-fcgi
^^^^^^^^^^

If you prefer to use spawn-fcgi instead of fcgiwrap, the first step is to
install the package:

.. code-block:: bash

  sudo apt-get install spawn-fcgi


Then, introduce the following block in your NGINX server configuration:

.. code-block:: nginx

     location /qgisserver {
         gzip           off;
         include        fastcgi_params;
         fastcgi_pass   unix:/var/run/qgisserver.socket;
     }

And restart NGINX to take into account the new configuration:

.. code-block:: bash

 sudo service nginx restart

Finally, considering that there is no default service file for spawn-fcgi, you
have to manually start QGIS Server in your terminal:

.. code-block:: bash

 sudo spawn-fcgi -f /usr/lib/bin/cgi-bin/qgis_mapserv.fcgi \
                 -s /var/run/qgisserver.socket \
                 -U www-data -G www-data -n

Of course, you may write an init script (like a ``qgisserver.service`` file
with Systemd) to start QGIS Server at boot time or whenever you want.

QGIS Server is now available at http://localhost/qgisserver.

Configuration
^^^^^^^^^^^^^

The **include fastcgi_params;** used in previous configuration is important
as it adds the parameters from ``/etc/nginx/fastcgi_params``:

.. code-block:: nginx

 fastcgi_param  QUERY_STRING       $query_string;
 fastcgi_param  REQUEST_METHOD     $request_method;
 fastcgi_param  CONTENT_TYPE       $content_type;
 fastcgi_param  CONTENT_LENGTH     $content_length;

 fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
 fastcgi_param  REQUEST_URI        $request_uri;
 fastcgi_param  DOCUMENT_URI       $document_uri;
 fastcgi_param  DOCUMENT_ROOT      $document_root;
 fastcgi_param  SERVER_PROTOCOL    $server_protocol;
 fastcgi_param  REQUEST_SCHEME     $scheme;
 fastcgi_param  HTTPS              $https if_not_empty;

 fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
 fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

 fastcgi_param  REMOTE_ADDR        $remote_addr;
 fastcgi_param  REMOTE_PORT        $remote_port;
 fastcgi_param  SERVER_ADDR        $server_addr;
 fastcgi_param  SERVER_PORT        $server_port;
 fastcgi_param  SERVER_NAME        $server_name;

 # PHP only, required if PHP was built with --enable-force-cgi-redirect
 fastcgi_param  REDIRECT_STATUS    200;


Of course, you may override these variables in your own configuration. For
example:

.. code-block:: nginx

    include fastcgi_params;
    fastcgi_param SERVER_NAME domain.name.eu;


Moreover, you can use some :ref:`qgis-server-envvar` to configure QGIS Server.
With NGINX as HTTP Server, you have to use ``fastcgi_param`` to define these
variables as shown below:

.. code-block:: nginx

    fastcgi_param  QGIS_DEBUG              1;
    fastcgi_param  QGIS_SERVER_LOG_FILE    /tmp/qgis-000.log;
    fastcgi_param  QGIS_SERVER_LOG_LEVEL   0;

.. note::

    When using spawn-fcgi, you may directly define environment variables
    before running the server. For example:
    ``export QGIS_SERVER_LOG_FILE=/home/user/qgis.log``


Xvfb
====

QGIS Server needs a running X Server to be fully usable. But if you don't have
one, you may use xvfb to have a virtual X environment.

To install the package:

.. code-block:: bash

 sudo apt-get install xvfb

Then, according to your HTTP server, you should configure the **DISPLAY**
parameter or directly use **xvfb-run**.

For example with NGINX and spawn-fcgi using xvfb-run:

.. code-block:: bash

 xvfb-run /usr/bin/spawn-fcgi -f /usr/lib/bin/cgi-bin/qgis_mapserv.fcgi \
                              -s /tmp/qgisserver.socket \
                              -G www-data -U www-data -n

The other option is to start a virtual X server environment with a specific
display number thanks to **Xvfb**:

.. code-block:: bash

 /usr/bin/Xvfb :99 -screen 0 1024x768x24 -ac +extension GLX +render -noreset

Then we just have to set the **DISPLAY** environment variable in the HTTP server
configuration. For example with NGINX:

.. code-block:: nginx

 fastcgi_param  DISPLAY       ":99";

Or with Apache:

.. code-block:: apache

 FcgidInitialEnv DISPLAY       ":99"



Serve a project
===============

Now that QGIS Server is installed and running, we just have to use it.

Obviously, we need a QGIS project to work on. Of course, you can fully
customize your project by defining contact information, precise some
restrictions on CRS or even exclude some layers. Everything you need to know
about that is described later in :ref:`Creatingwmsfromproject`.

But for now, we are going to use a simple project already configured. To
retrieve the project:

.. code-block:: bash

 cd /home/user/
 wget https://github.com/qgis/QGIS-Training-Data/archive/master.zip -O qgis-server-tutorial.zip
 unzip qgis-server-tutorial.zip
 mv QGIS-Training-Data-master/training_manual_data/qgis-server-tutorial-data ~

The project file is ``qgis-server-tutorial-data-master/world.qgs``. Of course,
you can use your favorite GIS software to open this file and take a look on the
configuration and available layers.

By opening the project and taking a quick look on layers, we know that 4
layers are currently available:

- airports
- places
- countries
- countries_shapeburst

You don't have to understand the full request for now but you may retrieve
a map with some of the previous layers thanks to QGIS Server by doing something
like this in your web browser to retrieve the *countries* layer:

.. code-block:: bash

  http://localhost/qgisserver?
    MAP=/home/user/qgis-server-tutorial-data-master/world.qgs&
    LAYERS=countries&
    SERVICE=WMS&
    REQUEST=GetMap&
    CRS=EPSG:4326&
    WIDTH=400&
    HEIGHT=200

If you obtain the next image, then QGIS Server is running correctly:

.. figure:: /static/user_manual/working_with_ogc/server_basic_getmap.png
  :align: center

  Server response to a basic GetMap request

Note that you may define **PROJECT_FILE** environment variable to use a project
by default instead of giving a **MAP** parameter (see :ref:`qgis-server-envvar`).

For example with spawn-fcgi:

.. code-block:: bash

 export PROJECT_FILE=/home/user/qgis-server-tutorial-data-master/world.qgs
 spawn-fcgi -f /usr/lib/bin/cgi-bin/qgis_mapserv.fcgi \
            -s /var/run/qgisserver.socket \
            -U www-data -G www-data -n



.. _`Creatingwmsfromproject`:

Configure your project
======================

To provide a new QGIS Server WMS, WFS or WCS, you have to create a QGIS project
file with some data or use one of your current project. Define the colors and
styles of the layers in QGIS and the project CRS, if not already defined.

.. _figure_server_definitions:

.. figure:: /static/user_manual/working_with_ogc/ows_server_definition.png
   :align: center

   Definitions for a QGIS Server WMS/WFS/WCS project

Then, go to the :guilabel:`QGIS Server` menu of the
:menuselection:`Project --> Project Properties` dialog and provide
some information about the OWS in the fields under
:guilabel:`Service Capabilities`.
This will appear in the GetCapabilities response of the WMS, WFS or WCS.
If you don't check |checkbox| :guilabel:`Service capabilities`,
QGIS Server will use the information given in the :file:`wms_metadata.xml` file
located in the :file:`cgi-bin` folder.

.. warning::

 If you're using the QGIS project with styling based on SVG files using
 relative paths then you should know that the server considers the path
 relative to its :file:`qgis_mapserv.fcgi` file (not to the :file:`qgs` file).
 So, if you deploy a project on the server and the SVG files are not placed
 accordingly, the output images may not respect the Desktop styling.
 To ensure this doesn't happen, you can simply copy the SVG files relative
 to the :file:`qgis_mapserv.fcgi`. You can also create a symbolic link in the
 directory where the fcgi file resides that points to the directory containing
 the SVG files (on Linux/Unix).

WMS capabilities
----------------

In the :guilabel:`WMS capabilities` section, you can define
the extent advertised in the WMS GetCapabilities response by entering
the minimum and maximum X and Y values in the fields under
:guilabel:`Advertised extent`.
Clicking :guilabel:`Use Current Canvas Extent` sets these values to the
extent currently displayed in the QGIS map canvas.
By checking |checkbox| :guilabel:`CRS restrictions`, you can restrict
in which coordinate reference systems (CRS) QGIS Server will offer
to render maps.
Use the |signPlus| button below to select those CRSs
from the Coordinate Reference System Selector, or click :guilabel:`Used`
to add the CRSs used in the QGIS project to the list.

If you have print composers defined in your project, they will be listed in the
`GetProjectSettings` response, and they can be used by the GetPrint request to
create prints, using one of the print composer layouts as a template.
This is a QGIS-specific extension to the WMS 1.3.0 specification.
If you want to exclude any print composer from being published by the WMS,
check |checkbox| :guilabel:`Exclude composers` and click the
|signPlus| button below.
Then, select a print composer from the :guilabel:`Select print composer` dialog
in order to add it to the excluded composers list.

If you want to exclude any layer or layer group from being published by the
WMS, check |checkbox| :guilabel:`Exclude Layers` and click the
|signPlus| button below.
This opens the :guilabel:`Select restricted layers and groups` dialog, which
allows you to choose the layers and groups that you don't want to be published.
Use the :kbd:`Shift` or :kbd:`Ctrl` key if you want to select multiple entries.

You can receive requested GetFeatureInfo as plain text, XML and GML. Default is XML,
text or GML format depends the output format chosen for the GetFeatureInfo request.

If you wish, you can check |checkbox| :guilabel:`Add geometry to feature response`.
This will include in the GetFeatureInfo response the geometries of the
features in a text format. If you want QGIS Server to advertise specific request URLs
in the WMS GetCapabilities response, enter the corresponding URL in the
:guilabel:`Advertised URL` field.
Furthermore, you can restrict the maximum size of the maps returned by the
GetMap request by entering the maximum width and height into the respective
fields under :guilabel:`Maximums for GetMap request`.

If one of your layers uses the :ref:`Map Tip display <maptips>` (i.e. to show text using
expressions) this will be listed inside the GetFeatureInfo output. If the
layer uses a Value Map for one of its attributes, this information will also
be shown in the GetFeatureInfo output.

WFS capabilities
----------------

In the :guilabel:`WFS capabilities` area you can select the layers you
want to publish as WFS, and specify if they will allow update, insert and
delete operations.
If you enter a URL in the :guilabel:`Advertised URL` field of the
:guilabel:`WFS capabilities` section, QGIS Server will advertise this specific
URL in the WFS GetCapabilities response.

WCS capabilities
----------------

In the :guilabel:`WCS capabilities` area, you can select the layers that you
want to publish as WCS. If you enter a URL in the :guilabel:`Advertised URL`
field of the :guilabel:`WCS capabilities` section, QGIS Server will advertise
this specific URL in the WCS GetCapabilities response.

Fine tuning your OWS
----------------------

For vector layers, the :guilabel:`Fields` menu of the
:menuselection:`Layer --> Properties` dialog allows you to define for each
attribute if it will be published or not.
By default, all the attributes are published by your WMS and WFS.
If you don't want a specific attribute to be published, uncheck the corresponding
checkbox in the :guilabel:`WMS` or :guilabel:`WFS` column.

You can overlay watermarks over the maps produced by your WMS by adding text
annotations or SVG annotations to the project file.
See the :ref:`sec_annotations` section for instructions on
creating annotations. For annotations to be displayed as watermarks on the WMS
output, the :guilabel:`Fixed map position` checkbox in the
:guilabel:`Annotation text` dialog must be unchecked.
This can be accessed by double clicking the annotation while one of the
annotation tools is active.
For SVG annotations, you will need either to set the project to save absolute
paths (in the :guilabel:`General` menu of the
:menuselection:`Project --> Project Properties` dialog) or to manually modify
the path to the SVG image so that it represents a valid relative path.
