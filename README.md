# Messages
This package defines and build off-the-shelf messages for DLS2 and their related wrappers. It also provides you the cmake function that can be used by the user to generate custom messages and wrappers.

# Message generation
Each message is defined by an idl file inside the [idls](idls) folder. See [here](https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/dataTypes/dataTypes.html) for details about how to define a message in IDL format.

To generate messages for fastdds, the [fastddgen](https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/introduction/introduction.html) tool is used (check the cmake function _dls_add_message_ inside [dls_message.cmake](cmake/dls_message.cmake) file).

# dls_std_msgs
This repository contains the standard messages(.idl files) used by DLS2.

# ROS2 compatibility
For ROS2 communication, message definitions should comply with the following conventions:

- each message uses a CamelCase name, i.e. `MyMessage`, **not** `my_message`
- the file name follows the same convention: `MyMessage.idl`
- the message should be wrapped in the modules `dls2_interface` and `msg`, i.e.
```
module dls2_interface
{
  module msg
  {
    struct MyMessage
    {
      ...
    }
  }
}
```
