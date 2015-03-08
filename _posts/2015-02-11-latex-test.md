---
layout: post
title: How PDF Encryption works
---
[Source](https://www.cs.cmu.edu/~dst/Adobe/Gallery/anon21jul01-pdf-encryption.txt)

The following explains how PDF encryption, using Adobe's "Standard Security Handler", works.
This is what you get if you select document security options in PDF 1.3 (Acrobat 4.x) or earlier.
All of this information is in the publicly available [PDF spec](http://partners.adobe.com/asn/developer/technotes/acrobatpdf.html).

The goal of protecting a PDF file is typically something like this:

	You want a viewer application to be able to display the file but not be able to print it.  
	
PDF has additional options to disallow copy-and-pasting text and editing the file; the same argument applies to them.

**The problem** here is that exactly the same information is used for both functions.
	
	Once you have a (decrypted) page description, 
		- it can be turned into either pixels on the screen or
		- toner dots on a printed page.

Adobe's PDF protection scheme is a classic example of **security through obscurity**.
They encrypt the content of a PDF file and hope that no one figures out how to decrypt it.
When Adobe's viewer encounters an encrypted PDF file, it checks a set of flags, and allows certain operations (typically viewing) while disabling others (typically printing).

Now PDF is supposedly an open standard, and in fact, Adobe has been pretty good about documenting it.
They initially refused to release detailed information on the encryption; presumably they were aware of the security/obscurity issues.
But they eventually relented.  Various third-party PDF viewers have been able to display encrypted PDF files for a few years now.

## Preliminaries
 
Before explaining how PDF file encryption works, let me give some
background on PDF files.  A PDF file consists of a series of objects,
each identified by two numbers (object number and generation number).
There is also a cross reference table which maps object numbers to
their positions in the file.  There are several object types.  The
important ones here are:

    dictionary: a table that maps names to objects (like a Perl hash)

    stream: an arbitrary chunk of data; streams are used for font
            files, page descriptions (much like a simplified
            PostScript prorgram), image data, etc.

Every document has a "*trailer dictionary*" which holds references to a few important things 
(like the tree of page objects which contains the document content) and optionally to an encryption dictionary.
If the encryption dictionary is present (i.e., if the document is encrypted), it contains the information needed to decrypt the document.
An example:

    % Trailer dictionary
    trailer
    <<
        /Size 95         % number of objects in the file
        /Root 93 0 R     % the page tree is object ID (93,0)
        /Encrypt 94 0 R  % the encryption dict is object ID (94,0)
        /ID [<1cf5...>]  % an arbitrary file identifier
    >>

    % Encryption dictionary
    94 0 obj
    <<
        /Filter /Standard   % use the standard security handler
        /V 1                % algorithm 1
        /R 2                % revision 2
        /U (xxx...xxx)      % hashed user password (32 bytes)
        /O (xxx...xxx)      % hashed owner password (32 bytes)
        /P 65472            % flags specifying the allowed operations
    >>
    endobj

There are **two passwords**: "user" and "owner".
Typically, the user password is not set (i.e., set to the empty string), thus allowing anyone to view the file.
If a PDF file is loaded into Adobe Acrobat, and the user supplies the owner password, all operations are allowed
(including re-encrypting the PDF file with different passwords, etc.).
Remember that neither password is actually used in the encryption --
the viewer application is simply expected to check that the user
supplied the password before allowing him/her to print (etc.).

In the above example, the permission flags are 65472 (decimal) or
1111111111000000 (binary).  Bits 0 and 1 are reserved (always 0), bit
2 is the print permission (0 here, meaning that printing is not
allowed), and bits 3, 4, and 5, are the "modify", "copy text", and
"add/edit annotations" permissions (all disallowed in this example).
The higher bits are reserved.

The encryption key is generated as follows:

    1. Pad the user password out to 32 bytes, using a hardcoded
       32-byte string:
           28 BF 4E 5E 4E 75 8A 41 64 00 4E 56 FF FA 01 08
           2E 2E 00 B6 D0 68 3E 80 2F 0C A9 FE 64 53 69 7A
       If the user password is null, just use the entire padding
       string.  (I.e., concatenate the user password and the padding
       string and take the first 32 bytes.)

    2. Append the hashed owner password (the /O entry above).

    3. Append the permissions (the /P entry), treated as a four-byte
       integer, LSB first.

    4. Append the file identifier (the /ID entry from the trailer
       dictionary).  This is an arbitrary string of bytes; Adobe
       recommends that it be generated by MD5 hashing various pieces
       of information about the document.

    5. MD5 hash this string; the first 5 bytes of output are the
       encryption key.  (This is a 40-bit key, presumably to meet US
       export regulations.)

Note that the inputs to this algorithm are: the user password
(typically empty) and various information specified in the PDF file.

The hashed user password (/U entry) is simply the 32-byte padding
string above, encrypted with RC4, using the 5-byte file key.
Compliant PDF viewers will check the password given by the user (by
attempting to decrypt the /U entry using the file key, and comparing
it against the padding string) and allow or refuse certain operations
based on the permission settings.

The hashed owner password is also generated using MD5 and RC4.
Details are given in the PDF 1.3 manual, but it's completely
irrelevant if you have the user password (or the user password is
null).

All stream (and string) objects in the PDF file are encrypted.  This
is sufficient to render the file useless (that is, if it weren't so
easy to decrypt).  Stream/string decryption works like this:

    1. Take the 5-byte file key (from above).

    2. Append the 3 low-order bytes (LSB first) of the object number
       for the stream/string object being decrypted.

    3. Append the 2 low-order bytes (LSB first) of the generation
       number.

    4. MD5 hash that 10-byte string.

    5. Use the first 10 bytes of the output as an RC4 key to decrypt
       the stream or string.  (This apparently still meets the US
       export regulations because it's a 40-bit key with an additional
       40-bit "salt".)

To decrypt a PDF file (i.e., generate a new PDF file, identical except
that all encryption is removed), just filter the file, applying the
above algorithm to decrypt every stream and string object.  Then
remove the /Encrypt entry in the trailer dictionary.

If you have the user password or the user password is null (or you
have the owner password), you can decrypt the PDF file.  The only time
when there is some protection is when both the user and owner
passwords are set; in this case you'd be stuck doing a brute force
attack on 40-bit RC4.

PDF 1.4 (Acrobat 5.x) includes a revised security handler.  I have
only briefly skimmed the PDF 1.4 spec, but it appears to add three
things:

    * algorithm 2, which is identical to algorithm 1 (above) except
      that it allows longer keys

    * algorithm 3, which is unpublished (which the PDF 1.4 spec claims
      is "an export requirement of the U.S. Department of Commerce".)

    * revision 3 (which applies to algorithms 1-3), which adds some
      steps (runing MD5 and RC4 multiple times), presumably to slow
      down brute force attacks

There are also third-party encryption plugins, which are, of course,
not publicly documented.  The ElcomSoft presentation points out that
some of these are even easier to break than Adobe's encryption.

