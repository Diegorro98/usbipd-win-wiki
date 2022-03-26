## Running on Console

The first thing to do is to stop the Windows service and try running on a console.
As an Administrator, run:
```pwsh
sc stop usbipd
usbipd server
```
This will allow you to see a lot more logging on the console (see below).

Once you are done troubleshooting, you can stop the console process by pressing `Ctrl+C` and then start the service again:
```
sc start usbipd
```

## Logging

When running as a service, logging is done to the Application EventLog. All warnings and errors are logged;
at information level, only client device attach/detach is logged.

When running on the console, however, logging can be more extensive. For example, to get all available logging, run the server with:
```pwsh
usbipd server Logging:LogLevel:Default=Trace
```

For details on specifying logging configuration options, see <https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-6.0>.

Strictly speaking, it is also possible to change the logging levels for the service, but you will have to change the service startup command. This is possible, but not trivial.

Finally, you can also collect logs on a running service (without reconfiguring its startup command) using `dotnet trace` and `perfview.exe`, but that requires developer tools.

## USB capture

ℹ️ This requires version 2.2.0 or higher

Run the server with
```pwsh
usbipd server Logging:LogLevel:Default=Trace "usbipd:PcapNg:Path=C:\FULL\PATH\TO\FILE.pcapng"
```

This will capture all USB traffic in the PcapNg-format, see https://datatracker.ietf.org/doc/draft-tuexen-opsawg-pcapng/. The file can be analyzed with tools like WireShark. `usbipd-win` writes to the file such that WireShark can open it for reading even while data is being captured, flushing data about once every 5 seconds.

By default, all data is being captured. This may lead to very large capture files. For debugging, usually only the first 128 bytes (or so) of each URB is interesting. Therefore, you can truncate the captured URBs as follows:
```pwsh
usbipd server Logging:LogLevel:Default=Trace "usbipd:PcapNg:Path=C:\FULL\PATH\TO\FILE.pcapng" usbipd:PcapNg:SnapLength=128
```
The absolute minimum SnapLength is 64, but that will probably truncate too much for useful debugging.

:warning: Before posting capture files, please do realize that the capture file may contain sensitive information; for example, when capturing webcam traffic or mass storage devices. This is true even when truncating using SnapLength.

ℹ️ Capture files are usually highly compressible; please consider using ZIP before posting them. 