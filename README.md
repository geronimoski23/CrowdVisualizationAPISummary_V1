# Crowd Visualization Project


### Project Goal

This project aims to use data obtained from wifi syslogs from buildings around the UMass Amherst campus to display a heatmap of the campus at a specific minute in the day. In order to do this, raw data from previous years was processed and stored in CSV files containing information about devices and their connections to access points within the various buildings on campus. Through data manipulation and file reading, we created an API view using the Django web framework to display the necessary and relevant information for our front end to call and display on an interactive heatmap using the leaflet javascript library.

### Backend Version 1.0

#### Campus Occupancy

The goal for this API was to return building information for all the buildings on campus which would allow the client to display a heatmap with both minute and hour granularity. In order to create the API, we first had to go through the CSV data and figure out what each column meant and what data was relevant for the heatmap display. We determined that the only information we needed were the start and end times of each connection made to an endpoint within the building and the building. 


```python
import pandas as pd
```


```python
data = pd.read_csv('info_20210301_sessions_final.csv', low_memory=False)
```


```python
d_head = data.head()
markdown_table = d_head.to_markdown(index=False)
```

| Building   |   AP_Count | AP_List                                                  | AP_Start_Time        | AP_End_Time                  | MAC                      | Date       |
|:-----------|-----------:|:---------------------------------------------------------|:---------------------|:-----------------------------|:-------------------------|:-----------|
| PIER       |          3 | ['PIER-422-1', 'PIER-414-1', 'PIER-421-1']               | [67, 81, 152]        | [81.0, 152.0, 153.0]         | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [545]                | [603.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [698]                | [716.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          4 | ['PIER-318-1', 'PIER-421-1', 'PIER-317-1', 'PIER-421-1'] | [816, 819, 832, 835] | [817.0, 832.0, 835.0, 845.0] | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [873]                | [898.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |

The start and end time for each access point connection was processed as a minutes offset from 12:00 am (e.g. 545 = 9:05 am). The abbreviation was the building's abbreviation, in the example above, PIER represented the Pierpont Hall.

To create the heatmap for the campus, the front end required building coordinates, latitude & longitude, as well as an intensity or count. In order to get the latitude and longitude, we manually found the coordinates for each building in the CSV file and created a dictionary that mapped the building abbreviation to the buildings coordinates. 

The count was obtained in two different ways since we had to display both minute and hour granularity. To display minute granularity, we had to check whether the device was connected to the building within the first start time and the last end time. Using the second row in the example above, the device was connected to an access point within PIER from 9:05 am to 10:03 am. If the time the client selected for was 10:00 am, the row above would be added as one to the running total of connections made to the building "PIER". To display hour granularity, we followed the same logic, except we would check if the device was connected to an access point at any time within the hour. For example, if the client was checking for the hour 9:00 am, we would check all devices that were connected within 9:00 am - 9:59 am. After iterating and adding a device to the count, the device ID would be added to a set to avoid duplicate device IDs being counted. 

A portion of the JSON response with the following api endpoint, *http://localhost:8000/api/v1/campus/datetime/2021-03-01T13:00/?granularity=hour*, would return 

```json
"data": [
        {
            "date": "2019-02-28",
            "building": "PIER",
            "building_lat": 42.38138,
            "building_long": -72.530921,
            "connection_count": 101
        },
        {
            "date": "2019-02-28",
            "building": "PRIN",
            "building_lat": 42.384059,
            "building_long": -72.528867,
            "connection_count": 70
        },
            {
            "date": "2019-02-28",
            "building": "BKDC",
            "building_lat": 42.381875,
            "building_long": -72.529885,
            "connection_count": 28
        },
```

This response shows the date, the building and its coordinates, and how many devices were connected to it from 1:00 pm to 1:59 pm. 

When looking at minute granularity, the endpoint would be *http://localhost:8000/api/v1/campus/datetime/2021-03-01T13:00/?granularity=minute*

```json
"data": [
        {
            "date": "2019-02-28",
            "building": "PRIN",
            "building_lat": 42.384059,
            "building_long": -72.528867,
            "connection_count": 52
        },
        {
            "date": "2019-02-28",
            "building": "BKDC",
            "building_lat": 42.381875,
            "building_long": -72.529885,
            "connection_count": 17
        },
        {
            "date": "2019-02-28",
            "building": "SC",
            "building_lat": 42.389545,
            "building_long": -72.529299,
            "connection_count": 2
        },
```

The response would show the same stats except the count would be lower as it would only show devices connected to the building at the exact minute of 1:00 pm. 


#### Building Occupancy

In addition to seeing the campus view of the heatmap, we also wanted to add more interaction and information for the building. To do this, we created another api at the building level which shows the information specific to the building when called upon. At the minute granularity level, no new information is provided other than how many floors the building has. For example, using the same time previously and the building, Knowlton Hall (KNWL), the endpoint would be, http://localhost:8000/api/v1/building/KNWL/datetime/2021-03-01T13:00/?granularity=minute and the json response would be:

```json

"data": {
    "building": "KNWL",
    "building_lat": 42.393834,
    "building_long": -72.525742,
    "connection_count": 31,
    "no_floors": 4
}

```

When switching to hour granularity, more statistics are provided for the building such as the average and the standard deviation of time spent in the building within the hour. To calculate this, we first checked to see if the device was in the building within the hour selected. Since we wanted to provide stats specifically for the hour, if the device was connected to the building before the hour and left the building after the hour, its total stay time would be 59 minutes. If it started before the hour and left in the middle, we subtracted the lower bound of the hour selected from the end time to get its total stay time. If the device's session at the building started within the hour but ended after the hour, we subtracted the start time from the upper bound of the hour selected to get its total time. If the start and end time of the device session are within the bounds, the total time is simple the start time subtracted from the end time. Each elapsed time value was added to an array within a pandas series in order to get the mean and standard deviation of a devices total time spent in the building. An example using the same building in the previous example would have the endpoint, http://localhost:8000/api/v1/building/KNWL/datetime/2021-03-01T13:00/?granularity=hour and the json response would be:

```json
 "data": {
        "building": "KNWL",
        "building_lat": 42.393834,
        "building_long": -72.525742,
        "average": 40.8,
        "standard_deviation": 22.2,
        "connection_count": 36,
        "no_floors": 4
    }
```


#### Access Point Occupancy
---
The last API we added was one to show occupancy at different access points within a specific building. Since we did not have access to floor plans for any of the buildings given, we generated random latitude and longitude points for each access point within the building in order to demonstrate how this api call could be displayed by the client. The api returns a response similar to the format of the campus api but instead of a building, it returns and access point. In addition to this, the api filters each access point by floor in order to allow the client to display each floor's occupancy with ease. For example, an access point within the Life Sciences Lab is named "LSL-S473-1" which would mean that it is an access point on the fourth floor. 

An portion of the response for level 5 of the LSL building with minute granularity with the endpoint http://localhost:8000/api/v1/building/LSL/datetime/2021-03-01T13:00/access_point/?level=5&granularity=minute would be:

```json
  "data": [
    {
      "date": "2019-02-28",
      "access_point": "LSL-S540-1",
      "connection_count": 1,
      "building_lat": 42.392155639318474,
      "building_long": -72.5237615593268
    },
    {
      "date": "2019-02-28",
      "access_point": "LSL-N560C-1",
      "connection_count": 3,
      "building_lat": 42.392310688378785,
      "building_long": -72.52403717553342
    },
    {
      "date": "2019-02-28",
      "access_point": "LSL-N540-1",
      "connection_count": 1,
      "building_lat": 42.39263478211423,
      "building_long": -72.5239181749668
    },
```

If we were to switch it to hour granularity, the endpoint would be http://localhost:8000/api/v1/building/LSL/datetime/2021-03-01T13:00/access_point/?level=5&granularity=hour and the response would be:

```json
  "data": [
    {
      "date": "2019-02-28",
      "access_point": "LSL-N599L-1",
      "connection_count": 2,
      "building_lat": 42.392215070425834,
      "building_long": -72.52379034506785
    },
    {
      "date": "2019-02-28",
      "access_point": "LSL-S540-1",
      "connection_count": 3,
      "building_lat": 42.392155639318474,
      "building_long": -72.5237615593268
    },
    {
      "date": "2019-02-28",
      "access_point": "LSL-N560C-1",
      "connection_count": 4,
      "building_lat": 42.392310688378785,
      "building_long": -72.52403717553342
    },
```



#### Trajectory

The goal of processing the trajectory data was to process the data in such a way that the client could display each building that a device visited throughout the day. The information we recieved from the trajectory data was a list of device ids and each building or access point they had connected to throughout the day, as well as the time they stayed at each building or access point. We processed the data from the original csv to only include non-duplicate device ids and to only show data for devices that stayed in two or more unqiue buildings for longer than 10 minutes. The data we accounted for followed the format:


```python
data = pd.read_csv('info_20131129_final_traj.csv', low_memory=False)
```


```python
d_head = data.head()
markdown_table = d_head.to_markdown(index=False)
```

| Building   |   AP_Count | AP_List                                                  | AP_Start_Time        | AP_End_Time                  | MAC                      | Date       |
|:-----------|-----------:|:---------------------------------------------------------|:---------------------|:-----------------------------|:-------------------------|:-----------|
| PIER       |          3 | ['PIER-422-1', 'PIER-414-1', 'PIER-421-1']               | [67, 81, 152]        | [81.0, 152.0, 153.0]         | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [545]                | [603.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [698]                | [716.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          4 | ['PIER-318-1', 'PIER-421-1', 'PIER-317-1', 'PIER-421-1'] | [816, 819, 832, 835] | [817.0, 832.0, 835.0, 845.0] | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |
| PIER       |          1 | ['PIER-421-1']                                           | [873]                | [898.0]                      | #..937mQyk9UvpDtAM9j2wQ# | 2019-02-28 |

The endpoint for the api response requires the date for the trajectory data and the device id, http://localhost:8000/api/v1/trajectory/.ga9Aph.WKgKcgGMkPGJ3Q/date/2013-11-29/ and returns the following api response:

```json
{
    "data": [
        [
            {
                "building": "CC",
                "building_lat": 42.391733,
                "building_long": -72.527049,
                "start_time": "12:02 am",
                "end_time": "7:11 am",
                "total_time": 429.0
            },
            {
                "building": "STAD",
                "building_lat": 42.377243,
                "building_long": -72.536005,
                "start_time": "9:01 am",
                "end_time": "9:24 am",
                "total_time": 23.0
            },
            {
                "building": "STAD",
                "building_lat": 42.377243,
                "building_long": -72.536005,
                "start_time": "9:37 am",
                "end_time": "10:55 am",
                "total_time": 78.0
            }
        ]
    ]
}
```

This format shows the client that the particular device stayed in the CC building from 12:02 am to 7:11 am, stayed in the CC building form 9:01 am to 9:24 am, and the PRIN building from 9:37 am to 10:55 am.


### Conclusion

With these four APIs, we can create a server that can display an interactive heatmap and a device's trajectory. In order to improve upon the existing design, some ideas we will explore in the future include adding a database for users so that the historical data can be stored and called upon when needed or to predict where a user wants to go next at a specific time. Another way we could improve upon the design is to read wifi syslog data in real-time when creating the heatmap.

