# Log Streaming Service
### Technologies Used
REST API
 - Python(FastAPI)
 - Containerized via Docker

UI
 - Terminal Based
### API Routes

Routes | HTTP | Description
--- | --- | ---
**/stream-logs** | `GET` | Subscribe to log stream
**/existing-logs** | `GET` | Get all logs uptil now and subscribe to subsequent logs

## Installation

To run the API Locally, you need to perform the following steps:

1. download the application's source code into a desired directory and `cd` into it

   ```sh
   $ cd Log-Streaming-Service
   ```

2. Next we build the docker container which will run our API

   ⚠️ Note : Start Docker daemon first

   ```sh
   $ make build
   ```
This will build the container as defined in the `Dockerfile`

4. Initialize the API
   ```sh
   $ make run
   ```
You should be able to see relevant logs which notify that the application has started and the server will start running at `localhost:8000`

⚠️ once the app starts, a `logs` folder should have been created. If not, kindly create one at the root directory

### Testing

Assuming the log file is empty,
run the following script after spinning up the API

   ```sh
   $ make generate_logs
   ```

1. Streaming real-time logs

   ! (`-Lv` flag is optional (it renders extra info about the request)
   ```sh
   $ make stream
   ```

2. Getting all logs uptil now and then continue streaming all further logs(if any) post the request

   ```sh
   $ curl -N -H "Accept:text/event-stream" http://localhost:8000/existing-logs

   ```

    Note : To simulate concurrent clients, multiple terminal windows can be used to send different requests

   Steps for graceful shutdown
    - First terminate the curl requests which are streaming the logs in real-time
    - Next pause the `log_generator.py` script using keyborad interrupt
    - Then stop the API using keyborad interrupt
