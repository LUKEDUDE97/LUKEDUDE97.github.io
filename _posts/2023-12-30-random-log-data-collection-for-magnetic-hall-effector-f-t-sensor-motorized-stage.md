---
layout: post
title: Data Collection for Magnetic Hall Effector & F/T Sensor & Motorized
  Stage
date: 2023-12-30 11:39 +0100

categories: [Eins-Project, Engineering Log]
tags: [Whisker, F/T Sensor, Motorized Stage]

math: true
mermaid: true

post_description: Set up three Ros node to collect motorized position, magnetic vector and torque readings in synchronize, moreover format the data for later state estimation.
---

## **Data sampling software**

![Fig 1. Basic structure of ros nodes for the calibration stage](/localdata/assets/EinsProject/DataSamplingSoftware.png)
_Fig 1. Basic structure of ros nodes for the calibration stage_

Direct to my [**Git Repository**](https://github.com/LUKEDUDE97/whisker_calibrationstage_ws)

## **tle493dw2b6nano_processor (.ino)**

### build rosserial_arduino library
    
Install `rosserial` on the ROS workstation and install library into Arduino environment.
    
```bash
sudo apt-get install ros-${ROS_DISTRO}-rosserial-arduino
sudo apt-get install ros-${ROS_DISTRO}-rosserial

cd <sketchbook>/libraries  # set in your arduino ide preferance
rm -rf ros_lib
rosrun rosserial_arduino make_libraries.py .
```
    
If you have any your own custom ros message packages which need to add into the built library, you should make sure the package path was sourced and in you ros package path (check by `echo $ROS_PACKAGE_PATH` or `env | grep ROS`). After that:

```bash
rosrun rosserial_client make_library.py path_to_libraries your_message_package
```

Sometimes, it doesn’t work. It is wired the message path can’t be found from time to time. If so, you can also remove the `ros_list` and reinstall it. Make sure the path was included in the ros environment variable, and it will automatically build it for you. 

If there’re some of your message packages you not actually want the rosserial to build for you automatically, you can just erase the its path from the ros, by `unset ROS_PACKAGE_PATH && export ROS_PACKAGE_PATH=””` , notice that, delete the source action in `~/.bashrc` won’t help on that. 

>💡 BTW: Sourcing `~/.bashrc` does not magically reset all settings. The file is just a bunch of commands to be executed. If they are executed in a clean shell then you will get what you expect. But if the alias has already been defined, the lack of it in the file will not unalias it. Think about it: if you executed the commands from the file by hand, none would affect the already defined alias. There is no unalias reboot in the file. Sourcing `~/.bashrc` is exactly executing the commands, only not by hand. Do not treat `.bashrc` as a file that holds current settings for the shell. It is meant to be sourced once in a clean shell, automatically. People sometimes source it again manually and it works* if they add something, because the added thing gets executed. A removed thing cannot be executed or magically reverted. Start a new shell and let `.bashrc` do its job in the way it's designed for.
{: .prompt-info }
    
### regulate the publish rate & nodehandle buffer size & msg setting
    
Use `ls /dev/tty*` to scan the current serial port, and start the serial node by `rosrun rosserial_python serial_node.py <your connected port>` .

```bash
[INFO] [1698671358.684762]: ROS Serial Python Node
[INFO] [1698671358.690347]: Connecting to /dev/ttyUSB0 at 57600 baud
[INFO] [1698671361.117395]: Requesting topics...
[WARN] [WallTime: 1436540980.150618] Serial Port read failure:
[WARN] [WallTime: 1436540983.389940] Serial Port read returned short (expected 99 bytes, received 84 instead).
```

If the topic can’t be read from serial, after the connection has been established, it is probably because your refresh rate is too high or the buffer maximum has been surpassed. You can either, decrease the publisher rate by adjust the `rate` here:

```cpp
if (millis() > publish_timer)
    {

        // Request readings from sensor
        Tle493dMagnetic3DSensor1.updateData();
        delay(10);

        // Update data and format the message 
        FieldVector_msg.magnetic_x = Tle493dMagnetic3DSensor1.getX();
        ...
        
        FieldVector_msg.header.stamp = nh.now();
        pub.publish(&FieldVector_msg);

        publish_timer = millis() + rate; // publish it half a second
    }

    nh.spinOnce();
```

or increase the buffer for `ros::nodehandler` in `ros.h` :

```cpp
typedef NodeHandle_<ArduinoHardware> NodeHandle; // default 25, 25, 512, 512

// adjust to or whatever: 
// typedef ros::NodeHandle_<ArduinoHardware, 1, 1, 1024, 1024> nh;
```

But neither of them help on my case, therefore I recreate a message package for my self, which only contains a `Header` and three `float32`. It is totally enough for my case and won’t surpass the buffer size. 
    

## **cnc_interface (.py)**

```yaml
<param name="baudrate"     type="double" value="115200" />
<param name="port"         type="string" value="/dev/ttyUSB1"/>
```

`ls /dev/tty*` to check the serial port and `sudo chmod 0777 <postname>` to authorize it. 

### retrieve status data from device
    
One issue keeps showing when I trying to view the published topic message `/cnc_interface/position` after successfully launched the node. The `rostopic list` shows all the listed topic with no problem but the `rostopic echo /cnc_interface/position` has no response. In my case it is because it falls into an exception in the `getStatus()` function in my `cnc_class.py` file.

```python
__pos_pattern__ = re.compile('.Pos:(\-?\d+\.\d+),(\-?\d+\.\d+),(\-?\d+\.\d+)')

def getStatus(self):

        self.s.write("?".encode())
        
        while True:
            try: 
                status = self.s.readline()
                if status is not None:
                    try:
                        # Type of Matches : [list(tuple)] && Status : bytes -> need to decode into string
                        matches = self.__pos_pattern__.findall(status.decode()) 
                        if len(matches[0]) == 3:
                            self.pos = list(matches[0])				
                        return status
                    except IndexError:
                        print("No matches found in serial")
                else: break
            except:
                print("Report readiness but empty")
```

- `.encode()` and `.decode()` were to convert the data string into bytes for cnc device and vice versa.
- apply `re.compile` to build a expression pattern `__**pos_pattern__**` and resolve the data received from serial connection by `__**pos_pattern__**.findall()`.
- `matches` format: `[list(tuple)]`, make sure the reading is with correct length, 3, and convert it into list.

By the way, every time, when we connect to the cnc device and start it up, we should manually wake it up after that otherwise there may be no response after we send our gcode command. 

```python
def wakeup(self):
        self.s.write("\r\n\r\n".encode())
        time.sleep(2)
        self.s.flushInput()
```
    

## **whisker_cs_sampling(.cpp)**

Receive contact position data from `cnc_interface` and magnetic field vector from `MagneticSensor`, approximately synchronize different sampling sequences and publish it.

```cpp
void callback(const TwistStampedConstPtr &contact_msg, const MagneticFieldVectorConstPtr &magnetic_msg)
{
		...
}

int main(int argc, char **argv)
{
		...

		ros::init(argc, argv, "whisker_cs_sampling");

    ros::NodeHandle nh;

    pub = nh.advertise<calibration_stage_dataset>("/cs_sample_data", 10);
    message_filters::Subscriber<TwistStamped> contact_sub(
        nh, "/cnc_interface/position", 1);
    message_filters::Subscriber<MagneticFieldVector> magnetic_sub(
        nh, "/MagneticSensor", 1);

    typedef sync_policies::ApproximateTime<TwistStamped, MagneticFieldVector> MySyncPolicy;
    // ApproximateTime takes a queue size as its constructor argument, hence MySyncPolicy(10)
    Synchronizer<MySyncPolicy> sync(MySyncPolicy(10), contact_sub, magnetic_sub);
    sync.registerCallback(boost::bind(&callback, _1, _2));

    ros::spin();

		...
}
```

Message format: (it is necessary to add header into message, the `ApproximateSynchronizer` need ros timestamp to schedule)

```yaml
-> std_msgs/Header header
  -> uint32 seq
  -> time stamp
  -> string frame_id
-> float32 magnetic_x
-> float32 magnetic_y
-> float32 magnetic_z
-> geometry_msgs/Twist twist
  -> geometry_msgs/Vector3 linear
    -> float64 x
    -> float64 y
    -> float64 z
  -> geometry_msgs/Vector3 angular
    -> float64 x
    -> float64 y
    -> float64 z
```

👋🏼By the way, we didn’t configure the frame_id for different headers. It is fine when all the message headers are empty, with a same id that is to say. But the SensONE output the wrench message with a custom header frame id, `ft_sensor0_wrentch`, therefore all the other headers should be change their frame_id according to it. Otherwise the ApproximateSynchronizer won’t work properly.