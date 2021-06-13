## Running on Console

The first thing to do is to stop the Windows service and try running on a console.
From an Administrator command prompt, run:
```
sc stop usbipd-win
"C:\Program Files\dorssel\usbipd-win\UsbIpServer.exe" server
```
or, if you are using PowerShell,
```
sc stop usbipd-win
& 'C:\Program Files\dorssel\usbipd-win\UsbIpServer.exe' server
```
This will allow you to see a lot more logging on the console (see below).

Once you are done troubleshooting, you can stop the console process by pressing `Ctrl+C` and then start the service again:
```
sc start usbipd-win
```

## Logging

When running as a service, logging is done to the Application EventLog. All warnings and errors are logged;
at information level, only client device attach/detach is logged.

When running on the console, however, logging can be more extensive. For example, to get all available logging, run the server with:
```
UsbIpServer.exe server Logging:LogLevel:Default=Trace
```

For details on specifying Logging configuration options, see <https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-5.0>.

Strictly speaking, it is also possible to change the logging levels for the service, but you will have to change the service startup command. This is possible, but not trivial.

Finally, you can also collect logs on a running service (without reconfiguring its startup command) using `dotnet trace` and `perfview.exe`, but that requires developer tools.