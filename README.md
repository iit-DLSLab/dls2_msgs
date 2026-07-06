# DLS2 messages
Package defining and building off-the-shelf messages for DLS2.


# Message generation
Each message is defined by an idl file inside the [idls](idls) folder. See [here](https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/dataTypes/dataTypes.html) for details about how to define a message in IDL format.

To generate messages for fastdds, the [fastddgen](https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/introduction/introduction.html) tool is used (check the cmake function _generate_msg_library_ inside [generate_msg_library.cmake](cmake/generate_msg_library.cmake) file).

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
