# URL syntax and their use in curl

## Specifications

The official "URL syntax" is primarily defined in these two different
specifications:

 - [RFC 3986](https://tools.ietf.org/html/rfc3986) (although URL is called "URI" in there)
 - [The WHATWG URL Specification](https://url.spec.whatwg.org/)

RFC 3986 is the earlier one, and curl has always tried to adhere to that one
(since it shipped in January 2005).

The WHATWG URL spec was written later, is incompatible with the RFC 3986 and
changes over time.

## Variations

URL parsers as implemented in browsers, libraries and tools usually opt to
support one of the mentioned specifications. Bugs, differences in
interpretations and the moving nature of the WHATWG spec does however make it
very unlikely that multiple parsers treat URLs the exact same way!

## Security

Due to the inherent differences between URL parser implementations, it is
considered a security risk to mix different implementations and assume the
same behavior!

For example, if you use one parser to check if a URL uses a good host name or
the correct auth field, and then pass on that same URL to a *second* parser,
there will always be a risk it treats the same URL differently. There is no
right and wrong in URL land, only differences of opinions.

libcurl offers a separate API to its URL parser for this reason, among others.

Applications may at times find it convenient to allow users to specify URLs
for various purposes and that string would then end up fed to curl. Getting a
URL from an external untrusted party and using it with curl brings several
security concerns:

1. If you have an application that runs as or in a server application, getting
   an unfiltered URL can trick your application to access a local resource
   instead of a remote resource. Protecting yourself against localhost accesses is very
   hard when accepting user provided URLs.

2. Such custom URLs can access other ports than you planned as port numbers
   are part of the regular URL format. The combination of a local host and a
   custom port number can allow external users to play tricks with your local
   services.

3. Such a URL might use other schemes than you thought of or planned for.

## "RFC3986 plus"

curl recognizes a URL syntax that we call "RFC 3986 plus". It is grounded on
the well established RFC 3986 to make sure previously written command lines and
curl using scripts will remain working.

curl's URL parser allows a few deviations from the spec in order to
inter-operate better with URLs that appear in the wild.

### spaces

In particular `Location:` headers that indicate to the client where a resource
has been redirected to, sometimes contain spaces. This is a violation of RFC
3986 but is fine in the WHATWG spec. curl handles these by re-encoding them to
`%20`.

### non-ASCII

Byte values in a provided URL that are outside of the printable ASCII range
are percent-encoded by curl.

### multiple slashes

An absolute URL always starts with a "scheme" followed by a colon. For all the
schemes curl supports, the colon must be followed by two slashes according to
RFC 3986 but not according to the WHATWG spec - which allows one to infinity
amount.

curl allows one, two or three slashes after the colon to still be considered a
valid URL.

### "scheme-less"

curl supports "URLs" that do not start with a scheme. This is not supported by
any of the specifications. This is a shortcut to entering URLs that was
supported by browsers early on and has been mimicked by curl.

Based on what the host name starts with, curl will "guess" what protocol to
use:

 - `ftp.` means FTP
 - `dict.` means DICT
 - `ldap.` means LDAP
 - `imap.` means IMAP
 - `smtp.` means SMTP
 - `pop3.` means POP3
 - all other means HTTP

### globbing letters

The curl command line tool supports "globbing" of URLs. It means that you can
create ranges and lists using `[N-M]` and `{one,two,three}` sequences. The
letters used for this (`[]{}`) are reserved in RFC 3986 and can therefore not
legitimately be part of such a URL.

They are however not reserved or special in the WHATWG specification, so
globbing can mess up such URLs. Globbing can be turned off for such occasions
(using `--globoff`).

# URL syntax details

A URL may consist of the following components - many of them are optional:

    [scheme][divider][userinfo][hostname][port number][path][query][fragment]

Each component is separated from the following component with a divider
character or string.

For example, this could look like:

    http://user:password@www.example.com:80/index.hmtl?foo=bar#top

## Scheme

The scheme specifies the protocol to use. A curl build can support a few or
many different schemes. You can limit what schemes curl should acccept.

curl supports the following schemes on URLs specified to transfer. They are
matched case insensitvely:

`dict`, `file`, `ftp`, `ftps`, `gopher`, `http`, `https`, `imap`, `imaps`,
`ldap`, `ldaps`, `mqtt`, `pop3`, `pop3s`, `rtmp`, `rtmpe`, `rtmps`, `rtmpt`,
`rtmpte`, `rtmpts`, `rtsp`, `smb`, `smbs`, `smtp`, `smtps`, `telnet`, `tftp`

When the URL is specified to identify a proxy, curl recognizes the following
schemes:

`http`, `https`, `socks4`, `socks4a`, `socks5`, `socks5h`, `socks`

## Userinfo

The userinfo field can be used to set user name and password for
authentication purposes in this transfer. The use of this field is discouraged
since it often means passing around the password in plain text and is thus a
security risk.

URLs for IMAP, POP3 and SMTP also support *login options* as part of the
userinfo field. They're provided as a semicolon after the password and then
the options.

## Hostname

The hostname part of the URL contains the address of the server that you want
to connect to. This can be the fully qualified domain name of the server, the
local network name of the machine on your network or the IP address of the
server or machine represented by either an IPv4 or IPv6 address (within
brackets). For example:

    http://www.example.com/

    http://hostname/

    http://192.168.0.1/

    http://[2001:1890:1112:1::20]/

If curl was built with International Domain Name (IDN) support, it can also
handle host names using non-ASCII characters.

## Port number

If there's a colon after the hostname, that should be followed by the port
number to use. 1 - 65535. curl also supports a blank port number field - but
only if the URL starts with a scheme.

# Scheme specific behaviors

## FTP

The path part of an FTP request specifies the file to retrieve and from which
directory. If the file part is omitted then libcurl downloads the directory
listing for the directory specified. If the directory is omitted then the
directory listing for the root / home directory will be returned.

FTP servers typically put the user in its "home directory" after login, which
then differs between users. To explicitly specify the root directory of an FTP
server start the path with double slash `//` or `/%2f` (2F is the hexadecimal
value of the ascii code for the slash).

## FILE

When a `FILE://` URL is accessed on Windows systems, it can be crafted in a
way so that Windows attempts to connect to a (remote) machine when curl wants
to read or write such a path.

curl only allows the hostname part of a FILE URL to be one out of these three
alternatives: `localhost`, `127.0.0.1` or blank ("", zero characters).
Anything else will make curl fail to parse the URL.

On Windows, curl accepts that the FILE URL's path starts with a "drive
letter". That's a single letter `a` to `z` followed by a colon or a pipe
character (`|`).

## IMAP

The path part of an IMAP request not only specifies the mailbox to list or
select, but can also be used to check the `UIDVALIDITY` of the mailbox, to
specify the `UID`, `SECTION` and `PARTIAL` octets of the message to fetch and
to specify what messages to search for.

A top level folder list:

    imap://user:password@mail.example.com

A folder list on the user's inbox:

    imap://user:password@mail.example.com/INBOX

Select the user's inbox and fetch message with uid = 1:

    imap://user:password@mail.example.com/INBOX/;UID=1

Select the user's inbox and fetch the first message in the mail box:

    imap://user:password@mail.example.com/INBOX/;MAILINDEX=1

Select the user's inbox, check the `UIDVALIDITY` of the mailbox is 50 and
fetch message 2 if it is:

    imap://user:password@mail.example.com/INBOX;UIDVALIDITY=50/;UID=2

Select the user's inbox and fetch the text portion of message 3:

    imap://user:password@mail.example.com/INBOX/;UID=3/;SECTION=TEXT

Select the user's inbox and fetch the first 1024 octets of message 4:

    imap://user:password@mail.example.com/INBOX/;UID=4/;PARTIAL=0.1024

Select the user's inbox and check for NEW messages:

    imap://user:password@mail.example.com/INBOX?NEW

Select the user's inbox and search for messages containing "shadows" in the
subject line:

    imap://user:password@mail.example.com/INBOX?SUBJECT%20shadows

For more information about the individual components of an IMAP URL please see
RFC 5092.

## LDAP

The path part of a LDAP request can be used to specify the: Distinguished
Name, Attributes, Scope, Filter and Extension for a LDAP search. Each field is
separated by a question mark and when that field is not required an empty
string with the question mark separator should be included.

Search for the DN as `My Organisation`:

    ldap://ldap.example.com/o=My%20Organisation

the same search but will only return postalAddress attributes:

    ldap://ldap.example.com/o=My%20Organisation?postalAddress

Seearch for an empty DN and request information about the
`rootDomainNamingContext` attribute for an Active Directory server:

    ldap://ldap.example.com/?rootDomainNamingContext

For more information about the individual components of a LDAP URL please
see [RFC 4516](https://tools.ietf.org/html/rfc4516).

## POP3

The path part of a POP3 request specifies the message ID to retrieve. If the
ID is not specified then a list of waiting messages is returned instead.

## SCP

The path part of an SCP URL specifies the path and file to retrieve or
upload. The file is taken as an absolute path from the root directory on the
server.

To specify a path relative to the user's home directory on the server, prepend
`~/` to the path portion.

## SFTP

The path part of an SFTP URL specifies the file to retrieve or upload. If the
path ends with a slash (`/`) then a directory listing is returned instead of a
file. If the path is omitted entirely then the directory listing for the root
/ home directory will be returned.

## SMB
The path part of a SMB request specifies the file to retrieve and from what
share and directory or the share to upload to and as such, may not be omitted.
If the user name is embedded in the URL then it must contain the domain name
and as such, the backslash must be URL encoded as %2f.

curl supports SMB version 1 (only)

## SMTP

The path part of a SMTP request specifies the host name to present during
communication with the mail server. If the path is omitted, then libcurl will
attempt to resolve the local computer's host name. However, this may not
return the fully qualified domain name that is required by some mail servers
and specifying this path allows you to set an alternative name, such as your
machine's fully qualified domain name, which you might have obtained from an
external function such as gethostname or getaddrinfo.

## RTMP

There's no official URL spec for RTMP so libcurl uses the URL syntax supported
by the underlying librtmp library. It has a syntax where it wants a
traditional URL, followed by a space and a series of space-separated
`name=value` pairs.

While space is not typically a "legal" letter, libcurl accepts them. When a
user wants to pass in a `#` (hash) character it will be treated as a fragment
and get cut off by libcurl if provided literally. You will instead have to
escape it by providing it as backslash and its ASCII value in hexadecimal:
`\23`.