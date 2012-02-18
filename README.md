Signed Scripts In Firefox
=========================

 * Given: you have a web application
 * Task: allow users to do something from browser, that cannot be implemented in pure JS. For example sign message with digital signature with certificate from e-Token hardware device.
 * Solution: write Firefox addon to do stuff and call it from the JS script.

But things are not so simple with Firefox Addons.

Firefox security privileges
---------------------------

Firefox has security privileges system for various tasks, e.g. "UniversalFileRead" for reading files, "UniversalXPConnect" to use browser API, etc. This priviliges must be requested before use:

    netscape.security.PrivilegeManager.enablePrivilege('UniversalXPConnect')

User will be asked ones about "page want privilege" and decision would be remembered by Firefox. 

Signed Scripts
--------------

All looks fine, when you testing you addons access code in local HTML file. Then you deploy working example to server and suddenly Firefox starts to complain: "page was denied <privilege_name>".
It's explained by security restrictions, to enable privileges you should use signed scripts - http://www.mozilla.org/projects/security/components/signed-scripts.html
After reading this link thoroughly first reaction may be "What? Full page signed and served by 'jar:...' URL?! They must be joking."
No, this is not a joke, this security stuff was written over a decade ago for Netscape and underwent little changes. But webapp users like Firefox, so you should deal with it.

Simple Workaround
-----------------

Signing full pages is near impossible with today's web frameworks. So workaround is to sign one special page with all privileged code, and load it into invisible IFRAME. Then use it from JS as you need.
You can see example in privilegedframe/privileged.html. Parent (unsigned) page's example is in parent.html. Nothing really new here, but openssl and NSS-utils examples might be helpful.

Signing page
------------

_Note: some PKI (public key infrastructure) and crypto knowledge is required to read futher._

 For all this to work privilegedframe directory must by signed by code signing certificate, zipped and renamed to .jar. It sound more simple that it is.
 You need to get code signing certificate. For intranet applications you may (as usual) use your own (your company's, openssl generated, etc) CA.
 NSS (crypto library behind Firefox) has heavy restrictions for code signing certificates - http://www.mozilla.org/projects/security/pki/nss/tech-notes/tn3.html 
 To generate one NSS compatible you may use codesign.cnf from this project. If you have openssl-generated CA then you can generate code signing cert like this:

    # generate request
    openssl req -new -keyout newkey.pem -out newreq.pem -config codesign.cnf
    # sign request with CA
    openssl ca -policy policy_anything -config codesign.cnf -out newcert.pem -infiles newreq.pem
    # convert results int #PKCS12 format
    openssl pkcs12 -export -in newcert.pem -inkey newkey.pem -out codesign.p12 -name code_signing -caname your_ca_alias -chain -CAfile ./demoCA/cacert.pem 

Now you have codesign.p12 certificate. To sign privilegedframe directory you should use NSS utils: pk12util, certutil and signtool.

    mkdir certdb
    # create certificate db
    certutil -N -d certdb
    # load PKCS12 into db
    pk12util -i codsign.p12 -d certdb
    # list certs in db
    certutil -L -d certdb
    # list one cert info
    certutil -L -d certdb -n "alias"
    # change CA flags - you should do it for CA certs
    certutil -d certdb -M -t "TC,C,C" -n "ca_calias"
    # list certificates, that can be used by signtool
    signtool -l -d certdb
    # sign privilegedframe directory (it may contain HTML, JS etc)
    signtool -d certdb -k "alias" -p password -Z privilegedframe.jar privilegedframe

The work is done, some additional notes:

 * privilegedframe.jar, must be served with Content-Type=application/java-archive or application/x-jar.
 * user must have CA certificate installed in Firefox with trust option "This certificate can identificate software makers" enabled.
 * if you reject one of Firefox' security questions, the ony way to reenable it may be to delete Firerfox profile completely.

License information
-------------------
You can use any code from this project under terms of Apache Licence (http://www.apache.org/licenses/LICENSE-2.0)

