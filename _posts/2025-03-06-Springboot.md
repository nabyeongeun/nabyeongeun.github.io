---
categories: [linux]
tags: [linux]
---

Springboot는 Legacy Springframework 보다는 여러면에서 편의성이 더 뛰어나다.  
하지만, 몇 안되는 불편한 점이 있는데, 실행스크립트를 직접 작성해야 한다는 점이다.  
Legacy Springframework의 경우에는 VM에다가 tomcat을 설치한 후에 배포파일만 잘 넣어두고 tomcat에서 기본제공하는 `start.sh`, `shutdown.sh`을 사용하면 되는데, Springboot로 배포파일을 만들면 달랑 application.jar 파일 하나만 나오고, 해당 파일을 실행시키거나 여러가지 옵션을 부여하는 데에 번거롭다.  
따라서 Springboot로 배포를 쉽게 하기 위해서 몇가지 스크립트를 준비했다.  
단순히 실행/종료 시킬 뿐만 아니라, 중복실행을 막거나 변수명만 살짝 바꿔서 쉽게 다른 어플리케이션에 적용하기 쉽게 하는데에 중점을 두었다

# 실행/종료 관련 스크립트 

## `start.sh`
```shell
#!/bin/bash

# Define constants
APP_NAME="qwertyApplication"
APP_JAR="/home/qwerty/mw/qwertyApplication.jar"
JAVA_HOME="/home/qwerty/jdk21"
PIDFILE="/home/qwerty/mw/${APP_NAME}.pid"

echo "Starting ${APP_NAME} application..."

# Check if the application is already running
if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
    echo "${APP_NAME} is already running with PID: $(cat $PIDFILE)"
    exit 1
fi

# Start the Java application with nohup and redirect logs
nohup ${JAVA_HOME}/bin/java \
    -jar \
    -Duser.timezone=Asia/Seoul \
    -Dfile.encoding=UTF-8 \
    -Xms8g \
    -Xmx10g \
    ${APP_JAR} \
    --logging.level.root=WARN \
    --spring.profiles.active=prod \
    --server.port=28080 > /dev/null &

# Capture the PID of the newly started process
PID=$!

# Wait briefly to ensure the process starts
sleep 2

# Check if the process started successfully
if ps -p $PID > /dev/null; then
    echo "${APP_NAME} application started successfully with PID: $PID"
    echo $PID > "$PIDFILE"
else
    echo "Failed to start ${APP_NAME} application."
    # Clean up PID file if it exists
    [ -f "$PIDFILE" ] && rm -f "$PIDFILE"
    exit 1
fi

exit 0
```
프로젝트 내에 포함된 `logback`을 사용하는 것을 전제하여, OS레벨에서 로그를 남기지 않는 것으로 설정한다.

## `shutdown.sh`
```shell
#!/bin/bash

# Define constants
APP_NAME="qwertyApplication"
PIDFILE="/home/qwerty/mw/${APP_NAME}.pid"

# Function to find the PID (PID file 우선, 없으면 ps로 검색)
find_pid() {
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        cat "$PIDFILE"
    else
        ps aux | grep "[j]ava.*${APP_NAME}.jar" | awk '{print $2}'
    fi
}

# Function to check if the process is still running
is_running() {
    [ -n "$(ps -p $1 -o pid=)" ]
}

# Function to send a signal to the process
send_signal() {
    kill -$1 $2
    echo "Sent $1 signal to process $2"
}

# Main script
PID=$(find_pid)

if [ -z "$PID" ]; then
    echo "${APP_NAME} application is not running."
    exit 0
fi

echo "Found ${APP_NAME} application with PID: $PID"

# Try graceful shutdown first (SIGTERM)
send_signal TERM $PID
sleep 15  # Increase wait time for graceful shutdown

# Check if the process is still running
if is_running $PID; then
    echo "Application did not shut down gracefully. Attempting force quit..."
    send_signal KILL $PID
    sleep 5

    if is_running $PID; then
        echo "Failed to terminate the application. Please check manually."
        exit 1
    fi
fi

# Clean up PID file
[ -f "$PIDFILE" ] && rm -f "$PIDFILE"
echo "${APP_NAME} application has been successfully terminated."
exit 0               
```
## `restart.sh`
```shell
#!/bin/bash

sh /home/qwerty/mw/shutdown.sh

sleep 5

sh /home/qwerty/mw/start.sh
```

## `update.sh`
```shell
#!/bin/bash

# Define constants
APP_NAME="qwertyApplication"
APP_JAR="${APP_NAME}.jar"
APP_READY="${APP_NAME}_ready.jar"
MW_DIR="/home/qwerty/mw"
BACKUP_DIR="/home/qwerty/mw/backup"
SHUTDOWN_SCRIPT="${MW_DIR}/shutdown.sh"
START_SCRIPT="${MW_DIR}/start.sh"

# Step 1: Terminate the application using shutdown.sh
echo "Terminating ${APP_NAME} application..."
sh "${SHUTDOWN_SCRIPT}"
sleep 1

# Step 2: Rename current .jar to .jar_yyyymmdd (backup)
DATE=$(date +"%Y%m%d")
OLD="${MW_DIR}/${APP_JAR}"
NEW="${BACKUP_DIR}/${APP_JAR}_${DATE}"

if [ -f "$NEW" ]; then
    echo "Removing existing backup: $NEW"
    rm "$NEW"
fi

if [ -f "$OLD" ]; then
    echo "Backing up $OLD to $NEW"
    mv "$OLD" "$NEW"
else
    echo "$OLD not found!"
fi

# Step 3: Rename application_ready.jar to application.jar
READY="/home/qwerty/${APP_READY}"
if [ -f "$READY" ]; then
    echo "Deploying $READY to $OLD"
    mv "$READY" "$OLD"
else
    echo "$READY not found!"
fi

# Step 4: Start the application using start.sh
echo "Starting ${APP_NAME} application..."
sh "${START_SCRIPT}"
```
실제 톰캣 역할을 하는 것은 `qwertyApplication.jar`이고, 한 단계 아래 디렉토리에 `qwertyApplication_ready.jar`가 존재한다고 가정한다.  
그래서 톰캣을 종료한 후에 `qwertyApplication.jar`에 날자를 달아서 백업디렉토리로 이동시키고, `qwertyApplication_ready.jar`를 `qwertyApplication.jar`로 변경해서 실행시킨다.

참고로 배포파일의 이름을 바꾸려면 Springboot프로젝트 내에 `build.gradle`에 다음 내용을 추가하면 된다.
```properties
bootJar {
    archiveBaseName = 'qwertyApplication_ready'
    archiveFileName = 'qwertyApplication_ready.jar'
}
```

# 로그 관련 스크립트

어플리케이션을 최초 기동할 때에는 로그레벨을 `WARN`으로 실행시키고, 톰캣 재시작 없이 자세한 로그를 보기위해 `INFO`로 조정하는 상황이 있다고 가정한다.  
actuator를 통해 현재 로그 레벨을 변경하거나 조회한다.

## `loginfo.sh`
```shell
#!/bin/bash

curl -H "Content-Type: application/json" -X POST http://localhost:28080/actuator/loggers/ROOT?configuredLevel=INFO
curl -H "Content-Type: application/json" -X POST http://localhost:28080/actuator/loggers/org.springframework.web.servlet.DispatcherServlet?configuredLevel=DEBUG
```
DispatcherServlet을 DEBUG로 지정해야 request가 들어왔을때 request url이 로그에 남기 때문에 request의 시작과 끝을 파악하는 데에 큰 도움이 된다.

## `logwarn.sh`

```shell
#!/bin/bash

curl -H "Content-Type: application/json" -X POST http://localhost:28080/actuator/loggers/ROOT?configuredLevel=WARN
curl -H "Content-Type: application/json" -X POST http://localhost:28080/actuator/loggers/org.springframework.web.servlet.DispatcherServlet?configuredLevel=WARN

```

## `logstatus.sh`
```shell
#!/bin/bash

curl http://localhost:28080/actuator/loggers/ROOT

```
