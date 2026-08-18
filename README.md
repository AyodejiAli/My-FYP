# Adaptive Honeypot and Anomaly Detection System

This project is a combined honeypot monitoring system that uses Cowrie and OpenCanary to collect network attack data from multiple services.

Cowrie captures SSH and Telnet activity, while OpenCanary monitors services such as FTP, HTTP and MySQL. The generated logs are processed in real time, analysed using machine learning, stored in an SQLite database and displayed through a Flask web dashboard.

## Project Features

- SSH and Telnet monitoring using Cowrie
- Multi-protocol monitoring using OpenCanary
- Real-time processing of honeypot logs
- Unsupervised anomaly detection
- Isolation Forest and One-Class SVM models
- SQLite event storage
- Live Flask dashboard
- Real-time updates using Flask-SocketIO
- Attack simulation tools for testing
- Charts showing attack and anomaly activity
- API access to stored events

## System Architecture

The system follows this general flow:

```text
Cowrie and OpenCanary
        |
        v
Honeypot Log Files
        |
        v
Log Watchers
        |
        v
Feature Engineering
        |
        v
Anomaly Detection
        |
        v
SQLite Database
        |
        v
Flask API and WebSocket Server
        |
        v
Live Web Dashboard
```

Cowrie and OpenCanary act as the entry points for network activity. Their logs are continuously monitored by the custom Python log watchers.

Each new event is parsed and converted into numerical features. The event is then analysed by the machine-learning models, stored in the database and sent to the dashboard.

## Project Structure

```text
final-year-project/
|
├── cowrie/
│   ├── bin/
│   ├── cowrie-env/
│   ├── etc/
│   └── var/
│       └── log/
│           └── cowrie/
│               └── cowrie.json
|
├── database/
│   └── cowrie_events.db
|
├── simulators/
│   ├── ftp_simulator.py
│   ├── bulk_attack_simulator.py
│   └── opencanary_attack.py
|
├── static/
│   ├── js/
│   │   └── main.js
│   ├── style.css
│   └── simulated.css
|
├── templates/
│   ├── index.html
│   └── simulated.html
|
├── anomaly_model.py
├── db.py
├── log_watcher.py
├── webapp.py
├── main.py
├── requirements.txt
└── README.md
```

The exact structure may differ slightly depending on where Cowrie and the custom Python scripts were installed.

## Main Components

### Cowrie

Cowrie is an SSH and Telnet honeypot. It records information such as:

- Connection attempts
- Source IP addresses
- Usernames
- Passwords
- Commands entered by attackers
- Session activity
- File download attempts

Cowrie writes its structured events to:

```text
cowrie/var/log/cowrie/cowrie.json
```

### OpenCanary

OpenCanary is a lightweight multi-protocol honeypot. It expands the project beyond SSH and Telnet by monitoring services such as:

- FTP
- HTTP
- MySQL
- SMB
- Telnet
- Other configurable network services

SSH and Telnet should remain disabled in OpenCanary when Cowrie is already responsible for those services.

The OpenCanary configuration file is located at:

```text
/home/spinn/.opencanary.conf
```

The OpenCanary log file is located at:

```text
/var/tmp/opencanary.log
```

### Log Watcher

The `log_watcher.py` file monitors the Cowrie and OpenCanary logs.

It uses the `watchdog` library to detect when new information is written to either log file. Each new entry is parsed according to its source.

The watcher performs the following tasks:

1. Detects changes to a honeypot log.
2. Reads the newly added event.
3. Extracts useful values such as the IP address, username, password and command.
4. Sends the event to the anomaly-detection model.
5. Stores the processed event in the database.
6. broadcasts the event to the dashboard.

Separate handlers are used because Cowrie and OpenCanary produce different log structures.

### Anomaly-Detection Model

The `anomaly_model.py` file contains the machine-learning and feature-engineering logic.

The system uses:

- Isolation Forest
- One-Class SVM

These are unsupervised learning algorithms, meaning they can identify unusual activity without requiring a fully labelled dataset.

The generated features can include:

- IP-address entropy
- Number of previous events from the IP
- Time since the previous request
- Username and password characteristics
- Command length
- Command complexity
- Command frequency
- Login behaviour
- Source service

The model output is combined with additional rules. For example, an event may receive a higher anomaly score when:

- Both models identify it as unusual.
- The same IP generates many requests.
- Requests occur within a very short time.
- Suspicious or uncommon commands are entered.
- Login attempts use repeated credentials.

The models can use a sliding window containing the latest events so that the system adapts to recent network behaviour.

### Database

The `db.py` file manages the SQLite database.

The main application database is:

```text
database/cowrie_events.db
```

The database can store:

- Timestamp
- Source IP address
- Username
- Password
- Command
- Protocol or service
- Honeypot source
- Anomaly status
- Anomaly score

The live application should always use `cowrie_events.db`.

A file such as `cowrie_events_copy.db` should only be treated as a backup or exported snapshot. The application should not read from the copied database during normal operation.

The database may automatically remove older rows when it exceeds a configured limit. This prevents the local database from becoming unnecessarily large.

### Flask Web Application

The `webapp.py` file contains the Flask backend.

Its responsibilities include:

- Loading the dashboard
- Querying the database
- Returning events through API routes
- Receiving simulated attack events
- Formatting database results
- Broadcasting events using Flask-SocketIO

Important routes include:

| Route | Purpose |
|---|---|
| `/` | Displays the main dashboard |
| `/simulate` | Opens the simulated attack interface |
| `/api/events` | Returns recent events as JSON |
| `/api/simulated-attack` | Accepts simulated attack data |

Flask-SocketIO allows new events to appear on the dashboard without requiring the browser page to be refreshed.

### Frontend

The `templates` and `static` folders contain the dashboard interface.

The frontend includes:

- A live event table
- Anomaly status indicators
- Attack-source information
- Event counters
- Anomaly-ratio charts
- Command-frequency charts
- A simulated attack form

Chart.js can be used to display graphs such as:

- Normal events compared with anomalies
- Most common commands
- Most active IP addresses
- Events grouped by honeypot source

### Attack Simulators

The scripts inside `simulators/` generate controlled test activity.

They are used to verify that:

- Honeypots receive events.
- Logs are created correctly.
- Log watchers detect new entries.
- Features are generated.
- Machine-learning models process the events.
- Events are stored in the database.
- The dashboard updates successfully.

These scripts should only be used against the local honeypot environment or systems where testing has been authorised.

## Requirements

The project requires:

- Ubuntu or WSL Ubuntu
- Python 3
- Cowrie
- OpenCanary
- Flask
- Flask-SocketIO
- watchdog
- scikit-learn
- pandas
- NumPy
- SQLite

Install the Python requirements using:

```bash
cd ~/final-year-project
source cowrie/cowrie-env/bin/activate
pip install -r requirements.txt
```

If a requirements file has not been created, the main packages can be installed using:

```bash
pip install flask flask-socketio watchdog scikit-learn pandas numpy requests
```

## Starting the System

The components should be started in the following order:

1. Cowrie
2. OpenCanary
3. Flask dashboard and processing pipeline

### Start Cowrie

Open a terminal and run:

```bash
cd ~/final-year-project/cowrie
source cowrie-env/bin/activate
./bin/cowrie start
```

Check the Cowrie status:

```bash
./bin/cowrie status
```

Follow the Cowrie log in real time:

```bash
tail -f var/log/cowrie/cowrie.json
```

### Start OpenCanary

Make sure the OpenCanary environment is available in the current path:

```bash
export PATH="$HOME/.local/share/pipx/venvs/opencanary/bin:$PATH"
```

Start OpenCanary:

```bash
~/.local/bin/opencanaryd --start --no-daemon --config=/home/spinn/.opencanary.conf
```

Keep this terminal open while OpenCanary is running.

Follow the OpenCanary log in another terminal:

```bash
tail -f /var/tmp/opencanary.log
```

### Start the Flask Dashboard

Open another terminal and run:

```bash
cd ~/final-year-project
source cowrie/cowrie-env/bin/activate
python3 main.py
```

If `main.py` is stored inside the `Scripts` directory, use:

```bash
python3 Scripts/main.py
```

The main script starts:

- Cowrie log monitoring
- OpenCanary log monitoring
- Anomaly detection
- Database access
- Flask web server
- Flask-SocketIO updates
- Any configured background simulators

The dashboard should then be available at:

```text
http://localhost:5000
```

The simulated attack interface should be available at:

```text
http://localhost:5000/simulate
```

## Stopping the System

### Stop the Flask Application

In the terminal running `main.py`, press:

```text
CTRL+C
```

If port 5000 remains in use, find the process:

```bash
lsof -i :5000
```

Stop the required process using its PID:

```bash
kill <PID>
```

Only use a forced stop if the normal command does not work:

```bash
kill -9 <PID>
```

### Stop OpenCanary

Press `CTRL+C` in the terminal where OpenCanary is running.

If it continues running, locate the process:

```bash
ps aux | grep opencanary
```

Stop it using:

```bash
sudo pkill -f opencanary
```

### Stop Cowrie

Run:

```bash
cd ~/final-year-project/cowrie
source cowrie-env/bin/activate
./bin/cowrie stop
```

Confirm its status:

```bash
./bin/cowrie status
```

## Start and Stop Summary

| Task | Command |
|---|---|
| Start Cowrie | `./bin/cowrie start` |
| Check Cowrie | `./bin/cowrie status` |
| Stop Cowrie | `./bin/cowrie stop` |
| Start OpenCanary | `~/.local/bin/opencanaryd --start --no-daemon --config=/home/spinn/.opencanary.conf` |
| Stop OpenCanary | `sudo pkill -f opencanary` |
| Start dashboard | `python3 main.py` |
| Stop dashboard | `CTRL+C` |
| Open dashboard | `http://localhost:5000` |
| Open simulator | `http://localhost:5000/simulate` |

## Simulating an Attack

The system provides a local API route that can be used to test the full processing pipeline.

Make sure the Flask application is running before submitting a simulated event.

Run:

```bash
curl -X POST http://localhost:5000/api/simulated-attack \
-H "Content-Type: application/json" \
-H "X-Forwarded-For: 192.168.1.55" \
-d '{
  "username": "admin",
  "password": "1234",
  "command": "ls -la"
}'
```

The simulated event should:

1. Be received by Flask.
2. Be converted into model features.
3. Be analysed for anomalous behaviour.
4. Be inserted into the SQLite database.
5. Appear on the dashboard.

### Test Repeated Login Attempts

Repeated requests can be submitted using:

```bash
for i in {1..20}
do
  curl -s -X POST http://localhost:5000/api/simulated-attack \
  -H "Content-Type: application/json" \
  -H "X-Forwarded-For: 192.168.1.55" \
  -d '{
    "username": "admin",
    "password": "password123",
    "command": "whoami"
  }'
done
```

This can test whether the frequency and timing features detect automated behaviour.

## Running the FTP Simulator

From the project root, activate the Python environment:

```bash
cd ~/final-year-project
source cowrie/cowrie-env/bin/activate
```

Run the simulator:

```bash
python3 simulators/ftp_simulator.py
```

Stop it using:

```text
CTRL+C
```

## Running Unit Tests

Activate the project environment:

```bash
cd ~/final-year-project
source cowrie/cowrie-env/bin/activate
```

Install `pytest` if required:

```bash
pip install pytest
```

Run all tests:

```bash
pytest
```

Run a specific test file:

```bash
pytest test_core.py
```

If the tests are inside the `Scripts` directory, run:

```bash
pytest Scripts/test_core.py
```

Modules such as `db` and `anomaly_model` are local project files. They should not be installed using `pip`.

Tests should be executed from the project root so Python can find the local modules.

## Resetting the Database

Stop the Flask application before resetting the database.

Delete the existing database:

```bash
cd ~/final-year-project
rm database/cowrie_events.db
```

Reinitialise it:

```bash
python3 db.py
```

If `db.py` is inside the `Scripts` directory, use:

```bash
python3 Scripts/db.py
```

Restart the application:

```bash
python3 main.py
```

Deleting the database permanently removes the stored events. Create a backup first if the data is required.

## Creating a Database Backup

Stop the application before copying the database to avoid creating an incomplete backup.

Run:

```bash
cp ~/final-year-project/database/cowrie_events.db \
/mnt/c/Users/spinn/Desktop/cowrie_events_copy.db
```

The copied file is only a backup. The live system should continue reading from and writing to:

```text
database/cowrie_events.db
```

## Log Locations

| Component | Log file |
|---|---|
| Cowrie | `cowrie/var/log/cowrie/cowrie.json` |
| OpenCanary | `/var/tmp/opencanary.log` |
| Flask application | Terminal running `main.py` |
| SQLite events | `database/cowrie_events.db` |

View Cowrie logs:

```bash
tail -f ~/final-year-project/cowrie/var/log/cowrie/cowrie.json
```

View OpenCanary logs:

```bash
tail -f /var/tmp/opencanary.log
```

## Troubleshooting

### Flask Cannot Connect on Port 5000

Check whether Flask is running:

```bash
lsof -i :5000
```

Start the application:

```bash
cd ~/final-year-project
source cowrie/cowrie-env/bin/activate
python3 main.py
```

### OpenCanary Command Not Found

Add the OpenCanary environment to the path:

```bash
export PATH="$HOME/.local/share/pipx/venvs/opencanary/bin:$PATH"
```

Check that OpenCanary is installed:

```bash
~/.local/bin/opencanaryd --help
```

### Module Not Found

Make sure the correct environment is active:

```bash
source ~/final-year-project/cowrie/cowrie-env/bin/activate
```

Install the missing dependency:

```bash
pip install <package-name>
```

For local files such as `db.py` or `anomaly_model.py`, run the command from the project root instead of trying to install them.

### Pytest Not Found

Install it inside the active virtual environment:

```bash
pip install pytest
```

### No Events Appear on the Dashboard

Check the following:

1. Cowrie or OpenCanary is running.
2. The correct log paths are configured.
3. `main.py` is running.
4. The Flask dashboard is available on port 5000.
5. The watchers have permission to read the logs.
6. The database path points to `cowrie_events.db`.
7. The browser console does not show a WebSocket error.

## Security and Ethical Use

This project is intended for educational, research and authorised security-testing purposes.

Only deploy or test the system on networks and machines that you own or have permission to use.

Because a honeypot is intentionally exposed to suspicious activity, it should be isolated from important personal or organisational systems. Any files or commands collected from attackers should be treated as potentially malicious.

## Possible Future Improvements

Potential extensions include:

- Threat-intelligence API integration
- Automatic IP reputation checking
- Geographic attack visualisation
- Email or messaging alerts
- Docker deployment
- Cloud-hosted monitoring
- Model-performance monitoring
- Improved attack classification
- Larger database support
- Additional honeypots and protocols
- Exportable incident reports

## Technologies Used

- Python
- Cowrie
- OpenCanary
- Flask
- Flask-SocketIO
- watchdog
- scikit-learn
- Isolation Forest
- One-Class SVM
- SQLite
- JavaScript
- Chart.js
- HTML
- CSS

## Author

Ayodeji Ali

Final-Year Computer Science Project
