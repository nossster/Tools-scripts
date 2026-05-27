---
order: 3
title: Clean UP script
---

Этот BAT-скрипт -- это утилита очистки системы от пользовательских данных, аккаунтов и следов популярных лаунчеров/игровых клиентов. По сути, он делает «жёсткий сброс» пользовательской среды на Windows.

## Общая логика

-  Проверяет, запущен ли с правами администратора.

-  Создаёт лог-файл `C:\cleanup.log`.

-  Останавливает процессы и сервисы.

-  Удаляет данные авторизации (логины, токены, конфиги).

-  Чистит пользовательские папки.

-  Очищает корзину.

---

## Остановка процессов

Принудительно завершает процессы:

-  Браузеры и лаунчеры: Chrome, EpicGamesLauncher, EA Desktop, Steam, [Battle.net](http://Battle.net) и др.

-  Игровые клиенты: FACEIT, GameCenter, Innova (4game), WGC и т.д.

Также останавливает сервис:

-  `Steam Client Service`

---

## Очистка Steam

-  Читает путь установки Steam из реестра.

-  Удаляет файл:

   -  `loginusers.vdf` (хранит список аккаунтов и автологин)

---

## Удаление учетных данных приложений

Удаляет или чистит данные из:

-  EA Desktop (`cookie.ini`)

-  GameCenter (частично очищает `GameCenter.ini`, убирая поля логина)

-  Epic Games (`GameUserSettings.ini`)

-  Steam (`%localappdata%\Steam\*`)

-  Lesta / Wargaming (`user_info.xml`)

-  [Battle.net](http://Battle.net) (`*.config`)

-  Chrome профиль (`User Data\*`)

-  Discord, FACEIT (полные папки)

-  Реестр:

   -  `HKCU\SOFTWARE\FACEIT\Client -> token`

-  4game / Innova:

   -  Local Storage (LevelDB)

   -  `accounts-live.db` в ProgramData

Особенность:

-  Для GameCenter не удаляет файл полностью, а фильтрует чувствительные строки (логин/UID), оставляя остальное.

---

## Очистка пользовательских папок

Полностью очищает содержимое:

-  `%localappdata%\Temp`

-  `%localappdata%\CrashDumps`

-  `%USERPROFILE%\Downloads`

-  `%USERPROFILE%\Pictures`

-  `%USERPROFILE%\Videos`

-  `%USERPROFILE%\Music`

Удаляются:

-  все файлы

-  все подпапки

---

## Очистка корзины

-  Удаляет `C:\$Recycle.bin` (для всех пользователей)

---

## Логирование

Все действия пишутся в:

-  `C:\cleanup.log`

С таймстампами:

-  что удалено

-  что не найдено

-  где были ошибки

---

## Итог

Скрипт:

-  разлогинивает пользователя из всех популярных лаунчеров

-  удаляет пользовательские данные приложений

-  чистит временные и личные папки

-  освобождает место

-  оставляет систему в состоянии «почти как новая с точки зрения пользователя»

---

Если коротко: это агрессивный cleanup + деавторизация всех игровых/социальных клиентов.

Если хочешь -- могу разобрать, где тут потенциально опасные места (там есть пара реально жёстких моментов типа удаления Downloads и Chrome профиля).

```
@echo on
setlocal EnableDelayedExpansion
:: Check for administrative privileges
net session >nul 2>&1
if %errorlevel% NEQ 0 (
    echo This script requires administrative privileges. Please run as Administrator.
    pause
    exit /b
)
set "Version=0.3.8"
title Cleanup Utility - %Version%
:: Log file in the root of the system drive (e.g. C:\cleanup.log)
set "LogFile=%systemdrive%\cleanup.log"
if exist "%LogFile%" del /f /q "%LogFile%"
echo Cleanup Utility
echo Deleting user accounts and settings...
call :WriteLog "Cleanup Utility started"
call :WriteLog "Script execution started"
:: --- Stop processes (including Innova.Launcher) ---
for %%P in (chrome EpicGamesLauncher EpicWebHelper EADesktop EABackgroundService FACEIT GameCenter discord "Battle.net" Steam steamwebhelper lgc wgc Innova.Launcher) do (
    taskkill /F /IM "%%P.exe" >nul 2>&1
    if errorlevel 1 (
         echo Error stopping process [%%P].
         call :WriteLog "Error stopping process [%%P]."
    ) else (
         echo Process [%%P] stopped.
         call :WriteLog "Process [%%P] stopped."
    )
)
net stop "Steam Client Service" >nul 2>&1
if errorlevel 1 (
    echo Error stopping service 'Steam Client Service'.
    call :WriteLog "Error stopping service 'Steam Client Service'."
) else (
    echo Service 'Steam Client Service' stopped.
    call :WriteLog "Service 'Steam Client Service' stopped."
)
:: --- Clean Steam credentials ---
for /f "tokens=2,*" %%A in ('reg query "HKLM\SOFTWARE\Wow6432Node\Valve\Steam" /v InstallPath 2^>nul ^| find "REG_SZ"') do set "SteamPath=%%B"
if defined SteamPath (
    if exist "%SteamPath%\config\loginusers.vdf" (
        del /f /q "%SteamPath%\config\loginusers.vdf" >nul 2>&1
        if errorlevel 1 (
            echo Error deleting Steam credentials.
            call :WriteLog "Error deleting Steam credentials."
        ) else (
            echo Steam credentials deleted.
            call :WriteLog "Steam credentials deleted."
        )
    ) else (
        echo Steam credentials not found.
        call :WriteLog "Steam credentials not found."
    )
) else (
    echo Steam not found.
    call :WriteLog "Steam not found."
)
timeout /t 1 /nobreak >nul
:: --- Delete application credentials (including 4Game and registry modification) ---
set "Cred0=%localappdata%\Electronic Arts\EA Desktop\cookie.ini"
set "Cred1=%localappdata%\GameCenter\GameCenter.ini"
set "Cred2=%localappdata%\EpicGamesLauncher\Saved\Config\Windows\GameUserSettings.ini"
set "Cred3=%localappdata%\Steam\*"
set "Cred4=%appdata%\Lesta\GameCenter\user_info.xml"
set "Cred5=%appdata%\Wargaming.net\GameCenter\user_info.xml"
set "Cred6=%appdata%\Battle.net\*.config"
set "Cred7=%localappdata%\Google\Chrome\User Data\*"
set "Cred8=%appdata%\discord\*"
set "Cred9=%appdata%\FACEIT\*"
set "Cred10=REG:HKEY_CURRENT_USER\SOFTWARE\FACEIT\FACEIT Client,token"
set "Cred11=%localappdata%\Innova\4game_cis\CEF\global\Local Storage\leveldb"
set "Cred12=C:\ProgramData\Innova\4game_cis\accounts-live.db"
for /L %%i in (0,1,12) do (
    if %%i EQU 1 (
         echo Processing custom GameCenter cleanup...
         call :WriteLog "Processing custom GameCenter cleanup for Cred1."
         pushd "%UserProfile%\AppData\Local\GameCenter"
         dir >nul 2>&1
         type GameCenter.ini | findstr /i /v "<MyComUserLogin> <MyComUserMagic2> <MyComUserMagic4> <MyComUserUid> <CurrentUserName> <CurrentUserNick>" > temp.txt
         del /f /q GameCenter.ini >nul 2>&1
         rename temp.txt GameCenter.ini >nul 2>&1
         if errorlevel 1 (
              echo Error processing GameCenter.ini.
              call :WriteLog "Error processing GameCenter.ini in Cred1."
         ) else (
              echo Custom GameCenter.ini processed.
              call :WriteLog "Custom GameCenter.ini processed in Cred1."
         )
         popd
    ) else (
         call set "CurrentCred=%%Cred%%i%%"
         if "!CurrentCred:~0,4!"=="REG:" (
             set "regData=!CurrentCred:~4!"
             for /f "tokens=1,2 delims=," %%A in ("!regData!") do (
                 set "regKey=%%A"
                 set "regValue=%%B"
             )
             reg delete "!regKey!" /v !regValue! /f >nul 2>&1
             if !ERRORLEVEL! EQU 0 (
                 echo Registry entry for !regValue! deleted.
                 call :WriteLog "Registry entry for !regValue! deleted."
             ) else (
                 echo Registry entry for !regValue! not found or error occurred.
                 call :WriteLog "Registry entry for !regValue! not found or error occurred."
             )
         ) else (
             if exist "!CurrentCred!" (
                 del /f /q "!CurrentCred!" >nul 2>&1
                 if errorlevel 1 (
                     echo Error deleting file "!CurrentCred!".
                     call :WriteLog "Error deleting file '!CurrentCred!'."
                 )
                 rmdir /s /q "!CurrentCred!" >nul 2>&1
                 if errorlevel 1 (
                     echo Error removing directory "!CurrentCred!".
                     call :WriteLog "Error removing directory '!CurrentCred!'."
                 )
                 echo Data at !CurrentCred! deleted.
                 call :WriteLog "Data at !CurrentCred! deleted."
             ) else (
                 echo Data not found: !CurrentCred!
                 call :WriteLog "Data not found: !CurrentCred!"
             )
         )
    )
)
timeout /t 1 /nobreak >nul
:: --- Clean system folders ---
set "Sys0=%localappdata%\Temp"
set "Sys1=%localappdata%\CrashDumps"
set "Sys2=%userprofile%\Downloads"
set "Sys3=%userprofile%\Pictures"
set "Sys4=%userprofile%\Videos"
set "Sys5=%userprofile%\Music"
for /L %%i in (0,1,5) do (
    call set "CurrentPath=%%Sys%%i%%"
    if exist "!CurrentPath!" (
        del /f /q "!CurrentPath!\*" >nul 2>&1
        if errorlevel 1 (
            echo Error deleting files in folder "!CurrentPath!".
            call :WriteLog "Error deleting files in folder '!CurrentPath!'."
        )
        for /d %%D in ("!CurrentPath!\*") do (
            rmdir /s /q "%%D" >nul 2>&1
            if errorlevel 1 (
                echo Error removing subdirectory "%%D" in folder "!CurrentPath!".
                call :WriteLog "Error removing subdirectory '%%D' in folder '!CurrentPath!'."
            )
        )
        echo Folder !CurrentPath! cleaned.
        call :WriteLog "Folder !CurrentPath! cleaned."
    ) else (
        echo Folder not found: !CurrentPath!
        call :WriteLog "Folder not found: !CurrentPath!"
    )
)
timeout /t 1 /nobreak >nul
:: --- Clear Recycle Bin ---
rd /s /q "%systemdrive%\$Recycle.bin" >nul 2>&1
if errorlevel 1 (
    echo Error clearing Recycle Bin.
    call :WriteLog "Error clearing Recycle Bin."
) else (
    echo Recycle Bin cleared.
    call :WriteLog "Recycle Bin cleared."
)
call :WriteLog "Script execution completed."
echo Script execution completed.
pause
endlocal
goto :EOF
:WriteLog
set "timestamp=[%date% %time:~0,8%]"
echo %timestamp% %~1 >> "%LogFile%"
goto :EOF
```