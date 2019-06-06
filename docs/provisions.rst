.. _doc-provisions:

Provisions
==========

Provisions are a new feature introduced in v1.1 to solve the problem of
implementing a complex device provisioning work flow. A provision is sandboxed
Javascript code that is executed on a per device basis. Apart from a few
built-in functions, the code is standard ES6 code executed in strict mode.

Provisions are mapped to devices using presets. Note that the added performance
overhead when using provisions as opposed to simple preset configuration is
relatively small. Anything that can be done through preset configuration can be
done in a provision script. In fact, the old configuration format is still
supported mainly for backward compatibility and it is recommended to use
provisions for all configuration.

When executing a provision script through a preset, arguments to be passed to
the script can be provided and will be available in the script in the ``args``
array.

.. note::

  Provision scripts will be executed multiple times until GenieACS detects no
  more changes. This is normal behavior.

Built-in functions
------------------

declare(path, timestamps, values)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This function is for declaring parameter values to be set as well as
constraints on how recent you'd like the parameter value (or other attributes)
to have been refreshed from the device. If the timestamp provided is lower than
the timestamp of the last time the value was read from the device then this
function will return the last known value. Otherwise, it will fetch the current
value from the device and return that in order to satisfy said time
constraints.

The timestamp argument is an object where the key is the attribute name (i.e.
``value``, ``object``, ``writable``, ``path``) and the value is an integer
value representing a Unix timestamp.

The values argument is an object similar to the timestamp argument with the
exception that the property values are the parameter values to be set.

The possible attributes in timestamps and values arguments are:

- ``value``: a [value, type] pair

This attribute is not available for objects or object instances. When setting
the value, it's not necessary to use an array of [value, type] as the parameter
type is already known.

- ``writable``: boolean

The meaning of this attribute can vary depending on the type of the parameter.
In the case of regular parameter, it indicates whether or not the value is
writable. In the case of objects, it indicates whether or not it's possible to
add new object instances. In the case of object instances, it indicates whether
or not this instance can be deleted.

- ``object``: boolean

True if this is an object or object instance.

- ``path``: string

This attribute is special in that it's not a parameter attribute per se, but it
refers to the presence of parameters matching the given path. For example,
given the following wildcard path:

``InternetGatewayDevice.LANDevice.1.Hosts.Host.*.MACAddress``

Using a recent timestamp for path in ``declare()`` will result in a sync with the
device to rediscover all Host instances (``Host.*``). The path attribute can also
be used to create or delete object instances as described in the [path format]
(#path-format) section.

The return value of ``declare()`` is an iterator to access parameters that match
the given path. Each item in the iterator has the attribute 'path' in addition
to any other attribute given in the ``declare()`` call. The iterator object itself
has convenience attribute accessors which come in handy when you're expecting a
single parameter (e.g. when path does not contain wildcards or aliases).

.. code:: javascript

  // Example: Setting the SSID as the last 6 characters of the serial number
  let serial = declare("Device.DeviceInfo.SerialNumber", {value: 1});
  declare("Device.LANDevice.1.WLANConfiguration.1.SSID", null, {value: serial.value[0]});

clear(path, timestamp)
~~~~~~~~~~~~~~~~~~~~~~

This function invalidates the database copy of parameters (and their child
parameters) that match the given path and have a last refresh timestamp that is
less than the given timestamp. The most obvious use for this function is to
invalidate the database copy of the entire data model when the device is
factory reset:

.. code:: javascript

  // Example: Clear cached device data model Note
  // Make sure to apply only on "0 BOOTSTRAP" event
  clear("Device", Date.now());
  clear("InternetGatewayDevice", Date.now());

commit()
~~~~~~~~

This function commits the pending declarations and performs any necessary sync
with the device. It's usually not required to call this function as it called
implicitly at the end of the script and when accessing any property of the
promise-like object returned by the ``declare()`` function. Calling this
explicitly may be necessary for certain use case such as implementing complex
device provisioning work flow.

ext(file, function, arg1, arg2, ...)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Execute an extension script and return the result. The first argument is the
script filename and the sector argument is the function name within that
script. Any remaining arguments will be passed to that function. See
:ref:`doc-extensions` for more details.

Path format
-----------

A parameter path may contain a wildcard (``*``) or an alias filter
(``[name:value]``). A wildcard segment in a parameter path will apply the
declared configuration to zero or more parameters that match the given path
where the wildcard segment can be anything.

An alias filter is like wildcard, but additionally performs filtering on child
parameters based on the key-value pairs provided. For example, the following
path:

``Device.WANDevice.1.WANConnectionDevice.1.WANIPConnection.[AddressingType:DHCP].ExternalIPAddress``

will return a list of ExternalIPAddress parameters (0 or more) where the
sibling parameter AddressingType is set to "DHCP".

This can be useful when the exact parameter path may be different from one
device to another. It is possible to use more than one key-value pair in the
alias filter. It's also possible to use multiple filters or use a combination
of filters and wildcards.

Creating/deleting object instances
----------------------------------

Given the declarative nature of provisions, we cannot explicitly tell the
device to create or delete an instance under a given object. Instead, we
specify the number of instances we want there to be, and based on that GenieACS
will determine whether or not it needs to create or delete instances. For
example, the following declaration will ensure we have one and only one
WANIPConnection object:

.. code:: javascript

  // Example: Ensure we have one and only one WANIPConnection object
  declare("InternetGatewayDevice.WANDevice.1.WANConnectionDevice.1.WANIPConnection.*", null, {path: 1});

Note the wildcard at the end of the parameter path.

It is also possible to use alias filters as the last path segment which will
ensure the declared number of instances is satisfied given the alias filter:

.. code:: javascript

  //Ensure that *all* other instances are deleted
  declare("InternetGatewayDevice.X_BROADCOM_COM_IPAddrAccCtrl.X_BROADCOM_COM_IPAddrAccCtrlListCfg.[]", null, {path: 0});

  //Add the two entries we care about
  declare("InternetGatewayDevice.X_BROADCOM_COM_IPAddrAccCtrl.X_BROADCOM_COM_IPAddrAccCtrlListCfg.[SourceIPAddress:192.168.1.0,SourceNetMask:255.255.255.0]",  {path: now}, {path: 1});
  declare("InternetGatewayDevice.X_BROADCOM_COM_IPAddrAccCtrl.X_BROADCOM_COM_IPAddrAccCtrlListCfg.[SourceIPAddress:172.16.12.0,SourceNetMask:255.255.0.0]", {path: now}, {path: 1});

Special GenieACS parameters
---------------------------

In addition to the parameters exposed in the device's data model through
TR-069, GenieACS has its own set of special parameters:

DeviceID
~~~~~~~~

This parameter sub-tree includes the following read-only parameters:

- ``DeviceID.ID``
- ``DeviceID.SerialNumber``
- ``DeviceID.ProductClass``
- ``DeviceID.OUI``
- ``DeviceID.Manufacturer``

Tags
~~~~

The tags root parameter is used to expose device tags in the data model. Tags
appear as child parameters that are writable and have a value of [true/false,
"xsd:boolean"]. Setting a tag to false will delete that tag, and setting the
value of a non-existing tag parameter to true will create it.

.. code:: javascript

  // Example: Remove "tag1", add "tag2", and read "tag3"
  declare("Tags.tag1", null, {value: false});
  declare("Tags.tag2", null, {value: true});
  let tag3 = declare("Tags.tag3", {value: 1});

Reboot
~~~~~~

The ``Reboot`` root parameter hold the timestamp of the last reboot command.
The parameter value is writable and declaring a timestamp value that is larger
than the current value will trigger a reboot.

.. code:: javascript

  // Example: Reboot the device only if it hasn't been rebooted in the past 300 seconds
  declare("Reboot", null, {value: Date.now() - (300 * 1000)});

FactoryReset
~~~~~~~~~~~~

Works like ``Reboot`` parameter but for factory reset.

.. code:: javascript

  // Example: Default the device to factory settings
  declare("FactoryReset", null, {value: Date.now()});

Downloads
~~~~~~~~~

The Downloads sub-tree holds information about the last download command in
``Downloads.1.*`` like ``Download`` (timestamp), ``LastFileType``,
``LastFileName`` and so on. The parameters ``FileType``, ``FileName``,
``TargetFileName`` and ``Download`` are writable and can be used to trigger a
new download:

.. code:: javascript

  declare("Downloads.[FileType:1 Firmware Upgrade Image]", {path: 1}, {path: 1});
  declare("Downloads.[FileType:1 Firmware Upgrade Image].FileName", {value: 1}, {value: "firmware-2017.01.tar"});
  declare("Downloads.[FileType:1 Firmware Upgrade Image].Download", {value: 1}, {value: Date.now()});

Common file types are:

- ``1 Firmware Upgrade Image``
- ``2 Web Content``
- ``3 Vendor Configuration File``
- ``4 Tone File``
- ``5 Ringer File``

Combined with a preset that executes at ``1 BOOT``, its possible to upgrade the
firmware of a CPE when it reboots. By adding more entries to the config map
below, its to support dozens of different CPEs without having to change the
core code below.

.. code:: javascript

  let now = Date.now();
  let model = declare("InternetGatewayDevice.DeviceInfo.ModelName", {value: 1}).value[0];
  
  // Map the CPE model to the config file
  let cfgs = {"SR360n": "360_defaults.conf"};

  let lastConfigFile = declare("Downloads.[FileType:3 Vendor Configuration File].FileName", {value: Date.now()});

  let configFile = cfgs[model];

  if (!configFile) {
    //log('No config for CPE', {model: model, cfgs: cfgs});
    return;
  }

  if (lastConfigFile !== undefined && lastConfigFile.value !== undefined) {
    lastConfigFile = lastConfigFile.value[0];
  } else {
    lastConfigFile = null;
  }

  if (lastConfigFile !== configFile) {
    log('Upgrading config', {model: model, configFile: configFile});
    declare("Downloads.[FileType:3 Vendor Configuration File]", {path: 1}, {path: 1});
    declare("Downloads.[FileType:3 Vendor Configuration File].FileName", {value: 1}, {value: configFile});
    declare("Downloads.[FileType:3 Vendor Configuration File].Download", {value: 1}, {value: now});
  }

When the CPE has finished downloading the config file, it will send a ``7
TRANSFER COMPLETE`` event. Create a preset that triggers on that event which
fires off a different provision script. In this new provision script put:

.. code:: javascript

  declare("Reboot", null, {value: Date.now()});

This will cause the CPE to reboot after downloading the updated config file.