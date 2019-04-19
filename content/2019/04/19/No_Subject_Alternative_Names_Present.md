"No subject alternative names present" on New JDK
===

- date: 2019-04-19
- tags: ssl, ldaps, jdk, java

------------

# Abstract
In newer JDK implemetation(maybe JDK >= 8) you may suffer a CertificateException of "No subject alternative names present" when trying to create a TLS/SSL connection before you make Encrypted/HTTPS/LDAPS request.  

A new property named `com.sun.jndi.ldap.object.disableEndpointIdentification` has been introduced in new JDK versions (tested in JDK 11 compared to JDK 8) to decide whether make additional verification of endpoint name to certificate provided by server. The problem is that it will turn on by default. So if you suffering the problem you may try one of the solutions below:  

1. Set the property of `com.sun.jndi.ldap.object.disableEndpointIdentification` to false.
2. When you make plain TLS connections don't use `sslParameters.setEndpointIdentificationAlgorithm("HTTPS");` for your SSLSocket. (Not for LDAPS/HTTPS)
3. When you use `HttpsURLConnection` you can add your custom hostname verifier by `setDefaultHostnameVerifier`.
4. Sign the server certificate with SAN information in extension.(**Recommended**)


Please noted that it's totally a different issue compared to `Self Signed Certificate Issue`.

# First Meet It
One of our project choose `docker-java`[1] enabling operations on remote `dockerd`. Connection is encrypted by TLS 1.2.  

Everything goes right before we switch the commend implementation from `Jersey` to `NettyDockerCmdExecFactory`. Because `ExeStartCmd` doesn't support stdin stream when using `Jersey` we switch into netty implementation.  

Then we meet the exception `No subject alternative names present` and can't communicate to server anymore. We are quite sure the connection can be established and encrypted smoothly when using jersey backend.  

After hours' research we found that the exception was mainly targeting to the unsuitable certificate the server gave out. SSLContext will do additional hostname(or calling endpoint) verification after passed through `TrustManager[]`. Logic is located in `sun.security.util.HostnameChecker` with two form: `IP` and `Domain` matching with certificate(In this case it's IP).  

Checker will fetch `SAN` (Subject Alternative Names) from extension of certificate and compare it with the server actually ip address. Extension structure is defined with version 3 of X509 but our certificate is generated by some old scripts in version 1 and only have `CN=xxx.xxx.xxx.xxx` in Subject field. So the logic will fail and throw the exception.

Because the `jersey` backend works correctly so I guess it's not a required verification. Working a little more I figure out the key statement:  

```java
sslParameters.setEndpointIdentificationAlgorithm("HTTPS");
```

The netty backend adds the specific parameter manually which will enable the additional verification.  

There are two sulutions:

1. Remove the line of `setEndpointIdentificationAlgorithm`
2. Regenerate the certificate using version 3. Add the SAN information required in extension part.(Recommended)

Knowing the reasons we regenerated the certificate following the rules.  

# Meet It Again
Things seem like going well but the same error happens again two days later. The affected logic is the connection of LDAP(Using SSL). It's strange because it works well before.  

Fortunately we can debug it in source code level (package `sun.security`) because we change the JDK to version 11 days ago. After some more time we arrived:  

`sun.security.util.HostnameChecker`  
```java
    private static void matchIP(String expectedIP, X509Certificate cert)
            throws CertificateException {
        Collection<List<?>> subjAltNames = cert.getSubjectAlternativeNames();
        if (subjAltNames == null) {
            throw new CertificateException
                                ("No subject alternative names present");
        }
        for (List<?> next : subjAltNames) {
            // For IP address, it needs to be exact match
            if (((Integer)next.get(0)).intValue() == ALTNAME_IP) {
                String ipAddress = (String)next.get(1);
                if (expectedIP.equalsIgnoreCase(ipAddress)) {
                    return;
                } else {
                    // compare InetAddress objects in order to ensure
                    // equality between a long IPv6 address and its
                    // abbreviated form.
                    try {
                        if (InetAddress.getByName(expectedIP).equals(
                                InetAddress.getByName(ipAddress))) {
                            return;
                        }
                    } catch (UnknownHostException e) {
                    } catch (SecurityException e) {}
                }
            }
        }
        throw new CertificateException("No subject alternative " +
                        "names matching " + "IP address " +
                        expectedIP + " found");
    }
```

Oh it enters into the `matchIP` too like we were in the first case. But the stack may be different. After more debugging:  

`sun.security.ssl.SSLContextImpl`
```java
    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType,
            SSLEngine engine) throws CertificateException {
        tm.checkServerTrusted(chain, authType);
        checkAdditionalTrust(chain, authType, engine, false);
    }

    private void checkAdditionalTrust(X509Certificate[] chain, String authType,
                Socket socket, boolean isClient) throws CertificateException {
        if (socket != null && socket.isConnected() &&
                                    socket instanceof SSLSocket) {

            SSLSocket sslSocket = (SSLSocket)socket;
            SSLSession session = sslSocket.getHandshakeSession();
            if (session == null) {
                throw new CertificateException("No handshake session");
            }

            // check endpoint identity
            String identityAlg = sslSocket.getSSLParameters().
                                        getEndpointIdentificationAlgorithm();
            if (identityAlg != null && identityAlg.length() != 0) {
                String hostname = session.getPeerHost();
                X509TrustManagerImpl.checkIdentity(
                                    hostname, chain[0], identityAlg);
            }

            // try the best to check the algorithm constraints
            AlgorithmConstraints constraints;
            if (ProtocolVersion.useTLS12PlusSpec(session.getProtocol())) {
                if (session instanceof ExtendedSSLSession) {
                    ExtendedSSLSession extSession =
                                    (ExtendedSSLSession)session;
                    String[] peerSupportedSignAlgs =
                            extSession.getLocalSupportedSignatureAlgorithms();

                    constraints = new SSLAlgorithmConstraints(
                                    sslSocket, peerSupportedSignAlgs, true);
                } else {
                    constraints =
                            new SSLAlgorithmConstraints(sslSocket, true);
                }
            } else {
                constraints = new SSLAlgorithmConstraints(sslSocket, true);
            }

            checkAlgorithmConstraints(chain, constraints, isClient);
        }
    }
```

It shows that the verification is relying the value of `sslSocket.getSSLParameters().getEndpointIdentificationAlgorithm();`.  

But we don't set it to any value when creating the ssl socket and printing its value also shows to `null`:  

```java
    @Override
    public Socket createSocket(String string, int i) throws IOException, UnknownHostException {
        SSLSocketImpl ssl = (SSLSocketImpl)socketFactory.createSocket(string, i);
        SSLParameters param = ssl.getSSLParameters();
        System.out.println(param.getEndpointIdentificationAlgorithm());
        return ssl;
    }
```

Console output:
```
null
```

So who sets it? After deeper tracing I arrived:  

`com.sun.jndi.ldap.Connection`
```java
    private Socket createSocket(String host, int port, String socketFactory,
            int connectTimeout) throws Exception {

        Socket socket = null;

        if (socketFactory != null) {

            // create the factory

            @SuppressWarnings("unchecked")
            Class<? extends SocketFactory> socketFactoryClass =
                (Class<? extends SocketFactory>)Obj.helper.loadClass(socketFactory);
            Method getDefault =
                socketFactoryClass.getMethod("getDefault", new Class<?>[]{});
            SocketFactory factory = (SocketFactory) getDefault.invoke(null, new Object[]{});

            // create the socket

            if (connectTimeout > 0) {

                InetSocketAddress endpoint =
                        createInetSocketAddress(host, port);

                // unconnected socket
                socket = factory.createSocket();

                if (debug) {
                    System.err.println("Connection: creating socket with " +
                            "a timeout using supplied socket factory");
                }

                // connected socket
                socket.connect(endpoint, connectTimeout);
            }

            // continue (but ignore connectTimeout)
            if (socket == null) {
                if (debug) {
                    System.err.println("Connection: creating socket using " +
                        "supplied socket factory");
                }
                // connected socket
                socket = factory.createSocket(host, port);
            }
        } else {

            if (connectTimeout > 0) {

                    InetSocketAddress endpoint = createInetSocketAddress(host, port);

                    socket = new Socket();

                    if (debug) {
                        System.err.println("Connection: creating socket with " +
                            "a timeout");
                    }
                    socket.connect(endpoint, connectTimeout);
            }

            // continue (but ignore connectTimeout)

            if (socket == null) {
                if (debug) {
                    System.err.println("Connection: creating socket");
                }
                // connected socket
                socket = new Socket(host, port);
            }
        }

        // For LDAP connect timeouts on LDAP over SSL connections must treat
        // the SSL handshake following socket connection as part of the timeout.
        // So explicitly set a socket read timeout, trigger the SSL handshake,
        // then reset the timeout.
        if (socket instanceof SSLSocket) {
            SSLSocket sslSocket = (SSLSocket) socket;
            if (!IS_HOSTNAME_VERIFICATION_DISABLED) {
                SSLParameters param = sslSocket.getSSLParameters();
                param.setEndpointIdentificationAlgorithm("LDAPS");
                sslSocket.setSSLParameters(param);
            }
            if (connectTimeout > 0) {
                int socketTimeout = sslSocket.getSoTimeout();
                sslSocket.setSoTimeout(connectTimeout); // reuse full timeout value
                sslSocket.startHandshake();
                sslSocket.setSoTimeout(socketTimeout);
            }
        }
        return socket;
    }
```

So the last block rely to the value of `IS_HOSTNAME_VERIFICATION_DISABLED`, and:

```java
    private static final boolean IS_HOSTNAME_VERIFICATION_DISABLED
            = hostnameVerificationDisabledValue();

    private static boolean hostnameVerificationDisabledValue() {
        PrivilegedAction<String> act = () -> System.getProperty(
                "com.sun.jndi.ldap.object.disableEndpointIdentification");
        String prop = AccessController.doPrivileged(act);
        if (prop == null) {
            return false;
        }
        return prop.isEmpty() ? true : Boolean.parseBoolean(prop);
    }
```

Things are clear. By default the property `com.sun.jndi.ldap.object.disableEndpointIdentification` is empty and the value will be set to `true` which means additional verification will be taken. But why the logic worked correctly before?  

I turn to `Open JDK 7/8` for source code reference and find they are lack of the final piece of code:  

```java
        // For LDAP connect timeouts on LDAP over SSL connections must treat
        // the SSL handshake following socket connection as part of the timeout.
        // So explicitly set a socket read timeout, trigger the SSL handshake,
        // then reset the timeout.
        if (socket instanceof SSLSocket) {
            SSLSocket sslSocket = (SSLSocket) socket;
            if (!IS_HOSTNAME_VERIFICATION_DISABLED) {
                SSLParameters param = sslSocket.getSSLParameters();
                param.setEndpointIdentificationAlgorithm("LDAPS");
                sslSocket.setSSLParameters(param);
            }
            if (connectTimeout > 0) {
                int socketTimeout = sslSocket.getSoTimeout();
                sslSocket.setSoTimeout(connectTimeout); // reuse full timeout value
                sslSocket.startHandshake();
                sslSocket.setSoTimeout(socketTimeout);
            }
        }
        return socket;
    }
```

That's the key point.

# Conclusion
1. When you get `No subject alternative names present` you should make sure the server certificate is in version 3 and has the proper SAN field value in extension.
2. Related project upgraded from order JDK to JDK11 (JDK9 has not been tested in current case) may hit the problem. You can try to disable it by set property `com.sun.jndi.ldap.object.disableEndpointIdentification` to `false`.
3. For manually created ssl connections you can enable/disable by setting the parameter `endpointIdentificationAlgorithm` of SSLSocket. Its value can only be `SSL`/`LDAPS`. `SSL` is for all the non-ldap connections.


It's recommended that regenerated the issued certificate using the right properties.

