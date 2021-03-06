
# FluentFTP
FluentFTP is a fully managed FTP client that is designed to be easy to use and easy to extend. It supports file and directory listing, uploading and dowloading files and SSL/TLS connections. It can connect to Unix and Windows/IIS based FTP servers. This project is entirely developed in managed C#. All credits go to [J.P. Trosclair](https://github.com/jptrosclair) for developing and maintaining the library till 2016. FluentFTP is released under the permissive MIT License.

## Latest Release
- [Nuget](https://www.nuget.org/packages/FluentFTP)
- [FluentFTP 16.0.12](https://github.com/hgupta9/FluentFTP/releases)

## Features

- File and directory listing for [various formats](#file-listings)
  - UNIX long listings
  - Windows/IIS/DOS listings
  - Machine listings
  - Ability to add your own parsers
- Explicit and Implicit SSL connections for the control and data connections using .NET's `SslStream`
- Support for adding [client certificates](#client-certificates)
- Passive and Active data connections (PASV, EPSV, PORT and EPRT)
- Support for DrFTPD's (non-standard) PRET command
- Support for FTP Proxies (User@Host, HTTP 1.1)
- Returns `Stream` objects for reading or writing files on the server
- Attempts to make the FTP protocol safer with threads by cloning the control connection for file transfers (can be disabled)
  - Implements its own internal locking in an effort to keep transactions synchronized
- Implements the `IAsyncResult` pattern for almost all methods
- Includes support for non-standard hashing/checksum commands when supported by the server
- Easily issue any unsupported FTP command using the `Execute()` method with the exception of those requiring a data connection (file listings and transfers).
- Transaction logging using `TraceListeners` (passwords are automatically omitted)
- Examples for nearly all methods

## Submitting issues

Please use the [Issue tracker](https://github.com/hgupta9/FluentFTP/issues) to report bugs if you think you have found one. Aside from a good description of how to reproduce the problem, include the transaction log, the exact revision number of the library, and the exact version of the server OS and FTP software that you were connected to when the problem occurred.

## Documentation
### Stream Handling

FluentFTP returns a `Stream` object for file transfers. This stream **must** be properly closed when you are done. Do not leave it for the GC to cleanup otherwise you can end up with uncatchable exceptions, i.e., a program crash. The stream objects are actually wrappers around `NetworkStream` and `SslStream` which perform cleanup routines on the control connection when the stream is closed. These cleanup routines can trigger exceptions so it's vital that you properly dispose the objects when you are done, no matter what. A proper implementation should go along the lines of:

``````
try {
   using(Stream s = ftpClient.OpenRead()) {
       // perform your transfer
   }
}
catch(Exception) {
   // Typical exceptions here are IOException, SocketException, or a FtpCommandException
}
``````

The using statement above will ensure that `Dispose()` is called on the stream which in turn will call `Close()` so that the necessary cleanup routines on the control connection can be performed. If an exception is triggered you will have a chance to catch and handle it. Another valid approach might look like so:

``````
Stream s = null;

try {
    s = ftpClient.OpenRead();
    // perform transfer
}
finally {
     if(s != null)
           s.Close()
}
``````

The finally block above ensures that `Close()` is always called on the stream even if a problem occurs. When `Close()` is called any resulting exceptions can be caught and handled accordingly.

### Exception Handling during Dispose()

FluentFTP includes exception handling in key places where uncatchable exceptions could occur, such as the `Dispose()` methods. The problem is that part of the cleanup process involves closing out the internal sockets and streams. If `Dispose()` was called because of an exception and triggers another exception while trying to clean-up you could end up with an un-catchable exception resulting in an application crash. To deal with this `FtpClient.Dispose()` and `FtpSocketStream.Dispose()` are setup to handle `SocketException` and `IOException` and discard them. The exceptions are written to the FtpTrace `TraceListeners` for debugging purposes, in an effort to not hide important errors while debugging problems with the code.

The exception that propagates back to your code should be the root of the problem and any exception caught while disposing would be a side affect however while testing your project pay close attention to what's being logged via FtpTrace. See the Debugging example for more information about using `TraceListener` objects with FluentFTP.

### File Listings

Some of you may already be aware that RFC959 does not specify any particular format for file listings (LIST). As time has passed extensions have been added to address this problem. Here's what you need to know about the situation:

1. **UNIX File Listings :** UNIX style file listings are NOT reliable. Most FTP servers respond to LIST with a format that strongly resembles the output of ls -l on UNIX and UNIX-like operating systems. This format is difficult to parse and has shortcomings with date/time values in which assumptions have to made in order to try to guess an as accurate as possible value. FluentFTP provides a LIST parser but there is no guarantee that it will work right 100% of the time. You can add your own parser if it doesn't. See the examples project in the source code. You should include the `FtpListOption.Modify` flag for the most accurate modification dates (down to the second). MDTM will fail on directories on most but not all servers. An attempt is made by FluentFTP to get the modification time of directories using this command but do not be surprised if it fails. Modification times of directories should not be important most of the time. If you think they are you might want to reconsider your reasoning or upgrade the server software to one that supports MLSD (machine listings: the best option).

2. **DOS/IIS File Listings :** DOS style file listings (default IIS LIST response) are mostly supported. There is one issue where a file or directory that begins with a space won't be correctly parsed because of the arbitrary amount of spacing IIS throws in its directory listings. IIS can be configured to throw out UNIX style listings so if this is an issue for you, you might consider enabling that option. If you know a better way to parse the listing you can roll your own parser per the examples included with the downloads. If it works share it! It's also worth noting that date/times in DOS style listings don't include seconds. As mentioned above, if you pass the `FtpListOption.Modify` flag to GetListing() MDTM will be used to get the accurate (to the second) date/time however MDTM on directories does not work with IIS.

3. **Machine Listings :** FluentFTP prefers machine listings (MLST/MLSD) which are an extension added to the protocol. This format is reliable and is always used over LIST when the server advertises it in its FEATure list unless you override the behavior with `FtpListOption` flags. With machine listings you can expect a correct file size and modification date (UTC). If you run across a case that you are not it's very possible it's due to a bug in the machine listing parser and you should report the issue along with a sample of the file listing (see the debugging example in the source).

4. **Name Listings :** Name Listings (NLST) are the next best thing when machine listings are not available however they are MUCH slower than either LIST or MLSD. This is because NLST sends a list of objects in the directory and the server has to be queried for the rest of the information on file-by-file basis, such as the file size, the modification time and an attempt to determine if the object is a file or directory. Name listings can be forced using `FtpListOption` flags. The best way to handle falling back to NLST is to query the server features (FtpClient.Capabilities) for the FtpCapability.MLSD flag. If it's not there, then pass the necessary flags to GetListing() to force a name listing.

### Client Certificates

When you are using Client Certificates, be sure that:

1. You use X509Certificate2 objects, not the incomplete X509Certificate implementation.

2. You do not use pem certificates, use p12 instead. See this [Stack Overflow thread](http://stackoverflow.com/questions/13697230/ssl-stream-failed-to-authenticate-as-client-in-apns-sharp) for more information. If you get SPPI exceptions with an inner exception about an unexpected or badly formatted message, you are probably using the wrong type of certificate.

### Slow SSL Negotiation

FluentFTP uses `SslStream` under the hood which is part of the .NET framework. `SslStream` uses a feature of windows for updating root CA's on the fly, at least that's the way I understand it. These updates can cause a long delay in the certificate authentication process which can cause issues in FluentFTP related to the SocketPollInterval property used for checking for ungraceful disconnections between the client and server. This [MSDN Blog](http://blogs.msdn.com/b/alejacma/archive/2011/09/27/big-delay-when-calling-sslstream-authenticateasclient.aspx) covers the issue with SslStream and talks about how to disable the auto-updating of the root CA's.

The latest builds of FluentFTP log the time it takes to authenticate. If you think you are suffering from this problem then have a look at Examples\Debug.cs for information on retrieving debug information.

### Handling Ungraceful Interruptions in the Control Connection

FluentFTP uses `Socket.Poll()` to test for connectivity after a user-definable period of time has passed since the last activity on the control connection. When the remote host closes the connection there is no way to know, without triggering an exception, other than using `Poll()` to make an educated guess. When the connectivity test fails the connection is automatically re-established. This process helps a great deal in gracefully reconnecting however it does not eliminate your responsibility for catching IOExceptions related to an ungraceful interruption in the connection. Usually, maybe always, when this occurs the InnerException will be a SocketException. How you want to handle the situation from there is up to you.

```````
try {
    // ftpClient.SomeMethod();
}
catch(IOException e) {
    if(e.InnertException is SocketException) {
         // the control connection was interrupted
    }
}
```````

### XCRC / XMD5 / XSHA

XCRC, XMD5, and XSHA are non standard commands and to the best that I can tell contain no kind of formal specification. Support for them exists as extension methods in the FluentFTP.Extensions namespace as of the latest revision. They are not guaranteed to work and you are strongly encouraged to check the FtpClient.Capabilities flags for the respective flag (XCRC, XMD5, XSHA1, XSHA256, XSHA512) before calling these methods.

Support for the MD5 command as described [here](http://tools.ietf.org/html/draft-twine-ftpmd5-00#section-3.1) has also been added. Again, check for FtpFeature.MD5 before executing the command.

Experimental support for the HASH command has been added to FluentFTP. It supports retrieving SHA-1, SHA-256, SHA-512, and MD5 hashes from servers that support this feature. The returned object, FtpHash, has a method to check the result against a given stream or local file. You can read more about HASH in [this draft](http://tools.ietf.org/html/draft-bryan-ftpext-hash-02).

### Pipelining

If you just wanting to enable pipelining (in `FtpClient` and `FtpControlConnection`), set the `EnablePipelining` property to true. Hopefully this is all you need but it may not be. Some servers will drop the control connection if you flood it with a lot of commands. This is where the `MaxPipelineExecute` property comes into play. The default value here is 20, meaning that if you have 100 commands queued, 20 of the commands will be written to the underlying socket and 20 responses will be read, then the next 20 will be executed, and so forth until the command queue is empty. The value 20 is not a magic number, it's just the number that I deemed stable in most scenarios. If you increase the value, do so knowing that it could break your control connection.

### Pipelining your own Commands

Pipelining your own commands is not dependent on the `EnablePipelining` feature. The `EnablePipelining` property only applies to internal pipelining performed by FtpClient and FtpControlConnection. You can use the facilities for creating pipelines at your own discretion. 

If you need to cancel your pipeline in the middle of building your queue, you use the `CancelPipeline()` method. These methods are implemented in the `FtpControlConnection` class so people that are extending this class also have access to them. This feature is also used in `FtpClient.GetListing()` to retrieve last write times of the files in the listing when the LIST command is used. 

You don't need to worry about locking the command channel (`LockControlConnection()` or `UnlockControlConnection()`) because the code that handles executing the pipeline does so for you.

Here's a quick example:

`````
FtpClient cl = new FtpClient();

...

// initalize the pipeline
cl.BeginExecute();

// execute commands as normal
cl.Execute("foo");
cl.Execute("bar");
cl.Execute("baz");

...

// execute the queued commands
FtpCommandResult[] res = cl.EndExecute();

// check the result status of the commands
foreach(FtpCommandResult r in res) {
	if(!r.ResponseStatus) {
          // we have a failure
	}
}
``````

### Bulk Downloads

When doing a large number of transfers, one needs to be aware of some inherit issues with data streams. When a socket is opened and then closed, the socket is left in a linger state for a period of time defined by the operating system. The socket cannot reliably be re-used until the operating system takes it out of the TIME WAIT state. This matters because a data stream is opened when it's needed and closed as soon as that specific task is done:
- Download File
  - Open Data Stream
    - Read bytes
  - Close Data Stream

This is not a bug in FluentFTP. RFC959 says that EOF on stream mode transfers is signaled by closing the connection. On downloads and file listings, the sockets being used on the server will stay in the TIME WAIT state because the server closes the socket when it's done sending the data. On uploads, the client sockets will go into the TIME WAIT state because the client closes the connection to signal EOF to the server.

RFC959 defines another data mode called block that allows persistent data connections but it is not implemented by this library and will not be in the foreseeable future. Support for block transfers on the server side of things is not that common. I know IIS supports them however I cannot name a single other server that implements MODE B. I cannot justify making an already complicated process more so by adding in a feature that just isn't that likely to be used.