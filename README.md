# vCloud-Director

<h2>Does vCloud Director have VMCA as default certificate authority / what is the default certificate authority for vCD ?</h2>

vCloud Director do not have VMCA as the default certificate authority. By default, vCD uses a self-signed custom certificate. VMCA provisions vCenter Server components and ESXi hosts with certificates that use VMCA as the root certificate authority. That too, vSphere 6.0 version onwards. So older vSphere versions also have self-sign default certs. If you are upgrading to vSphere 6 from an earlier version of vSphere, all self-signed certificates are replaced with certificates that are signed by VMCA.

[[
“VMCA can handle all certificate management. VMCA provisions vCenter Server components and ESXi hosts with certificates that use VMCA as the root certificate authority. If you are upgrading to vSphere 6 from an earlier version of vSphere, all self-signed certificates are replaced with certificates that are signed by VMCA. 
If you do not currently replace VMware certificates, your environment starts using VMCA-signed certificates instead of self-signed certificates.
IMP LINKS :
Certificate Management Overview :
https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.psc.doc/GUID-3D0DE463-D0EC-442E-B524-64759D063E25.html
vCloud Director security guide 9.5 :
https://docs.vmware.com/en/vCloud-Director/9.5/vcd_sec.pdf
]]

vCD has 2 IP address which allows support for 2 different SSL endpoints (http and consoleproxy). Each endpoint requires its own SSL certificate. vCloud Director uses a java keystore to read its SSL certificates from.  In a Multi-cell environment you need to create 2 certificates for each cell and import the certificates into vcd java keystore.
</pre>

                                                                           07/08/2019
<h2>Configure certs on vCD using self-signed cert:</h2>

IMP LINKS:

Keytool Location :
/opt/vmware/vcloud-director/jre/bin/keytool

Step 1:
keytool 
   -keystore myCert.ks
   -alias http 
   -storepass passwd
   -keypass passwd
   -storetype JCEKS
   -genkeypair
   -keyalg RSA
   -keysize 2048
   -validity 365 
   -dname "CN= v12n-host-149.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" 
   -ext "san=dns: v12n-host-149.pne.ven.veritas.com,dns: v12n-host-149,ip:10.210.128.149"

For Copy:
keytool -keystore myCert.ks -alias http -storepass passwd -keypass passwd -storetype JCEKS -genkeypair -keyalg RSA -keysize 2048 -validity 365 -dname "CN= v12n-host-149.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" -ext "san=dns:v12n-host-149.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.149"


Step 2:
keytool 
   -keystore myCert.ks
   -alias consoleproxy 
   -storepass passwd
   -keypass passwd
   -storetype JCEKS
   -genkeypair
   -keyalg RSA
   -keysize 2048
   -validity 365 
   -dname "CN= v12n-host-150.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" 
   -ext "san=dns: v12n-host-150.pne.ven.veritas.com,dns: v12n-host-149,ip:10.210.128.150"

For Copy:

keytool -keystore myCert.ks -alias consoleproxy -storepass passwd -keypass passwd -storetype JCEKS -genkeypair -keyalg RSA -keysize 2048 -validity 365 -dname "CN= v12n-host-150.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" -ext "san=dns:v12n-host-150.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.150"


Step 3:

To verify that all the certificates are imported, list the contents of the keystore file.

keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -list

Output :
Keystore type: JCEKS
Keystore provider: SunJCE

Your keystore contains 2 entries

consoleproxy, 7 Aug, 2019, PrivateKeyEntry,
Certificate fingerprint (SHA1): 6E:1E:76:35:B9:F6:34:7E:0B:57:DE:86:F6:A0:27:21:3A:DF:03:37
http, 7 Aug, 2019, PrivateKeyEntry,
Certificate fingerprint (SHA1): 2B:90:8B:E4:7E:C4:73:60:96:D4:AB:B9:B3:64:DD:67:13:AE:77:2A

Next step is to run configure script :
Configure script location :
/opt/vmware/vcloud-director/bin/configure

Step 1:
Stop vCloud Director service using following command:
service vmware-vcd stop

Step 2:
Edit responses.properties file

Step 3:
Run config script :
/opt/vmware/vcloud-director/bin/configure -r /opt/vmware/vcloud-director/responses.properties

Start vCloud Director service using following command:
service vmware-vcd start


                                                                                              08/08/2019
<h2>Create your own CA for signing certs:</h2>

•	Step 1: Create private key :
openssl genrsa -des3 -out myCA.key 2048

•	Step 2: Generate root certificate :
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 1825 -out myCA.pem

Congratulations, you’re now a CA. Sort of.
To become a real CA, you need to get your root certificate on all the devices in the world. Let’s start with the ones you own.

For signing any CSR with your root cert you will require :

1.	Your CA certificate
2.	CA private key
3.	Config file (The config file is needed to define the Subject Alternative Name (SAN) extension)
4.	& obvious CSR file.

Create config file as: v12n-host-149.pne.ven.veritas.com.ext :

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = v12n-host-149.pne.ven.veritas.com


Now we run the command to create the certificate:

openssl x509 -req -in v12n-host-149.pne.ven.veritas.com.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial \ -out v12n-host-149.pne.ven.veritas.com.crt -days 1825 -sha256 -extfile v12n-host-149.pne.ven.veritas.com.ext



<h2>Configure vCD using CA signed cert :</h2>


High level steps for replacing vCD certificates can be summarized as below:
•	Create untrusted certificates with JAVA keytool command.
•	Send certificates to your Certificate Authority and obtain signed certificates.
•	Import the Certificate Authority root certificate.
•	Import httpd and consoleproxy signed certificates.
•	Stop vCD Cell service
•	Invoke vCD configuration script

Location of keytool command is : /opt/vmware/vcloud-director/jre/bin

Step 1: Create an untrusted certificate for the HTTP service.

keytool -keystore myCert.ks -alias http -storepass passwd -keypass passwd -storetype JCEKS -genkeypair -keyalg RSA -keysize 2048 -validity 365 -dname "CN= v12n-host-149.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" -ext "san=dns:v12n-host-149.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.149"


Step 2: Create an untrusted certificate for the console proxy service.

keytool -keystore myCert.ks -alias consoleproxy -storepass passwd -keypass passwd -storetype JCEKS -genkeypair -keyalg RSA -keysize 2048 -validity 365 -dname "CN= v12n-host-150.pne.ven.veritas.com, OU=Engineering, O=Veritas, L=Pune, S=Maharashtra, C=IN" -ext "san=dns:v12n-host-150.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.150"

Step 3 : Create a certificate signing request for the HTTP service and for the console proxy service.

a.	Create a certificate signing request in the http.csr file.
keytool -keystore myCert.ks -storetype JCEKS -storepass passwd -certreq -alias http -file http.csr -ext "san=dns:v12n-host-149.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.149"

b.	Create a certificate signing request in the consoleproxy.csr file.

keytool -keystore myCert.ks -storetype JCEKS -storepass passwd -certreq -alias consoleproxy -file consoleproxy.csr -ext "san=dns:v12n-host-150.pne.ven.veritas.com,dns:v12n-host-149,ip:10.210.128.150"


Step 4: Send the certificate signing requests to your Certificate Authority.

Step 5: If you obtained the signed certificates in PEM format, you must import them to a PKCS12 intermediate keystore. 
If the certificates are not in PEM format, skip to step 6. 

Step 6: Import the signed certificates into the JCEKS keystore file.

a.	Import the Certificate Authority's root certificate from the root.cer file to the certificates.ks keystore file. 
keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -import -alias root -file root.cer
b.	If you received intermediate certificates, import them from the intermediate.cer file to the certificates.ks keystore file. 
keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -import -alias intermediate -file intermediate.cer
c.	Import the HTTP service certificate.
keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -import -alias http -file http.cer
[[ keytool -importkeystore -deststoretype JCEKS -deststorepass keystore password -destkeystore myCert.ks -srckeystore http.p12 -srcstoretype PKCS12 -srcstorepass keystore password ]]
d.	Import the console proxy service certificate. 
keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -import -alias consoleproxy -file consoleproxy.cer
[[  keytool -importkeystore -deststoretype JCEKS -deststorepass keystore password -destkeystore myCert.ks -srckeystore consoleproxy.p12 -srcstoretype PKCS12 -srcstorepass keystore password
]]

Step 7: To check if the certificates are imported, run the command to list the contents of the keystore file.
keytool -storetype JCEKS -storepass passwd -keystore myCert.ks -list

Step 8: Run configure command.
/opt/vmware/vcloud-director/bin/configure

