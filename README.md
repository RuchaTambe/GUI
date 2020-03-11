
How to run virtual camera on Docker:
1. Enter in the folder “Verkada-Assignment” using 
cd Verkada-Assignment on terminal 
2. Start docker-desktop
3. Run command 
	docker-compose up --build
URL to send request from client to server:
http://localhost:5001/logs 
Request type: GET

Explanation of code and how to make it production-ready
Camera side code

File Name: camera.py
Folder name: Client-Docker.

Explanation of functions:

1. generate_log():
The camera generates an event randomly selected from an array - events, every 10 seconds. The event is inserted into the log file ‘camera.log’ stored locally on camera side.

2. send_request():
The camera sends a POST request to the API server every 60 seconds with the recently updated logs. The request is timed out every 60 seconds and a new request is made. The process of sending logs is optimized by sending only the new logs generated in every minute, in place of sending all of the logs from start to current time. This reduces the amount of data to be sent to the server, every minute.
Previous log records are stored in the log file: camera.log on camera side and camera1.log on server side. Both are in sync with each other. 

This is done by storing contents of global variable temp_arr in another local temporary variable-local_temp and sending local_temp in the port request. Before sending the request, the contents of global temp_arr are made empty, so that upon generation of the new 6 event logs of the next minute, temp_arr will contain only the new logs. 


Server side code 
File Name: apiserver.py
Folder name: Server-Docker

Explanation of functions:
add_to_events():
	Invoked when a POST request is made by the camera. The request looks like http://server:5001/camera (A more detailed explanation for the generation of links is given in the Docker section)
	An object collects the data sent by camera and it is  written to a file camera1.log.
Long polling is implemented with the use of a global variable- user_requested, initially set to False. While there is no log request incoming from the client, the server keeps request from camera open until timeout of request or user_requested variable is True- implemented by a while loop in function. 

2. send_logs_to_user():
	Invoked when a GET request is made by client. URL would be:
http://localhost:5001/logs. 
The boolean variable user_requested is set to True so that the server can exit its wait state and fetch event logs from camera. The logs are then returned.


docker-compose.yml
The file contains two services: client and server.
Server:
The files in path provided in build tag are built by docker. (Server-Docker folder)
Port exposed is 5001, for other containers to send requests to the server.
Client
The files in path provided in build tag are built by docker. (Client-Docker folder)
There is a link created between client container and server container. This is why the POST request from camera to server contains ‘server’ and port number 5001 (http://server:5001/camera).

Making the code production-ready: 
1. Possibility of application crashes:
In case of an application crash of camera.py, there is a danger that all of the content in temp_arr is wiped out. In this case, logs are lost. 
Handling application crash: 
The process of sending logs can be made more robust by maintaining a checkpoint in the camera.log file. 
A. The checkpoint would be pointing to the last log added in the log file. Once new logs are added, every minute, only that data between the checkpoint +1 (pointing to the last log entry of the second to last set of log entries) and the last log of newly updated data should be sent to the server. The checkpoint is then updated to point to the last log in the log file. 
For Example, at the end of t=2mins, the checkpoint points to 12th log entry. At the end of t=3 mins, there are 6 more entries added to the log file. Now only entries from 13 to 18 will be sent to server. This way, the amount of data being sent from camera to server can be reduced. Checkpoint now points to entry number 18. 

In case of an application crash in the midst of sending logs to server, the checkpoint will point to an intermediate log entry of camera.log. We then need to read the log file and transmit log data that was not transmitted earlier. 
The application crash can be checked by checking the value of checkpoint at the beginning of send_request function.
	 If value of checkpoint !=start  of camera.log and value of checkpoint!=end of camera.log:
	A crash has occurred.
	Send the missing logs in post request by A.

2. Avoid reading camera.log file as much as possible
    File I/O operations take very long time to be performed, so all logs are written to the temp_arr list to be sent to server, along with writing them to the camera.log file. In this way, camera.log file reads are avoided during every minute interval, thus saving running time of the application. The file will only have to be read in case of a crash.
