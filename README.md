# RADAR Tracking
## Summary
   
This Radar Tracking algorithm is trace the center point of object by mainly using the clustering.   
There are 2 versions of Radar Tracking.   
Radar Tracking version 1 and 2 is obtaining the position and velocity by clustering(DBSCAN in scikit-learn) and moving average filter. (DBSCAN is Density Based Spatial Clustering of Applications with Noise) ( scikit-learn is python module)  
The main difference between version 1 and 2 is order of obtaining velocity and filtering.   
*version 1: obtaining position -> obtaining velocity -> filtering*    
*version 2: obtaining position -> filtering -> obtaining velocity*        
For testing the Radar Tracking, the experiment is conducted by using IWR6843ISK and situation is  that the person go straight and back.  
<img src="https://user-images.githubusercontent.com/97038348/168714663-ffe56093-8c5d-48ff-9e46-492f182d3403.gif" width="35%" height="60%"/>

There are how to install Radar Tracking and how to run Radar Tracking on ROS.  


## Contents

1. [Radar Tracking ver.1](#1-radar-tracking-ver1)
2. [Radar Tracking ver.2](#2-radar-tracking-ver2)
3. [Experiments](#3-experiments)
4. [Running on ROS](#4-running-on-ros)
5. [Appnedices](#5-appendices)

---
---
## 1. Radar Tracking ver.1
All about Radar Tracking version 1 is in *radar_tracking/src/ti_mmwave_rospkg/src/RadarTrackingVer1.py*
### Flowchart
This Flowchart shows the algorithm of Radar Tracking version 1.   
There are 4 steps.   
Each steps are explained below.   
<p align="center"><img src="https://user-images.githubusercontent.com/97038348/170816523-d23a8f21-4e40-4c95-8e33-42e15897e0ae.jpg" width="55%" height="55%"/>

### Process diagram
   This Process diagram also shows the algorithm of Radar Tracking version 1.     
Each steps are explained below.   
<img src="https://user-images.githubusercontent.com/97038348/170435088-1b3da441-cfbe-4785-abb5-eaea77b69926.PNG" width="95%" height="95%"/>

### First step: Obtain the points from Radar (IWR6843ISK)
 The values (point id, x position, y position, time) of detected points are subscribed from the ti_mmwave_rospkg.   
 
        * Package name: ti_mmwave_rospkg  
        * Launch name: 6843_multi_3d_0.launch
        * Topic name: /ti_mmwave/radar_scan      
        * msg Type : ti_mmwave_rospkg/RadarScan

           point_id - The number of scaned points
           x - x-coordinate of the point
           y - y-coordinate of the point
           header.stamp.secs - second
           header.stamp.nsecs - nano second
 
 #### Coordinate
<img src="https://user-images.githubusercontent.com/97038348/170430559-b56a5096-7a4c-40f0-bca2-26d9a74ec548.PNG" width="50%" height="50%"/>
 
 #### Specification
* Detection range in radial axis : 175(car), 98(person) [m]

* Range resolution : 0.039 ~ 0.044 [m]

* Azimuth angle of detection : ±60 [deg]

* Elevation angle of detection : ±20 [deg]

* Angular resolution (Azimuth) : 20 [deg]

* Angular resolution (Elevation) : 58 [deg]

* Maximum radial velocity : 9.59 [m/s]

(Note: Radial velocity represents how fast object go close or far from RADAR. It does not contain transverse velocity.)
   
   #### Reference 
   https://www.ti.com/design-resources/embedded-development/industrial-mmwave-radar-sensors.html#Evaluation
 
### Second step: Clustering the points from Radar and Finding Center points
 
 Point id =0 is the criterian for separating current and previous points.      
 When the point id is 0, the getPoint (Array) is clustered.   
 The function of **'Clustering'** is detecting objects and finding the center of each object.(Explaination of Clustering is [here](#explanation-about-clustering))       
 currentTime is the average of the time of current points
 
### Thrid step: Obtain Velocity
 
 Current points and Previous points are clusterd together.   
 The function of **'ClusteringVel'** is   identifying the same objects in current points and previous points and calculating the velocity.(Explaination of ClusteringVel is [here](#explanation-about-clustering))       
    Velocity is obtaining by following equation.   
 <img src="https://user-images.githubusercontent.com/97038348/170608130-58eb76d2-9783-4bbd-bb65-29f1209de1f4.PNG" width="50%" height="50%"/>
 
The position and velocity of current center point of the cluster is assigned to the *'Data'*.
 
 
 

 
### Fourth step: Moving average Filter
 #### Window set diagram
 For making the moving average filter, **"Window set"** is defined.
<img src="https://user-images.githubusercontent.com/97038348/170440208-d659c6c7-0936-4426-b006-7d40fe9efaae.PNG" width="90%" height="90%"/>

 Window set contain k number of **"window"**.   
 the number of window is equal to the number of objects currently observed.   
 One window has n number of **"Elements"**.   
 Element is 1 x 4 matrix which contains X, Y position and X, Y veloctiy of Data.   
 When the Data (current position and current velocity) is obtained from Second and Third step, the Data is clustered with first element of each window.    
   The function of **'ClusteringFilter'** is finding the window where Data belongs.(Explaination of ClusteringFilter is [here](#explanation-about-clustering))         
   If the matched window is founded, the Data becomes the first element and the previous elements are going down one by one.     
   *Object(X,Y,Vx,Vy)* (position and velocity of Object) is obtained by applying a moving average filter to that window.    
   #### Formula of moving average filter
<img src="https://user-images.githubusercontent.com/97038348/167053294-ee42ca01-9129-4031-84b3-cccb81380603.PNG" width="80%" height="80%"/>
 
 There are three formula to calculate the average of window. Wieghted moving average and Current-Weighted moving average consider more weight to the recently observed values. You can find the value of weight by experiment.   
  
  *# You can select one fomula and weight by adjusting the PARAMETER in RadarTrackingVer1.py*
   
   #### Reference
   https://en.wikipedia.org/wiki/Moving_average
   
   If the matched window is not founded, new window is created.       
   
   Because Noise always makes new window, window that have not been updated for a long time will be deleted by counting the **"Skip number"**.    
   When window is updated, skip count becomes 0. When window is not updated, 1 is added to skip count.       
 When the skip count exceeds the maximum number of skip count, the window would be deleted.   
 
 
 ### Explanation about Clustering
 Before read this, please read [Appnedices](#5-appendices)(5.1) first.
 There are three clustering in Radar Tracking version 1. (Clustering ClusteringVel ClusteringFilter)
   
 #### 1.Clustering  
   The eps(radius) for **Clustering** should be the size of one object.    
   In the "good" case, the eps is almost same with the size of one person.   
   In the "bad" case, the eps is bigger than the size of one person. Two person can be perceived as one person.   
   <img src="https://user-images.githubusercontent.com/97038348/170825498-a3bd2ef5-1b0f-4a60-bfa8-a0e4ef3ed465.PNG" width="50%" height="50%"/>   
   
 **Clustering**(eps = objectSize, min_samples = 2)   
   objectSize = the size of one object  
   min_samples = The minimum number of detected points from object. if you increase this number, accuracy would increase.
   
  #### 2.ClusteringVel 
   The eps(radius) for **ClusteringVel** should be the size of object moving.   
   <img src="https://user-images.githubusercontent.com/97038348/170825500-d2849e49-2fa8-4656-a033-6b62b6495ad3.PNG" width="50%" height="50%"/>   
   
  **ClusteringVel**(eps = objectMovingRange, min_samples = 1)  
   objectMovingRange = The range of object moving during time difference + **"Radar Error"**    
   min_samples = 1  &nbsp;&nbsp;&nbsp;*# ClusteringVel judges whether it's the same object or not. So, 1 is enough.*    
   
 ##### "Radar Error"
 <img src="https://user-images.githubusercontent.com/97038348/170826312-092fbf4f-a64d-41fa-a1a5-6e35ef26c408.PNG" width="70%" height="70%"/>   
          
   Because radar detect the points within the object randomly, the detected center is not always equal to the real center.   
 
   #### 3.ClusteringFilter
   This is same with **ClusteringVel**.   
  **ClusteringFilter**(eps = filterRange, min_samples = 2)   
   filterRange = The range of object moving for time difference + Radar Error    
   min_samples = 2  &nbsp;&nbsp;&nbsp;*# ClusteringFilter judges whether Data and window is same or not.*    
   
  ### Parameter setting 
 
 you have to set the parameters depending on your situation.     
   For example, if you want to detect car, then objectSize would be big.    
   If you want to detect rabbit, then the objectSize would be small.   
  &nbsp;&nbsp;&nbsp;&nbsp; *# You can set these values by adjusting the PARAMETER in RadarTrackingVer1.py* 
 
 <img src="https://user-images.githubusercontent.com/97038348/168008762-bc02b001-799d-446c-9f54-48c350a3fc4e.png" width="30%" height="30%"/>

 * **objectSize** = eps for Clustering
 * **objectMovingRange** = eps for ClusteringVel
 * **filteringRange** = eps for ClusteringFilter
 * **sizeOfWindow** = number of elements in window
 * **maxNumOfSkip** = the number of maximum skip count of window. If skip count over this number, that window will be deleted.
 * **filterMode** =
   * 0: Simple moving average
   * 1: Weighted moving average
   * 2: Current-Weighted moving average
 * **weight** = weight for Current-Weighted moving average
 
 
 ---
 ---
## 2. Radar Tracking ver.2
   All about Radar Tracking version 2 is in *radar_tracking/src/ti_mmwave_rospkg/src/RadarTrackingVer2.py*
 ### Flowchart
 This Flowchart shows the algorithm of Radar Tracking version 2.   
There are 4 steps.   
Each steps are explained below.   
 <p align="center"><img src="https://user-images.githubusercontent.com/97038348/170816525-d0e41878-6381-4f4b-96fa-82a38837d931.jpg" width="50%" height="50%"/>
 
  ### Process diagram
  This Process diagram also shows the algorithm of Radar Tracking version 2.     
Each steps are explained below. 
  <img src="https://user-images.githubusercontent.com/97038348/170443955-3477412f-cd52-4c49-ab6a-e796eba4d1dd.PNG" width="110%" height="110%"/>
  
  
  ### First step: Obtain the points from Radar (IWR6843ISK)
 The values (point id, x position, y position, time) of detected points are subscribed from the ti_mmwave_rospkg.   
 
        * Package name: ti_mmwave_rospkg  
        * Launch name: 6843_multi_3d_0.launch
        * Topic name: /ti_mmwave/radar_scan      
        * msg Type : ti_mmwave_rospkg/RadarScan

           point_id - The number of scaned points
           x - x-coordinate of the point
           y - y-coordinate of the point
           header.stamp.secs - second
           header.stamp.nsecs - nano second
 
 #### Coordinate
<img src="https://user-images.githubusercontent.com/97038348/170430559-b56a5096-7a4c-40f0-bca2-26d9a74ec548.PNG" width="50%" height="50%"/>
 
 #### Specification
* Detection range in radial axis : 175(car), 98(person) [m]

* Range resolution : 0.039 ~ 0.044 [m]

* Azimuth angle of detection : ±60 [deg]

* Elevation angle of detection : ±20 [deg]

* Angular resolution (Azimuth) : 20 [deg]

* Angular resolution (Elevation) : 58 [deg]

* Maximum radial velocity : 9.59 [m/s]

(Note: Radial velocity represents how fast object go close or far from RADAR. It does not contain transverse velocity.)
   
   #### Reference 
   https://www.ti.com/design-resources/embedded-development/industrial-mmwave-radar-sensors.html#Evaluation
  
  ### Second step: Clustering the points from Radar and Finding Center point
  Point id =0 is the criterian for separating current and previous points.      
 When the point id is 0, the getPoint (Array) is clustered.   
 The function of **'Clustering'** is detecting objects and finding the center of each object.(Explaination of Clustering is [here](#explanation-about-clustering-1))    
 The position of current center point is assigned to the *'centerPoint'*   
 currentTime is the average of the time of current points
  
  ### Third step: Moving average Filter
  #### Window set diagram
   For making the moving average filter, **"Window set"** is defined.    
    Everything is same with version 1, but Element is only different.   
  Element is 1 x 2 matrix which contains X, Y position of centerPoint (X, Y veloctiy is not contained)    
  <img src="https://user-images.githubusercontent.com/97038348/170445351-94a4f08b-cb1d-46df-9ec5-4ebb8fab1bfe.PNG" width="90%" height="90%"/>    
    
  Window set contain k number of **"window"**.   
 The number of window is equal to the number of objects currently observed.   
 One window has n number of **"Elements"**.   
 Element is 1 x 2 matrix which contains X, Y position of centerPoint.     
 When the centerPoint is obtained from Second step, the centerPoint is clustered with first element of each window.    
   The function of **'ClusteringFilter'** is finding the window where centerPoint belongs.(Explaination of ClusteringFilter is [here](#explanation-about-clustering-1))         
   If the matched window is founded, the centerPoint becomes the first element and the previous elements are going down one by one.     
   The *'filtered position'* is obtained by applying a moving average filter to that window.  
  
 #### Formula of moving average filter
<img src="https://user-images.githubusercontent.com/97038348/167053294-ee42ca01-9129-4031-84b3-cccb81380603.PNG" width="80%" height="80%"/>
 
 There are three formula to calculate the average of window. Wieghted moving average and Current-Weighted moving average consider more weight to the recently observed values. You can find the proper value of weight by experiment.      
   *# You can select one by adjusting the PARAMETER in RadarTrackingVer2.py*
    
   #### Reference
   https://en.wikipedia.org/wiki/Moving_average
  
  If the matched window is not founded, new window is created.  
  
  
  Because Noise always makes new window, window that have not been updated for a long time will be deleted by counting the **"skip number"**.    
   When window is updated, skip count becomes 0. When window is not updated, 1 is added to skip count.       
 When the skip count exceeds the maximum number of skip count, the window would be deleted.
  
  ### Fourth step: Obtain Velocity
  For obtaining the velocity, **"Velocity Window set"** is defined.   
  
  #### Velocity Window set
  
  <img src="https://user-images.githubusercontent.com/97038348/170452480-c9b1d9e4-db0a-42a1-9da4-c6687d53b297.PNG" width="90%" height="90%"/>
 
  Velocity Window set contain k number of **"Velocity Window"**    
  Each window has each velocity window.   
  One velocity window has 2 elements.   
  X, Y of **"First element"** is current filtered position.   
  X, Y of **"Second element"** is previous filtered position.   
  Velocity is obtaining by following equation.     
 <img src="https://user-images.githubusercontent.com/97038348/170439150-20b51d3e-1eb0-4e49-b363-a61a8d55d2f9.PNG" width="40%" height="40%"/>   
  Obtained velocity is assigned to the Vx, Vy of First element. (current velocity)   
  *Object(X,Y,Vx,Vy)* is First element of the velocity window.    
 
  When the skip count exceeds the maximum number of skip count, the velocity window would be also deleted.
    
 ### Explanation about Clustering
 Before read this, please read [Appnedices](#5-appendices)(5.1) first.   
 There are two clustering in Radar Tracking version 2. (Clustering ClusteringFilter)
   
 #### 1.Clustering  
   The eps(radius) for **Clustering** should be the size of one object.    
   In the "good" case, the eps is almost same with the size of one person.   
   In the "bad" case, the eps is bigger than the size of one person. Two person can be perceived as one person.   
   <img src="https://user-images.githubusercontent.com/97038348/170825498-a3bd2ef5-1b0f-4a60-bfa8-a0e4ef3ed465.PNG" width="50%" height="50%"/>   
 
 **Clustering**(eps = objectSize, min_samples = 2)   
   objectSize = the radious of object  
   min_samples = The minimum number of detected points from object. if you increase this number, accuracy would increase.
   
  #### 2.ClusteringFilter 
   The eps(radius) for **ClusteringFilter** should be the size of object moving.   
   <img src="https://user-images.githubusercontent.com/97038348/170825500-d2849e49-2fa8-4656-a033-6b62b6495ad3.PNG" width="50%" height="50%"/> 
    
  **ClusteringFilter**(eps = filterRange, min_samples = 2)   
   filterRange = The range of object moving for time difference + Radar Error    
   min_samples = 2  &nbsp;&nbsp;&nbsp;*# ClusteringFilter judges whether Data and window is same or not. The code is made by this number. You should not change this*    
   
 ##### "Radar Error"
 <img src="https://user-images.githubusercontent.com/97038348/170826312-092fbf4f-a64d-41fa-a1a5-6e35ef26c408.PNG" width="70%" height="70%"/>   
   
  Because radar detect the points within the object randomly, the detected center is not always equal to the real center.       
 
    
  
   #### Parameter setting
 
 you have to set the parameters depending on what you want to detect. 
   For example, if you want to detect car, then objectSize would be big.    
   If you want to detect rabbit, then the objectSize would be small.
  &nbsp;&nbsp;&nbsp;&nbsp; *# You can set these values by adjusting the PARAMETER in RadarTrackingVer2.py* 
 
 <img src="https://user-images.githubusercontent.com/97038348/168715439-9b1a7074-5f56-476c-8430-5cb68311338d.png" width="30%" height="30%"/>

 * **objectSize** = eps for Clustering
 * **filteringRange** = eps for ClusteringFilter
 * **sizeOfWindow** = number of elements in window
 * **maxNumOfSkip** = the number of maximum skip count of window. If skip count over this number, that window will be deleted.
 * **filterMode** =
   * 0: Simple moving average
   * 1: Weighted moving average
   * 2: Current-Weighted moving average
 * **weight** = weight for Current-Weighted moving average
    
 
 
 ---
 ---
 ## 3. Experiments
 
 Radar Tracking was tested on people moving mainly along the Y-axis direction. The topic from Radar Tracking version 1 and 2 is recorded by rosbag. The bag file is converted into the csv file and the position and velocity graph of person is drawn by using matplotlib. The below figure is top view.   

 
 <img src="https://user-images.githubusercontent.com/97038348/168016836-3ccce287-880f-4b96-bf47-f4c3e0060de5.png" width="35%" height="60%"/>
 <img src="https://user-images.githubusercontent.com/97038348/168714663-ffe56093-8c5d-48ff-9e46-492f182d3403.gif" width="35%" height="60%"/>
  
 
 ### 3.1 Radar Tracking ver.1
 
  <p align="center"><img src="https://user-images.githubusercontent.com/97038348/170398622-cba6d74c-2130-4f74-aebd-544133e7a421.PNG" width="80%" height="80%"/>
   
   The green line is original data which is Data(X,Y,Vx,Vy).      
   The blue line is filterd data which is Object(X,Y,Vx,Vy).  
   You can judge the best one.
   
  
  
 ### 3.2 Radar Tracking ver.2
   

  <p align="center"><img src="https://user-images.githubusercontent.com/97038348/170398627-0db46132-48e5-4235-940a-a72d7533f1f9.PNG" width="80%" height="80%"/>
   
   The green line is original data which is centerPoint(X,Y).      
   The pink line is filterd data which is Object(X,Y,Vx,Vy).   
   You can judge the best one.
  
   
 ### 3.3 Experiment Data
   #### .csv and .bag
   <img src="https://user-images.githubusercontent.com/97038348/168712909-59a1d2a4-b2ef-4729-b087-0d0ca54f7fb0.png" width="50%" height="50%"/>
   You can find the .bag and .csv files in /experiment
    
   #### Reference
   https://colab.research.google.com/drive/1FAUvLSvKU4EWGs6MzC3Q67TUwbQ0ep9M#scrollTo=D06d4Hf05jRH
   https://colab.research.google.com/drive/1ElcqtUOXyF-yy6ymjZjCCcIXuI_Uwfsk#scrollTo=iNH4_roKjgVh
 
## 4. Running on ROS
   TI mmWave's IWR6843ISK is used for this radar tracking.   
    <img src="https://user-images.githubusercontent.com/97038348/169230164-4c639ac4-2d81-40c8-93e9-5b02415d1502.png" width="30%" height="30%"/>
   
   
 ### 4.1 Set up

install numpy and sklearn!

    $ sudo apt install python3-pip
    $ sudo pip3 install numpy
    $ sudo pip3 install scikit-learn
   
 git clone
   
    $ cd ~
    $ git clone https://github.com/nabihandres/RADAR_Cluster-and-tracking.git
   
 catkin make
   
    $ cd ~/RADAR_Cluster-and-tracking/radar_tracking
    $ catkin_make
    $ source devel/setup.bash
   
 add dialout to have acess to the serial ports on Linux 
   
    $ sudo adduser <your_username> dialout
   
 allow permission to serial port. (with connecting radar by usb port)
   
    $ sudo chmod a+rw /dev/ttyUSB0
 
 ### 4.2 Run the Radar Tracking
 
launch the ti mmwave ros package and get the data from radar

    $ roslaunch ti_mmwave_rospkg 6843_multi_3d_0.launch
    
rosrun the RadarTracking.py

    $ rosrun ti_mmwave_rospkg RadarTrackingVer1.py
    or
    $ rosrun ti_mmwave_rospkg RadarTrackingVer2.py
    
        
        
RadarTracking.py publish following topic     
    
    Published Topic : /object_tracking
    
    msg Type : object_msgs/Objects
 
        Objects - [Object 1, Object 2, ... ]
 
    msg Type : Object_msgs/Object
 
        name - object number in Window Set
        position - [X, Y, Z]  
                    X: x-coordinate of the object
                    Y: y-coordinate of the object
                    Z: z-coordinate of the object (= 0)
        velocity - [Vx, Vy, Vz]  
                    Vx: x-velocity of the object
                    Vy: y-velocity of the object
                    Vz: z-velocity of the object (= 0)
 
 
        


    
### Reference
https://github.com/nabihandres/RADAR_Coop_2022/edit/main/README.md   
https://dev.ti.com/  (Explore/ Resource Explorer/ Software/ mmWave Sensors/ Industrial Toolbox/ Labs/ Robotics/ ROS Driver)
    
 
 
 

 ## 5. Appendices
 ### 5.1 Scikit-learn.DBSCAN (Density-Based Spatial Clustering of Applications with Noise)
 ### Fundamental
     

<p align="center"><img src="https://user-images.githubusercontent.com/97038348/170610425-eec6fbf8-e7a7-47ec-a8ec-65a0ab287bc7.PNG" width="50%" height="50%"/>
   
   Suppose that there are 7 points. First, draw a circle with a radius around Point 1. The radius can be set by parameter ‘eps’. There are another parameter ‘min_samples’ which means minimum number of points in the circle of Point. In this case, the min_sample is set to 3. Because circle of Point 1 does not contain 3 points, Point 1 is not core point. Second, draw a circle with a radius around Point 2. Circle of Point 2 contain 4 points and Point 2 is defined as the core point. Also, Point 1 is included in the circle of core point. Point 1 is defined as border point. The same processes are repeated on all points. Point 7 does not include 3 points and is not included in other circles. Point 7 is defined as noise point. Point 1~6 are recognized as an object. 

### Parameters
* **eps**: The maximum distance between two samples for one to be considered as in the neighborhood of the other, default=0.5
* **min_samples**: The number of samples in a neighborhood for a point to be considered as a core point, default=5
* **metric**: The metric to use when calculating distance between insatances in a feature array, default='euclidean'


<img src="https://user-images.githubusercontent.com/97038348/164665803-93d061e6-0b21-4176-8621-e106bf3c597f.png" width="30%" height="30%"/>

### Attributes
 * **core_sample_indices_**: Indices of core samples
 * **components_**: Copy of each core sample found by training
 * **labels_**: Cluster labels for each point in the dataset given to fit(). Noisy samples are give the label -1
 

### Examples
    
    from sklearn.cluster import DBSCAN
    import numpy as np
    
    model = DBSCAN(eps=0.5, min_samples=1)
    # vector array
    data = np.array([[1,1],[2,0],[3,0],[1,0]])       
    Clustering = model.fit_predict(data)
    print(Clustering)
    
### Reference
 
https://scikit-learn.org/stable/modules/clustering.html#dbscan   (2.3.7. DBSCAN)   
https://scikit-learn.org/stable/modules/generated/sklearn.cluster.DBSCAN.html
 
 
 ### 5.2 Moving Average Filter
<p align="center"><img src="https://user-images.githubusercontent.com/97038348/167051870-63823a77-ae42-410d-a3ea-eee8e1ce440b.png" width="40%" height="40%"/>  
 
A moving average is a calculation to analyze data points by creating a seires of averages of window. When new element is obtained, the window is modified by **"shifting forward"** that is, excluding the first number of the window and including the next value in the window. A moving average is commonly used with time series data to smooth out short-term fluctuations and highlight longer-term trends. 
 
 #### Reference
https://en.wikipedia.org/wiki/Moving_average
