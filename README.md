# dls_std_msgs
This repository contains the standard messages(.idl files) used by DLS2.

### ROS2 compatibility
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
