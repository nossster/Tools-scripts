---
order: 1
title: Скрипт оповещения о входе клиента
---

для работы скрипта в настройках *переменных среды* надо создать 3 переменные с названиями

область применения не важна.

| Название             | Значение                           |
|----------------------|------------------------------------|
| **ALLOWED_USERS**    | Список пользователей через запятую |
| **TELEGRAM_TOKEN**   | Телеграм токен вашего бота         |
| **TELEGRAM_CHAT_ID** | Ваш чат ID                         |

Далее создайте задачу *тип скрипта* **Batch** с таким содержимым. И установите в секции *Вход пользователя / Выход пользователя*

```
@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion

:: === CONFIG FROM ENVIRONMENT ===
set "TELEGRAM_TOKEN=%TELEGRAM_TOKEN%"
set "CHAT_ID=%TELEGRAM_CHAT_ID%"
set "ALLOWED_USERS=%ALLOWED_USERS%"

:: === GET VARIABLES FROM PROGRAM ===
set "CUR_USER=%CUR_USER%"
set "CUR_USER_GROUP=%CUR_USER_GROUP%"
set "HOST_NUMBER=%HOST_NUMBER%"
set "HOST_NAME=%HOST_NAME%"
set "CUR_HOST_GROUP_NAME=%CUR_HOST_GROUP_NAME%"
set "TELEGRAM_TOKEN=%TELEGRAM_TOKEN%"
set "CHAT_ID=%TELEGRAM_CHAT_ID%"
set "ALLOWED_USERS=%ALLOWED_USERS%"

:: === DEBUG (optional) ===
echo CUR_USER: [%CUR_USER%]
echo USER_GROUP: [%CUR_USER_GROUP%]
echo HOST_NUMBER: [%HOST_NUMBER%]
echo HOST_NAME: [%HOST_NAME%]
echo HOST_GROUP_NAME: [%CUR_HOST_GROUP_NAME%]
echo DATE: [%DATE%]
echo TIME: [%TIME%]

echo TELEGRAM_TOKEN: [%TELEGRAM_TOKEN%]
echo CHAT_ID: [%CHAT_ID%]
echo ALLOWED_USERS: [%ALLOWED_USERS%]

:: === FORMAT TIME WITHOUT SECONDS ===
for /f "tokens=1-2 delims=:," %%a in ("%TIME%") do set "FORMATTED_TIME=%%a:%%b"

:: === CHECK IF USER IS ALLOWED ===
set "SEND_NOTIFICATION=false"
for %%u in (%ALLOWED_USERS:,= %%) do (
    if /I "%%u"=="!CUR_USER!" (
        set "SEND_NOTIFICATION=true"
    )
)

:: === TELEGRAM NOTIFY ===
if "!SEND_NOTIFICATION!"=="true" (
    set "MESSAGE=User *%CUR_USER%* user group - *%CUR_USER_GROUP%* logOUT at *%HOST_NUMBER%* *%HOST_NAME%* *%CUR_HOST_GROUP_NAME%* on `%DATE% !FORMATTED_TIME!`"

    echo Sending: !MESSAGE!
    curl -s -X POST ^
        "https://api.telegram.org/bot%TELEGRAM_TOKEN%/sendMessage" ^
        -d "chat_id=%CHAT_ID%" ^
        -d "parse_mode=Markdown" ^
        -d "text=!MESSAGE!"
) else (
    echo User %CUR_USER% not allowed. No message sent.
)

endlocal
```


