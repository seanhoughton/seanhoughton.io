---
alias: /2011/07/thrift-protocols-ajax-and-language-support/index.html
date: "2011-07-18T00:00:00Z"
tags:
- thrift
- c#
thumbnail: /system/images/code-thumb.png
title: Thrift Protocols, AJAX, and Language Support
---
Two of the major strengths of Thrift are its support for a wide range of languages as well as its collection of available protocols. However, not every protocol is available for every language and not all protocols perform the same. I’ve spent a little time researching these issues and this is a quick summary of the results.

### Language Support

![](Thrift-Protocol-Compatibility.png)

Thrift supports a wide range of languages, but the support is not uniform. It’s clear that C++ is the primary language given that it supports all the protocols.

You can always extend thrift and build your own implementation using whatever language you want, but this is what comes stock.

```c#
    // based on code in https://issues.apache.org/jira/browse/THRIFT-322
    using System;
    using System.Web;
    using Telemetry.Server.Thrift;
    using Thrift;
    using Thrift.Protocol;
    using Thrift.Transport;

    namespace Shell.Web
    {
        public class ThriftRequestHandler
        {
            private readonly TProcessor _processor;
            private readonly TProtocolFactory _inputProtocolFactory;
            private readonly TProtocolFactory _outputProtocolFactory;

            public ThriftRequestHandler(TProcessor processor, TProtocolFactory inputProtocolFactory, TProtocolFactory outputProtocolFactory)
            {
                _processor = processor;
                _inputProtocolFactory = inputProtocolFactory;
                _outputProtocolFactory = outputProtocolFactory;
            }

            public void ProcessRequest(HttpContext context)
            {
                context.Response.ContentType = "application/x-thrift";
                context.Response.ContentEncoding = System.Text.Encoding.UTF8;

                var transport = new TStreamTransport(context.Request.InputStream, context.Response.OutputStream);
                try
                {
                    var inputProtocol = _inputProtocolFactory.GetProtocol(transport);
                    var outputProtocol = _outputProtocolFactory.GetProtocol(transport);

                    while (_processor.Process(inputProtocol, outputProtocol))
                    {
                        
                    }
                }
                catch (TTransportException)
                {
                    // Client died, just move on
                }
                catch (TApplicationException tx)
                {
                    Console.Error.Write(tx);
                }
                catch (Exception x)
                {
                    Console.Error.Write(x);
                }

                transport.Close();
            }
        }

        public class Thrift : IHttpHandler
        {
            private readonly ThriftRequestHandler _jsonHandler =
                new ThriftRequestHandler(
                    new TelemetryThriftService.Processor(new TelemetryHandler()), new TJSONProtocol.Factory(), new TJSONProtocol.Factory());

            private readonly ThriftRequestHandler _binaryHandler =
                new ThriftRequestHandler(
                    new TelemetryThriftService.Processor(new TelemetryHandler()), new TBinaryProtocol.Factory(), new TBinaryProtocol.Factory());
            
            public void ProcessRequest(HttpContext context)
            {
                if(string.Compare(context.Request.Params["protocol"], "json", StringComparison.InvariantCultureIgnoreCase) == 0)
                {
                    _jsonHandler.ProcessRequest(context);
                }
                else
                {
                    _binaryHandler.ProcessRequest(context);
                }
            }

            public bool IsReusable
            {
                get { return true; }
            }
        }
    }
```

Note: serving JSON to web pages that use the XMLHTTPRequest object require a server that handles the “OPTION” request if you’re using a cross-site service. None of the included HTTP transports currently handle this request type. This means the simple, autogenerated C++ skeleton apps wont work with AJAX clients. I’ve added support for this in the C++ template in my fork of thrift on GitHub. If you expose your service on the same host that’s serving web pages you don’t need to worry about it because no cross-site communication is happening.

### Performance

The JVM-Serializers project on GitHub provides detailed performance measurements for Thrift and compares them with other RPC frameworks such as Google’s protobuf. It doesn’t cover all of the protocols available in Thrift, but it’s a good source for comparing the binary protocol against other frameworks.