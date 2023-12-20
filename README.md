usage: 

TODO:

- [ ] write the data storage script for saving the received CAN messages locally in the mcap 
- [x] make service script that creates an instance of the mcap writer and the foxglove websocket
- [ ] the deserialization task for unpacking received data.
- [x] come up with a good way of associating the dbc file with the protobuf file

    - I will simply match the CAN frame id name to the message name, and match each signal name (spaces into underscores) to the field name in the proto. The protobuf message will be packed with the parsed and converted data from cantools.
    
    - notes:
        - I want each CAN ID to have its own protobuf message. perhaps in the protobuf message I will also include the CAN ID as a fixed part of the protobuf message in the creation of the proto file.
       
        - I know that I will be using cantools to create the DBC file so I might as well extend that creation script to create the proto at the same time. Additionally, I know that I will be using tim's auto-magic nix-proto for creation of the python auto-gen code.

- [ ] actually get current data from car into protobuf encoded CAN messages and send them from current TCU / SAB
- [ ] get the raspberry pi listening to CAN messages

## automation goals
- [ ] dbc and proto file generation using CI
- [ ] proto python lib using CI and the proto file that was generated from CI (using tim's nix flake)
- [ ] binary proto schema generation from the generated proto file using an addition to tim's nix flake
- [ ] CI generated platformio c/c++ library from DBC

## automation requirements:
- [x] nix flake packaging of all non existing packages
- [ ] nixification of data_acq
    - [x] package foxglove mcap support / other foxglove python stuff for nix
    - [ ] creation of executable for setup.py so that it is something that can be run in the flake

- [ ] nixification of the dbc and proto file generator module
    - im thinking that the dbc file gets stored in the repo for this as well (?)
- [ ] creation of at least 3 CI jobs

## high level overview
input: 
- protobuf stream (will be from CAN, this prototype will be from a port)

output: 
- saved files at time steps
- encoded websocket stream of data
- desired behavior for the data flow:
    - on hardware receive in the data_handler script data gets pushed into a container triggers both the webserver and the data writer to use that data
    - once both the data writer and the foxglove websocket have finished processing the data delete the data from the container
- a desired workflow is that it all we need to do to add a new input that we will be seeing over the wire is to add a .proto to a specific folder. No code changes should be required.

```mermaid
flowchart TD
    CAN[RPI CAN] --> py_async_q[encoded CAN data]
    py_async_q --> des[DBC based CAN parser] 
    des --> pb_pack[protobuf packet creation]
    
    pb_pack --> data_q1[webserver protobuf packet queue]
    pb_pack --> data_q2[MCAP file writer protobuf packet queue]
    subgraph websocket thread
        data_q1 --> enc[serialize into protobuf packet]
        enc --> py_foxglove[foxglove server websocket]
    end
    subgraph file writer thread
        data_q2 --> py_mcap[MCAP file writer]
    end
```

### notes:
- filter journalctl based on service: `journalctl -u nginx.service`
