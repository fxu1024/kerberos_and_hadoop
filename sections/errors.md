<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

# Error Messages to Fear

> The oldest and strongest emotion of mankind is fear, and the oldest and strongest kind of fear is fear of the unknown.

> *[Supernatural Horror in Literature](https://en.wikisource.org/wiki/Supernatural_Horror_in_Literature), HP Lovecraft, 1927.*


# OS/JVM Layer

Some of these are covered in Oracle's Troubleshooting Kerberos docs. This section just highlights some of the common causes, other causes that Oracle don't mention —and messages they haven't covered.

## Server not found in Kerberos database (7) 

* DNS is a mess and your machine does not know its own name.
* Your machine has a hostname, but it's not one there's an entry in the keytab for

## No valid credentials provided (Mechanism level: Illegal key size)]

Your JVM doesn't have the extended cryptography package and can't talk to the KDC. Switch to openjdk or go to your JVM supplier (Oracle, IBM) and download the JCE extension package, and install it in the hosts where you want Kerberos to work.

## No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt

This may appear in a stack trace starting with something like:

		javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]


Possible causes:

1. You aren't logged in via `kinit`.
2. You did specify a keytab but it isn't there or is somehow otherwise invalid
3. You don't have the Java Cryptography Extensions installed.

## Clock skew too great

		GSSException: No valid credentials provided (Mechanism level: Attempt to obtain new INITIATE credentials failed! (null)) . . . Caused by: javax.security.auth.login.LoginException: Clock skew too great

This comes from the clocks on the machines being too far out of sync. This can surface if you are doing Hadoop work on some VMs and have been suspending and resuming them; they've lost track of when they are. Reboot them.
If it's a physical cluster, make sure that your NTP daemons are pointing at the same NTP server, one that is actually reachable from the Hadoop cluster. And that the timezone settings of all the hosts are consistent.

## KDC has no support for encryption type

This crops up on the MiniKDC if you are trying to be clever about encryption types. It doesn't support many.

## Failure unspecified at GSS-API level (Mechanism level: Checksum failed)

1. Kerberos is very strict about hostnames and DNS
[http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization](http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization); 
2. Java 8 behaves differently from Java 6 & 7 here which can cause problems
[(HADOOP-11628](https://issues.apache.org/jira/browse/HADOOP-11628).

# Hadoop Web/REST APIs

## AuthenticationToken ignored
Surfaces in the HTTP logs of Hadoop REST/Web UIs:

	2015-06-26 13:49:02,239 WARN org.apache.hadoop.security.authentication.server.AuthenticationFilter: AuthenticationToken ignored: org.apache.hadoop.security.authentication.util.SignerException: Invalid signature

This means that the caller did not have the credentials to talk to a Kerberos-secured channel.

1. The caller may not be logged in.
2. The caller may have been logged in, but its kerberos token has expired, so its authentication headers are not considered valid any more.
3. The time limit of a negotiated token for the HTTP connection has expired. Here the calling app is expected to recognise this, discard its old token and renegotiate a new one. If the calling app is a YARN hosted service, then something should have been refreshing the tokens for you.
