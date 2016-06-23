# Java SSL (keytool)
## Import a CA cert into a java keystore
*keytool -import -file ca-cert.pem -keystore /path/to/keystore -trustcacerts*
(-trustcacets is important when adding a CA cert as it will add the CA cert to the keystore trust chain)
The root CA cert should be all that's necessary for Java cert verification; it doesn't seem to require the intermediate CA certs like OpenSSL does.


## List the certs in a keystore
keytool -list -v -keystore /path/to/keystore


## Working with JCE keystores
If you get this error:
keytool error: java.io.IOException: Invalid keystore format

## Check the filetype of the keystore:
$ file /path/to/keystore
keystore: Java JCE KeyStore

## If it's a JCE keystore, add -storetype JCEKS:
keytool -list -v -keystore /path/to/keystore -storetype JCEKS


## Testing Java SSL connectivity
java SSLPoke host port

## To point it to a custom keystore:
java -Djavax.net.ssl.trustStore=/my/custom/keystore SSLPoke host port

## Errors:
java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
If you're using -Djavax.net.ssl.trustStore, make sure the keystore you're pointing to exists

## java.security.KeyException
First try to upgrade nss (http://blog.backslasher.net/java-ssl-crash.html):
sudo rpm -y upgrade nss

## Generate a self-signed keystore
HOSTNAME=`hostname -f` keytool -genkey -v -keystore $HOSTNAME.jks -alias demoidentity -keyalg RSA -keysize 2048 -sigalg SHA256withRSA -validity 5480 -dname "CN=$HOSTNAME, OU=FOR TESTING ONLY, O=MyOrganization, L=MyTown, ST=MyState, C=US" -storepass changeit -keypass changeit


## Create a new keystore from an existing cert/key pair
Convert the cert/key pair into a PKCS12 keystore
HOSTNAME=`hostname -f` openssl pkcs12 -export -in $HOSTNAME.pem -inkey $HOSTNAME.key -out $HOSTNAME.p12 -name $HOSTNAME -passout pass:changeit

## Convert the PKCS12 keystore to a Java keystore (see below for more information)
HOSTNAME=`hostname -f` keytool -importkeystore -deststorepass changeit -destkeypass changeit -destkeystore $HOSTNAME.jks -srckeystore $HOSTNAME.p12 -srcstoretype PKCS12 -srcstorepass changeit -alias $HOSTNAME


## Export a cert from one keystore to another as a trusted CA cert
Export the cert from the first keystore (it can be a cert/key pair or a trusted CA cert) 
keytool -exportcert -rfc -keystore keystore1.jks -storepass changeit -alias aliastoexport -file aliastoexport.pem

## Import the cert into the second keystore
keytool -importcert -file aliastoexport.pem -keystore keystore2.jks -storepass changeit -alias aliastoexport -trustcacerts -noprompt


## Verify private key password in a keystore
Copy the keystore so you don't accidentally change it
cp keystore keystore-temp

## List the entries in the keystore, and get the alias of the private key whose password you want to verify
keytool -list -v -keystore keystore-temp -storepass PASSWORD

## Look for a PrivateKeyEntry, like this:
Alias name: importkey
Creation date: 9-Dec-2015
Entry type: PrivateKeyEntry

## Verify the private key password by attempting to change it
keytool -keypasswd -keystore keystore-temp -alias ALIAS -storepass PASSWORD
## This is the output you'll see if a key password isn't set, or if the key password is the same as the store password:
New key password for <ALIAS>:

## This is the output you'll see if a key password is set and you enter it incorrectly:
Enter key password for <ALIAS>
keytool error: java.security.UnrecoverableKeyException: Cannot recover key

## This is the output you'll see if a key password is set and you enter it correctly:
Enter key password for <ALIAS>
New key password for <ALIAS>: 

## Convert a PKCS12 keystore to a Java keystore
There are two methods:
The long way (preferred)
Get the alias of the private key entry to export
keytool -list -v -keystore keystore.p12 -storetype PKCS12 -storepass PASSWORD

Look for a PrivateKeyEntry, like this:
Alias name: importkey
Creation date: 9-Dec-2015
Entry type: PrivateKeyEntry

keytool -importkeystore -srckeystore keystore.p12 -srcstoretype PKCS12 -srcstorepass PASSWORD -srcalias ALIAS -deststorepass PASSWORD -destkeypass PASSWORD -destkeystore keystore.jks

Note: make sure you set deststorepass and destkeypass to the same value. If they're different, it will cause issues for certain applications, like Tomcat (https://bz.apache.org/bugzilla/show_bug.cgi?id=38217)
The short way
Warning: if you use this method, it will use the same private key password in the new keystore as the old keystore. This is likely to cause a mismatch in the new keystore between the private key password and the keystore password, which will cause the same issues mentioned above.
keytool -importkeystore -srckeystore keystore.p12 -srcstoretype PKCS12 -srcstorepass PASSWORD -deststorepass PASSWORD -destkeystore keystore.jks
