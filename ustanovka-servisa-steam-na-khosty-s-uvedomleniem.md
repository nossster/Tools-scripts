---
order: 2
title: "Установка сервиса Steam на хосты с уведомлением в телеграмм\_"
---

```
@echo off
setlocal

:: --- НАСТРОЙКИ (БЕЗ слэшей в конце) ---
set "BOT_TOKEN=you_bot_token"
set "CHAT_ID=you_chat_id"
set "SOURCE_PATH=D:\Tmp\Steam"
set "STEAM_PATH=%ProgramFiles(x86)%\Common Files\Steam"
:: --------------------------------------

:: 1. Проверка существования ИСТОЧНИКА (без лишних слэшей)
if not exist "%SOURCE_PATH%" (
    echo [ERROR] Source folder NOT FOUND at %SOURCE_PATH%
    set "MSG=ERROR on %computername%: Source folder '%SOURCE_PATH%' not found!"
    goto :SEND_AND_EXIT
)

:: 2. Проверка: нужно ли копировать (если папка назначения пуста или нет файла)
if exist "%STEAM_PATH%\steamservice.exe" (
    echo Service file already exists. Skipping copy.
    goto :INSTALL
)

:: 3. Копирование (Обратите внимание на кавычки вокруг путей)
echo Copying files from %SOURCE_PATH% to %STEAM_PATH%...
xcopy "%SOURCE_PATH%\*" "%STEAM_PATH%\" /s /e /i /y 

:: Проверка успешности копирования
if %errorlevel% neq 0 (
    set "MSG=ERROR on %computername%: xcopy failed with code %errorlevel%"
    goto :SEND_AND_EXIT
)

:INSTALL
:: 4. Запуск установки
echo Running install...
"%STEAM_PATH%\steamservice.exe" /install
set "MSG=Script finished on %computername%. Steam service installed/updated."

:SEND_AND_EXIT
:: 5. Отправка уведомления
curl -s -X POST "https://api.telegram.org/bot%BOT_TOKEN%/sendMessage" -d "chat_id=%CHAT_ID%&text=%MSG%" >nul

echo %MSG%
endlocal
timeout /t 15
```