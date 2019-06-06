.. _doc-extensions:

Extensions
==========

Extensions are a way to allow you to incorporate custom code into you GenieACS
provisioning process. An extension is a custom piece of Javascript which can be
called from your provisioning script.

The script must be created in the <genieacs home>/config/ext/ directory.
(you'll have to create the /ext/ directory)

The script should consist of one or more functions, and exit using the provided
callback. For example;

.. code:: javascript

  const logger = require('../../lib/logger');

  function getCreds(args, callback) {
    let serial = args[0];
    //Logger functions are: info, warn, error
    logger.info({message: 'Getting mgmt credentials', serial: serial}); //message is the only required argument
  
    let result = {
      username: "cpe",
      password: serial
    };
  
    callback(null, result);
  }
  
  exports.getManagementCreds = getCreds;

The first parameter on the callback is an error value. Set this to something
sensible to return an error to the provision script.

You can call other functions or load other scripts from the same directory to
perform complex functions. Any modules you install into GenieACS can be used
from the extension script. Use `require` to use installed modules.

To call the above script, create a provision script which contains the
instruction:

.. code:: javascript

  ext("fileName", "functionName", "arg1", ..."argn");

Example provision script that calls an external module:

.. code:: javascript

  let serial = declare("DeviceID.SerialNumber", {value: 1}).value[0];
  let response = ext("creds", "getManagementCreds", serial);

  declare("InternetGatewayDevice.ManagementServer.Username", {value: 1}, {value: response.username});
  declare("InternetGatewayDevice.ManagementServer.Password", {value: 1}, {value: response.password});
