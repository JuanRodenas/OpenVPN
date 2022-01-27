# OpenVPN
Proyecto para crear una vpn en Wireguard en Docker.

<div class="content-body tutorial-content" data-growable-markdown>
  
  <h3 id="introducción">Introducción</h3>

<p>Una <a href="https://en.wikipedia.org/wiki/Virtual_private_network">red privada virtual</a> (VPN), le permite atravesar redes no fiables como si estuviese en una red privada. Le permite acceder a internet de forma segura desde su smartphone o equipo portátil cuando se conecta a una red no fiable, como la WiFi de un hotel o de una cafetería.</p>

<p>Cuando se combina con <a href="https://en.wikipedia.org/wiki/HTTP_Secure">conexiones HTTPS</a>, esta configuración le permite proteger los inicios de sesión y las operaciones que realiza por medios inalámbricos. Puede evadir censuras y restricciones geográficas, y proteger su ubicación y el tráfico de HTTP no cifrado contra la actividad de las redes no fiables.</p>

<p><a href="https://openvpn.net">OpenVPN</a> es una solución VPN de seguridad en la capa de transporte (TLS) de código abierto y con características completas que aloja muchas configuraciones. En este tutorial, instalará OpenVPN en un servidor Ubuntu 20.04 y, luego, la configurará para que sea accesible desde la máquina de un cliente.</p>

<span class='note'><p>

<p>Para este tutorial, necesitará lo siguiente:</p>

<ul>
<li>Un servidor de Ubuntu 20.04 con un sudo non-root user y un firewall habilitado. Para configurarlo, puede seguir nuestro tutorial <a href="https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04">Configuración inicial para servidores con Ubuntu 20.04</a>. En todo el documento, nos referiremos a él como <strong>Servidor OpenVPN</strong>.</li>
<div class="content-body tutorial-content" data-growable-markdown>
<li>Un servidor Ubuntu 20.04 independiente configurado como Entidad de certificación (CA) privada, a la que nos referiremos como <strong>Servidor CA</strong> en esta guía. Después de ejecutar los pasos de la <a href="https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04">Guía de configuración inicial para servidores</a> en este servidor, puede seguir los pasos 1 a 3 de nuestra guía <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04">Cómo instalar y configurar una entidad de certificación (CA) en Ubuntu 20.04</a> para ello.</li>

<h2 id="Una entidad de certificación (CA)">Una entidad de certificación (CA)</h2>Las <a href="https://en.wikipedia.org/wiki/Certificate_authority">entidades de certificación (CA)</a> son responsables de emitir certificados digitales para verificar identidades en Internet. Las CA públicas son una opción popular para verificar la identidad de sitios web y otros servicios que se proporcionan al público en general, y las CA privadas suelen usarse para grupos cerrados y servicios privados.</p>

<p>Compilar una entidad de certificación privada le permitirá configurar, probar y ejecutar programas que requieren conexiones cifradas entre un cliente y un servidor. Con una CA privada, puede emitir certificados para usuarios, servidores o programas y servicios individuales dentro de su infraestructura.</p>

<p>Algunos ejemplos de programas de Linux que utilizan su propia CA privada son <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-18-04">OpenVPN</a> y <a href="https://www.digitalocean.com/community/tutorials/how-to-manage-puppet-4-certificates">Puppet</a>. También puede configurar su servidor web para que use certificados emitidos por una CA privada a fin de que los entornos de desarrollo y ensayo se adapten a los servidores de producción que utilizan TLS para cifrar conexiones.</p>

<p>En esta guía, aprenderá a instalar una entidad de certificación privada en un servidor de Ubuntu 20.04 y a generar y firmar un certificado de prueba con su CA nueva. También aprenderá a importar el certificado público del servidor de CA al almacén de certificados de su sistema operativo para poder verificar la cadena de confianza entre los usuarios o servidores remotos y la CA. Por último, aprenderá a revocar certificados y a distribuir una lista de revocación de certificados para asegurarse de que solo usuarios y sistemas autorizados puedan usar los servicios que se basan en su CA.</p>

<h2 id="requisitos-previos">Requisitos previos</h2>

<p>Para completar este tutorial, necesitará acceso a un servidor Ubuntu 20.04 a fin de alojar su servidor de CA. Antes de comenzar a seguir los pasos de esta guía, deberá configurar un non-<strong>root</strong> user con privilegios <code>sudo</code>. Puede seguir nuestra <a href="https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04">guía de configuración inicial de servidores para Ubuntu 20.04</a> para configurar un usuario con los permisos adecuados. A través del tutorial del enlace también se podrá configurar un <strong>firewall</strong>, que para esta guía se supone que está instalado.</p>

<p>En este tutorial, emplearemos el término <strong>servidor de CA</strong> para hacer referencia a este servidor.</p>

<p>Asegúrese de que el sistema del servidor de CA sea independiente. Se usará únicamente para importar, firmar y revocar solicitudes de certificados. No debe ejecutar ningún otro servicio y lo ideal es que se desconecte o desactive por completo cuando usted no esté trabajando activamente con su CA.</p>

<p><span class='note'><strong>Nota:</strong> La última sección de este tutorial es opcional si desea aprender a firmar y revocar certificados. Si decide completar esos pasos de prueba, necesitará un segundo servidor de Ubuntu 20.04 o, de forma alternativa, puede usar su propia computadora local de Linux con Debian o Ubuntu, o distribuciones derivadas de cualquiera de ellos.<br></span></p>

<h2 id="paso-1-instalar-easy-rsa">Paso 1: Instalar Easy-RSA</h2>

<p>La primera tarea de este tutorial es instalar el conjunto de secuencias de comandos <code>easy-rsa</code> en su servidor de CA. <code>easy-rsa</code> es una herramienta de gestión de entidades de certificación que utilizará para generar una clave privada y un certificado root público que, luego, usará para firmar las solicitudes de los clientes y servidores que se basarán en su CA.</p>

<p>Inicie sesión en su servidor de CA con el non-root sudo user que creó en los pasos de configuración iniciales y ejecute lo siguiente:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo apt update
</li><li class="line" data-prefix="$">sudo apt install easy-rsa
</li></ul></code></pre>
<p>Se le solicitará descargar e instalar el paquete. Presione <code>y</code> para confirmar que desea instalarlo.</p>

<p>En este punto, tiene todo lo que necesita para usar Easy-RSA. En el siguiente paso, creará una infraestructura de clave pública y, luego, empezará a crear su entidad de certificación.</p>

<h2 id="paso-2-preparar-un-directorio-para-la-infraestructura-de-clave-pública">Paso 2: Preparar un directorio para la infraestructura de clave pública</h2>

<p>Ahora que instaló <code>easy-rsa</code>, es el momento de crear una <a href="https://en.wikipedia.org/wiki/Public_key_infrastructure">infraestructura de clave pública</a> (PKI) de esqueleto en el servidor de CA. Verifique que siga conectado con su non-root user y cree un directorio <code>easy-rsa</code>. Asegúrese de <strong>no utilizar sudo</strong> para ejecutar ninguno de los siguientes comandos, dado que su usuario normal debe administrar la CA e interactuar con ella sin privilegios elevados.</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">mkdir /home/jrodenas/easy-rsa
</li></ul></code></pre>
<p>Con esto se creará un directorio nuevo llamado <code>easy-rsa</code> en su carpeta de inicio. Usaremos este directorio para crear enlaces simbólicos que apunten a los archivos del paquete <code>easy-rsa</code> que instalamos en el paso anterior. Estos archivos se encuentran en la carpeta <code>/usr/share/easy-rsa</code> en el servidor de CA.</p>

<p>Cree los enlaces simbólicos con el comando <code>ln</code>:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">ln -s /usr/share/easy-rsa/* /home/jrodenas/easy-rsa/
</li></ul></code></pre>
<p><span class='note'><strong>Nota:</strong> Aunque en otras guías se le indique copiar los archivos del paquete <code>easy-rsa</code> a su directorio de la PKI, en este tutorial, usaremos enlaces simbólicos. Como resultado, toda actualización del paquete <code>easy-rsa</code> se reflejará automáticamente en las secuencias de comandos de su PKI.<br></span></p>

<p>Para restringir el acceso a su nuevo directorio de la PKI, asegúrese de que solo el propietario pueda acceder a él usando el comando <code>chmod</code>:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">chmod <span class="highlight">700</span> /home/jrodenas/easy-rsa
</li></ul></code></pre>
<p>Por último, inicie la PKI dentro del directorio <code>easy-rsa</code>:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">./easyrsa init-pki
</li></ul></code></pre><pre class="code-pre "><code><div class="secondary-code-label " title="Output">Output</div>init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /home/jrodenas/easy-rsa/pki
</code></pre>
<p>Después de completar esta sección, tendrá un directorio con todos los archivos necesarios para crear una entidad de certificación. En la siguiente sección, creará la clave privada y el certificado público para su CA.</p>

<h2 id="paso-3-crear-una-entidad-de-certificación">Paso 3: Crear una entidad de certificación</h2>

<p>Para poder crear la clave privada y el certificado de su CA, debe crear y completar un archivo llamado <code>vars</code> con algunos valores predeterminados. Primero, usará <code>cd</code> para ingresar al directorio <code>easy-rsa</code> y, luego, creará y editará el archivo <code>vars</code> con <code>nano</code> o el editor de texto que prefiera:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">nano vars
</li></ul></code></pre>
<p>Una vez que se abra el archivo, pegue las siguientes líneas y sustituya cada valor resaltado por la información de su propia organización. Lo importante aquí es asegurarse de no dejar ninguno de los valores en blanco:</p>
<pre class="code-pre "><code><div class="secondary-code-label " title="/home/jrodenas/easy-rsa/vars">/home/jrodenas/easy-rsa/vars</div>set_var EASYRSA_REQ_COUNTRY    "<span class="highlight">US</span>"
set_var EASYRSA_REQ_PROVINCE   "<span class="highlight">NewYork</span>"
set_var EASYRSA_REQ_CITY       "<span class="highlight">New York City</span>"
set_var EASYRSA_REQ_ORG        "<span class="highlight">Organization</span>"
set_var EASYRSA_REQ_EMAIL      "<span class="highlight">admin@example.com</span>"
set_var EASYRSA_REQ_OU         "<span class="highlight">Community</span>"
set_var EASYRSA_ALGO           "ec"
set_var EASYRSA_DIGEST         "sha512"
</code></pre>
<p>Cuando termine, guarde y cierre el archivo. Si utiliza <code>nano</code>, puede hacerlo pulsando <code>CTRL+X</code>, <code>Y</code> y <code>ENTER</code> para confirmar. Con esto, estará listo para crear su CA.</p>

<p>Para crear el certificado root público y el par de claves privadas para su entidad de certificación, vuelva a ejecutar el comando <code>./easy-rsa</code>, aunque esta vez con la opción <code>build-ca</code>:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa build-ca
</li></ul></code></pre>
<p>En el resultado, verá algunas líneas sobre la versión de OpenSSL y se le solicitará ingresar una frase de contraseña para su par de claves. Asegúrese de elegir una frase de contraseña segura y anótela en un lugar resguardado. Deberá ingresar la frase de contraseña siempre que deba interactuar con su CA, por ejemplo, para firmar o revocar un certificado.</p>

<p>También se le solicitará confirmar el nombre común (CN) de su CA. El nombre común es el que se usa para hacer referencia a esta máquina en el contexto de la entidad de certificación. Puede ingresar cualquier secuencia de caracteres para el nombre común de la CA; sin embargo, para hacerlo más simple presione ENTER para aceptar el nombre predeterminado.</p>
<pre class="code-pre "><code><div class="secondary-code-label " title="Output">Output</div>. . .
Enter New CA Key Passphrase:
Re-Enter New CA Key Passphrase:
. . .
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/home/jrodenas/easy-rsa/pki/ca.crt
</code></pre>
<span class='note'><p>
<strong>Nota:</strong> Si no desea que se le solicite una contraseña cada vez que interactúe con su CA, puede ejecutar el comando <code>build-ca</code> con la opción <code>nopass</code>, de la siguiente forma:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa build-ca nopass
</li></ul></code></pre>
<p></p></span>

<p>Ahora, tiene dos archivos importantes, <code>/home/jrodenas/easy-rsa/pki/ca.crt</code> y <code>/home/jrodenas/easy-rsa/pki/private/ca.key</code>, que conforman los componentes públicos y privados de una entidad de certificación.</p>

<ul>
<li><p><code>ca.crt</code> es el archivo del certificado público de la CA. Los usuarios, los servidores y los clientes utilizarán este certificado para verificar que sean parte de la misma red de confianza. Todos los usuarios y los servidores que usen su CA deberán tener una copia de este archivo. Todas las partes se basarán en el certificado público para asegurarse de que nadie suplante un sistema y realice un <a href="https://en.wikipedia.org/wiki/Man-in-the-middle_attack">ataque con intermediario</a>.</p></li>
<li><p><code>ca.key</code> es la clave privada que usa la CA para firmar certificados para servidores y clientes. Si un atacante obtiene acceso a su CA y, a la vez, a su archivo <code>ca.key</code>, deberá destruir la CA. Esta es la razón por la cual su archivo <code>ca.key</code> deberá estar <strong>únicamente</strong> en su máquina de CA. A su vez, lo ideal sería que su máquina de CA estuviera desconectada cuando no firme solicitudes de certificados como medida de seguridad adicional.</p></li>
</ul>

<p>Con esto, estableció su CA y esta se encuentra lista para emplearse en la firma de solicitudes de certificados y revocar certificados.</p>

<h2 id="paso-4-distribuir-el-certificado-público-de-su-entidad-de-certificación">Paso 4: Distribuir el certificado público de su entidad de certificación</h2>

<p>Ahora, su CA está configurada y lista para funcionar como root de confianza para cualquier sistema que desee que la use. Puede agregar el certificado de la CA a sus servidores de OpenVPN, web y de correo, entre otros. Cualquier usuario o servidor que necesite verificar la identidad de otro usuario o servidor de su red debe contar con una copia del archivo <code>ca.crt</code> importada en el almacén de certificados de su sistema operativo.</p>

<p>Para importar el certificado público de la CA a un segundo sistema de Linux, como otro servidor o una computadora local, primero debe obtener una copia del archivo <code>ca.crt</code> de su servidor de CA. Puede usar el comando <code>cat</code> para ver el resultado en una terminal y, luego, copiarlo y pegarlo en un archivo en la segunda computadora en la que se importe el certificado. También puede usar herramientas como <code>scp</code> y <code>rsync</code> para transferir el archivo entre sistemas. Sin embargo, usaremos el método de copiar y pegar con <code>nano</code> en este paso, ya que funciona en todos los sistemas.</p>

<p>Como non-root user en el servidor de CA, ejecute el siguiente comando:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cat /home/jrodenas/easy-rsa/pki/ca.crt
</li></ul></code></pre>
<p>El resultado en su terminal será similar al siguiente:</p>
<pre class="code-pre "><code><div class="secondary-code-label " title="Output">Output</div>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUcR9Crsv3FBEujrPZnZnU4nSb5TMwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjAwMzE4MDMxNjI2WhcNMzAw
. . .
. . .
-----END CERTIFICATE-----
</code></pre>
<p>Copie todo, incluso las líneas <code>-----BEGIN CERTIFICATE-----</code> y <code>-----END CERTIFICATE-----</code> y los guiones.</p>

<p>En su segundo sistema de Linux, use <code>nano</code> o el editor de texto que prefiera para abrir un archivo llamado <code>/tmp/ca.crt</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">nano /tmp/ca.crt
</li></ul></code></pre>
<p>Pegue lo que acaba de copiar del servidor de CA en el editor. Cuando termine, guarde y cierre el archivo. Si utiliza <code>nano</code>, puede hacerlo pulsando <code>CTRL+X</code> y, luego, <code>Y</code> y <code>ENTER</code> para confirmar.</p>

<p>Ahora que tiene una copia del archivo <code>ca.crt</code> en su segundo sistema de Linux, es momento de importar el certificado al almacén de certificados de su sistema operativo.</p>

<p>En los sistemas basados en Ubuntu y Debian, ejecute los siguientes comandos como non-root user para importar el certificado:</p>
<div class="code-label " title="Ubuntu and Debian derived distributions">Ubuntu and Debian derived distributions</div><pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/
</li><li class="line" data-prefix="$">sudo update-ca-certificates
</li></ul></code></pre>
<p>Para importar el certificado del servidor de CA en un sistema basado en CentOS, Fedora o RedHat, copie el contenido del archivo y péguelo en el sistema, como en el ejemplo anterior, en un archivo denominado <code>/tmp/ca.crt</code>. A continuación, copie el certificado en <code>/etc/pki/ca-trust/source/anchors/</code> y luego ejecute el comando <code>update-ca-trust</code>.</p>
<div class="code-label " title="CentOS, Fedora, RedHat distributions">CentOS, Fedora, RedHat distributions</div><pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /tmp/ca.crt /etc/pki/ca-trust/source/anchors/
</li><li class="line" data-prefix="$">sudo update-ca-trust
</li></ul></code></pre>
<p>Ahora, su segundo sistema de Linux confiará en cualquier certificado firmado por el servidor de CA.</p>

<span class='note'><p>
<strong>Nota:</strong> Si utiliza su CA con servidores web y usa Firefox como navegador, deberá importar el certificado público <code>ca.crt</code> directamente a Firefox. Firefox no usa el almacén de certificados del sistema operativo local. Para obtener información sobre cómo agregar el certificado de su CA a Firefox, consulte el artículo de asistencia de Mozilla sobre <a href="https://support.mozilla.org/en-US/kb/setting-certificate-authorities-firefox">configuración de entidades de certificación (CA) en Firefox</a>.</p>

<p>Si usa su CA para la integración con un entorno de Windows o computadoras de escritorio, consulte la documentación sobre cómo usar <code>certutil.exe</code> <a href="https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/certutil#-installcert">para instalar un certificado de CA</a>.<br></p></span>

<p>Si completa este tutorial como requisito previo para otro o está familiarizado con la forma de firmar y revocar certificados, puede detenerse aquí. Si desea obtener más información sobre cómo firmar y revocar certificados, en la siguiente sección opcional, se explicará cada proceso en detalle.</p>

</ul>

<p><span class='note'><strong>Nota:</strong> Si bien es técnicamente posible usar su servidor de OpenVPN o su máquina local como CA, no se recomienda, dado que expone su VPN a algunas vulnerabilidades de seguridad. Según <a href="https://openvpn.net/index.php/open-source/documentation/">la documentación oficial de OpenVPN</a>, debería colocar su CA en una máquina independiente dedicada a importar y firmar solicitudes de certificado. Por este motivo, esta guía asume que su CA se encuentra en un servidor Ubuntu 20.04 independiente que también tiene un non-root user con privilegios sudo y un firewall básico habilitado.<br></span></p>

<p>Además de eso, necesitará una máquina de cliente que usará para conectarse a su servidor de OpenVPN. En esta guía, lo llamaremos <strong>Cliente de OpenVPN</strong>. A los efectos de este tutorial, le recomendamos que use su máquina local como el cliente de OpenVPN.</p>

<p>Una vez implementados estos requisitos previos, estará listo para comenzar a instalar y configurar un servidor de OpenVPN en Ubuntu 20.04.</p>

<p><span class='note'><strong>Nota:</strong> Tenga en cuenta que si deshabilita la autenticación de contraseña mientras configura estos servidores, es posible que experimente dificultades al transferir archivos entre ellos más adelante en esta guía. Para solucionar este problema, puede volver a habilitar la autenticación de contraseña en cada servidor. De manera alternativa, puede generar un par de claves SSH para cada servidor, luego agregar la clave SSH pública del servidor de OpenVPN al archivo <code>authorized_keys</code> y viceversa. Consulte <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04">Cómo configurar claves SSH en Ubuntu 20.04</a> para encontrar instrucciones sobre cómo aplicar cualquiera de estas soluciones.<br></span></p>

<h2 id="CREAMOS LA VPN OPENVPN UNA VEZ CREADA LA ENTIDAD CERTIFICADORA">CREAMOS LA VPN OPENVPN UNA VEZ CREADA LA ENTIDAD CERTIFICADORA</h2>

<p>El primer paso de este tutorial es instalar OpenVPN y Easy-RSA. Easy-RSA es una herramienta de gestión de infraestructura de clave pública (PKI) que usará en el servidor de OpenVPN para generar una solicitud de certificado que, luego, verificará y firmará en el servidor CA.</p>

<p>Para comenzar, actualice el índice de paquetes de su servidor de OpenVPN e instale y OpenVPN y Easy-RSA. Ambos paquetes están disponibles en los repositorios predeterminados de Ubuntu. Por lo tanto, puede utilizar <code>apt</code> para la instalación:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo apt update
</li><li class="line" data-prefix="$">sudo apt install openvpn easy-rsa
</li></ul></code></pre>
<p>A continuación, deberá crear un directorio nuevo en el servidor de OpenVPN como su non-root user llamado <code>/home/jrodenas/easy-rsa</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">mkdir /home/jrodenas/easy-rsa
</li></ul></code></pre>
<p>Ahora, deberá crear un symlink desde la secuencia de comandos de <code>easyrsa</code> que el paquete instaló en el directorio <code>/home/jrodenas/easy-rsa</code> que acaba de crear:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">ln -s /usr/share/easy-rsa/* /home/jrodenas/easy-rsa/
</li></ul></code></pre>
<p><span class='note'><strong>Nota:</strong> Aunque otras guías le indiquen que copie los archivos del paquete <code>easy-rsa</code> a su directorio de la PKI, en este tutorial, usaremos enlaces simbólicos. Como resultado, toda actualización del paquete <code>easy-rsa</code> se reflejará automáticamente en las secuencias de comandos de su PKI.<br></span></p>

<p>Por último, asegúrese de que el propietario del directorio sea su non-root sudo user y restrinja el acceso a ese usuario usando <code>chmod</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo chown <span class="highlight">jrodenas</span> /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">chmod 700 /home/jrodenas/easy-rsa
</li></ul></code></pre>
<p>Cuando haya instalado estos programas y se hayan movido a las ubicaciones correctas en su sistema, el siguiente paso es crear una infraestructura de clave pública (PKI) en el servidor de OpenVPN para que pueda solicitar y administrar certificados TLS para clientes y otros servidores que se conecten a su VPN.</p>

<h2 id="paso-2-crear-una-pki-para-openvpn">Paso 2: Crear una PKI para OpenVPN</h2>

<p>Antes de poder crear la clave y el certificado privados de su servidor de OpenVPN, deberá crear un directorio local de infraestructura de clave pública en su servidor de OpenVPN. Usará este directorio para gestionar las solicitudes de certificado de clientes y del servidor en vez de hacerlas directamente en su servidor CA.</p>

<p>Para crear un directorio de PKI en su servidor de OpenVPN, deberá completar un archivo llamado <code>vars</code> con algunos valores predeterminados. Primero, usará <code>cd</code> para ingresar al directorio <code>easy-rsa</code> y, luego, creará y editará el archivo <code>vars</code> utilizando nano o el editor de texto que prefiera.</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">nano vars
</li></ul></code></pre>
<p>Una vez abierto el archivo, pegue las siguientes dos líneas:</p>
<div class="code-label " title="/home/jrodenas/easy-rsa/vars">/home/jrodenas/easy-rsa/vars</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">set_var EASYRSA_ALGO "ec"
set_var EASYRSA_DIGEST "sha512"
</code></pre>
<p>Estas son las únicas dos líneas que necesita en este archivo <code>vars</code> en su servidor OpenVPN ya que no se usará como Autoridad de certificación. Estas líneas garantizarán que sus claves y solicitudes de certificado privadas estén configurados para utilizar la Elliptic Curve Cryptography (ECC) moderna para generar claves y firmas seguras para sus clientes y su servidor de OpenVPN.</p>

<p>Configurar sus servidores de OpenVPN y CA para que usen ECC significa que cuando un cliente y un servidor intenten establecer una clave simétrica compartida, pueden usar algoritmos de Elliptic Curve para realizar el intercambio. Usar ECC para un intercambio de claves es mucho más rápido que usar solo Diffie-Hellman con el clásico algoritmo RSA, dado que los números son mucho más pequeños y los cálculos son más rápidos.</p>

<span class='note'><p>
<strong>Antecedentes:</strong> Cuando los clientes se conectan a OpenVPN, usan el cifrado asimétrico (también conocido como la clave pública/privada) para realizar un protocolo de <a href="https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/">enlace TLS</a>. Sin embargo, al transmitir tráfico de VPN cifrado, el servidor y los clientes usan el cifrado simétrico, que también se conoce como cifrado de clave compartida.</p>

<p>Hay mucha menos estructura informática con cifrado simétrico que con el asimétrico: los números que se utilizan son mucho más pequeños y las CPU modernas <a href="https://en.wikipedia.org/wiki/AES_instruction_set">integran instrucciones para realizar operaciones de cifrado simétricas optimizadas</a>. Para hacer el cambio de cifrado de asimétrico a simétrico, el servidor de OpenVPN y el cliente usarán <a href="https://en.wikipedia.org/wiki/Elliptic_curve_Diffie-Hellman">el algoritmo Elliptic Curve Diffie-Hellman (ECDH)</a> para acordar una clave secreta compartida lo antes posible.<br></p></span>

<p>Cuando haya completado el archivo <code>vars</code>, puede continuar creando el directorio PKI. Para ello, ejecute la secuencia de comandos <code>easyrsa</code> con la opción <code>init-pki</code>. Aunque ya ejecutó este comando en el servidor CA como parte de los requisitos previos, debe ejecutarlo aquí porque su servidor de OpenVPN y el de CA tienen directorios PKI separados:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa init-pki
</li></ul></code></pre>
<p>Tenga en cuenta que no necesita crear una entidad de certificación en su servidor de OpenVPN. Su servidor de CA es el único responsable de validar y firmar certificados. La PKI de su servidor VPN solo se usa como una ubicación adecuada y centralizada para almacenar solicitudes de certificado y certificados públicos.</p>

<p>Después de iniciar su PKI en el servidor de OpenVPN, estará listo para continuar con el siguiente paso: crear una solicitud de certificado de servidor de OpenVPN y una clave privada.</p>

<h2 id="paso-3-crear-una-solicitud-de-certificado-de-servidor-de-openvpn-y-una-clave-privada">Paso 3: Crear una solicitud de certificado de servidor de OpenVPN y una clave privada</h2>

<p>Ahora que su servidor de OpenVPN tiene todos los requisitos previos instalados, el siguiente paso es generar una clave privada y una solicitud de firma de certificados (CSR) en su servidor de OpenVPN. A continuación, transferirá la solicitud a su CA para que se firme, lo que creará el certificado requerido. Cuando tenga un certificado firmado, lo transferirá de vuelta al servidor de OpenVPN y lo instalará para que el servidor lo use.</p>

<p>Para comenzar, diríjase al directorio <code>/home/jrodenas/easy-rsa</code> de su servidor de OpenVPN como su non-root user:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li></ul></code></pre>
<p>Invoque <code>easyrsa</code> con la opción <code>gen-req</code> seguida de un nombre común (CN) para la máquina. El CN puede ser el que prefiera, pero puede resultarle útil que sea descriptivo. Durante este tutorial, el CN del servidor de OpenVPN será <code>server</code>. Asegúrese de incluir también la opción <code>nopass</code>. Si no lo hace, se protegerá con contraseña el archivo de solicitud, lo que puede generar problemas de permisos más adelante.</p>

<p><span class='note'><strong>Nota:</strong> Si elige otro nombre que no sea <code>server</code>, deberá modificar algunas de las instrucciones a continuación. Por ejemplo, al copiar los archivos generados al directorio <code>/etc/openvpn</code>, deberá sustituir los nombres que correspondan. También deberá modificar el archivo <code>/etc/openvpn/server.conf</code> más adelante para señalar los archivos <code>.crt</code> y <code>.key</code> correctos.<br></span></p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa gen-req <span class="highlight">server</span> nopass
</li></ul></code></pre><pre class="code-pre  second-environment"><code><div class="secondary-code-label " title="Output">Output</div>Common Name (eg: your user, host, or server name) [server]:

Keypair and certificate request completed. Your files are:
req: /home/jrodenas/easy-rsa/pki/reqs/server.req
key: /home/jrodenas/easy-rsa/pki/private/server.key
</code></pre>
<p>Con esto, se crearán una clave privada para el servidor y un archivo de solicitud de certificado llamado <code>server.req</code>. Copie la clave del servidor al directorio <code>/etc/openvpn/server</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /home/jrodenas/easy-rsa/pki/private/server.key /etc/openvpn/server/
</li></ul></code></pre>
<p>Tras completar estos pasos, habrá creado correctamente una clave privada para su servidor de OpenVPN. También generó una solicitud de firma de certificado para el servidor de OpenVPN. Ahora la CSR está listo para firmar por su CA. En la siguiente sección de este tutorial, aprenderá a firmar una CSR con la clave privada de su servidor CA.</p>

<h2 id="paso-4-firmar-la-solicitud-de-certificado-del-servidor-de-openvpn">Paso 4: Firmar la solicitud de certificado del servidor de OpenVPN</h2>

<p>En el paso anterior, creó una solicitud de firma de certificado (CSR) y una clave privada para el servidor de OpenVPN. Ahora el servidor de CA necesita conocer el certificado <code>server</code> y validarlo. Una vez que el CA valide y devuelva el certificado al servidor de OpenVPN, los clientes que confían en su CA podrán confiar también en el servidor de OpenVPN.</p>

<p>En el servidor de OpenVPN, como su non-root user, use SCP u otro método de transferencia para copiar la solicitud de certificado <code>server.req</code> al servidor CA para firmar lo siguiente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">scp /home/jrodenas/easy-rsa/pki/reqs/server.req <span class="highlight">jrodenas</span>@<span class="highlight">your_ca_server_ip</span>:/tmp
</li></ul></code></pre>
<p>Si siguió el tutorial <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04">Cómo instalar y configurar una entidad de certificación (CA) en Ubuntu 20.04</a> de los requisitos previos, el siguiente paso será iniciar sesión en el <strong>servidor de CA</strong> como el non-root user que creó para administrar su CA. Utilice <code>cd</code> para el directorio <code>/home/jrodenas/easy-rsa</code> donde creó su PK y luego importe la solicitud de certificado utilizando la secuencia de comandos <code>easyrsa</code>:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">./easyrsa import-req /tmp/server.req server
</li></ul></code></pre><pre class="code-pre  third-environment"><code><div class="secondary-code-label " title="Output">Output</div>. . .
The request has been successfully imported with a short name of: server
You may now use this name to perform signing operations on this request.
</code></pre>
<p>A continuación, firme la solicitud ejecutando la secuencia de comandos <code>easyrsa</code> con la opción <code>sign-req</code> seguida del tipo de solicitud y el nombre común. El tipo de solicitud puede ser <code>client</code> o <code>server</code>. Como estamos trabajando con la solicitud de certificado del servidor de OpenVPN, asegúrese de usar el tipo de solicitud <code>server</code>:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa sign-req server <span class="highlight">server</span>
</li></ul></code></pre>
<p>En el resultado, se le solicitará que verifique que la solicitud provenga de una fuente de confianza. Escriba <code>yes</code> y pulse <code>ENTER</code> para confirmar:</p>
<pre class="code-pre  third-environment"><code><div class="secondary-code-label " title="Output">Output</div>You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 3650 days:

subject=
commonName = <span class="highlight">server</span>


Type the word 'yes' to continue, or any other input to abort.
Confirm request details: yes
. . .
Certificate created at: /home/jrodenas/easy-rsa/pki/issued/server.crt
</code></pre>
<p>Tenga en cuenta que si cifró su clave privada CA, se le solicitará su contraseña en este momento.</p>

<p>Completados estos pasos, ha firmado la solicitud de certificado del servidor de OpenVPN usando la clave privada del servidor de CA. El archivo <code>server.crt</code> resultante contiene la clave de cifrado pública del servidor de OpenVPN, así como una nueva firma del servidor de CA. El objetivo de la firma es indicar a todos los que confían en el servidor de CA que también pueden confiar en el servidor de OpenVPN cuando se conecten a él.</p>

<p>Para terminar de configurar los certificados, copie los archivos <code>server.crt</code> y <code>ca.crt</code> desde el servidor de CA al servidor de OpenVPN:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">scp pki/issued/server.crt <span class="highlight">jrodenas</span>@<span class="highlight">your_vpn_server_ip</span>:/tmp
</li><li class="line" data-prefix="$">scp pki/ca.crt <span class="highlight">jrodenas</span>@<span class="highlight">your_vpn_server_ip</span>:/tmp
</li></ul></code></pre>
<p>Ahora vuelva a su servidor de OpenVPN, copie los archivos de <code>/tmp</code> a <code>/etc/openvpn/server</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server
</li></ul></code></pre>
<p>Ahora su servidor de OpenVPN está casi listo para aceptar conexiones. En el siguiente paso, realizará algunos pasos adicionales para aumentar la seguridad del servidor.</p>

<h2 id="paso-5-configurar-materiales-de-cifrado-de-openvpn">Paso 5: Configurar materiales de cifrado de OpenVPN</h2>

<p>Para obtener una capa de seguridad adicional, añadiremos una clave secreta extra compartida que el servidor y todos los clientes usarán con <a href="https://build.openvpn.net/doxygen/group__tls__crypt.html#details">la directiva <code>tls-crypt</code> de OpenVPN</a>. Esta opción se usa para confundir el certificado TLS que se usa cuando un servidor y un cliente se conectan inicialmente. También lo usa el servidor de OpenVPN para realizar comprobaciones rápidas de los paquetes entrantes: si se firma un paquete usando la clave previamente compartida, el servidor lo procesa; si no se firma, el servidor sabe que es de una fuente no confiable y puede descartarlo sin tener que realizar otras tareas de descifrado.</p>

<p>Esta opción lo ayudará a asegurarse de que su servidor de OpenVPN pueda hacer frente al tráfico sin autenticación, a los escáneres de puerto y a los ataques de denegación de servicio, que pueden restringir recursos del servidor. También hace que sea más difícil identificar el tráfico de red de OpenVPN.</p>

<p>Para generar la clave <code>tls-crypt</code> antes compartida, ejecute lo siguiente en el servidor de OpenVPN en el directorio <code>/home/jrodenas/easy-rsa</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">openvpn --genkey --secret ta.key
</li></ul></code></pre>
<p>El resultado será un archivo llamado <code>ta.key</code>. Cópielo en el directorio <code>/etc/openvpn/server/</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp ta.key /etc/openvpn/server
</li></ul></code></pre>
<p>Con estos archivos implementados en el servidor de OpenVPN, estará listo para crear certificados de cliente y archivos de clave para sus usuarios, que usará para conectarse con la VPN.</p>

<h2 id="paso-6-generar-un-par-de-certificado-y-clave-de-cliente">Paso 6: Generar un par de certificado y clave de cliente</h2>

<p>Aunque puede generar una solicitud de claves y certificados privados en su máquina cliente y, luego, enviarla a la CA para que la firme, en esta guía se describe un proceso para generar la solicitud de certificado en el servidor de OpenVPN. El beneficio de este enfoque es que podemos crear una secuencia de comandos que generará de manera automática archivos de configuración de cliente que contienen las claves y los certificados necesarios. Esto le permite evitar la transferencia de claves, certificados y archivos de configuración a los clientes y optimiza el proceso para unirse a la VPN.</p>

<p>Generaremos un par individual de clave y certificado de cliente para esta guía. Si tiene más de un cliente, puede repetir este proceso para cada uno. Tenga en cuenta que deberá pasar un valor de nombre único a la secuencia de comandos para cada cliente. En este tutorial, el primer par de certificado y clave se denominará <code>client1</code>.</p>

<p>Comience por crear una estructura de directorios dentro de su directorio de inicio para almacenar los archivos de certificado y clave de cliente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">mkdir -p /home/jrodenas/client-configs/keys
</li></ul></code></pre>
<p>Debido a que almacenará los pares de certificado y clave de sus clientes y los archivos de configuración en este directorio, debe bloquear sus permisos ahora como medida de seguridad:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">chmod -R 700 /home/jrodenas/client-configs
</li></ul></code></pre>
<p>Luego, diríjase al directorio EasyRSA y ejecute la secuencia de comandos ​​​​​​<code>easyrsa</code> con las opciones <code>gen-req</code> y <code>nopass</code>, junto con el nombre común para el cliente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">./easyrsa gen-req <span class="highlight">client1</span> nopass
</li></ul></code></pre>
<p>Presione <code>ENTER</code> para confirmar el nombre común. Luego, copie el archivo <code>client1.key</code> al directorio <code>/home/jrodenas/client-configs/keys/</code> que creó antes:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cp pki/private/client1.key /home/jrodenas/client-configs/keys/
</li></ul></code></pre>
<p>Luego, transfiera el archivo <code>client1.req</code> a su servidor de CA usando un método seguro:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">scp pki/reqs/client1.req <span class="highlight">jrodenas</span>@<span class="highlight">your_ca_server_ip</span>:/tmp
</li></ul></code></pre>
<p>Ahora inicie sesión en su servidor de CA. A continuación, diríjase al directorio de EasyRSA e importe la solicitud de certificado:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/easy-rsa
</li><li class="line" data-prefix="$">./easyrsa import-req /tmp/client1.req client1
</li></ul></code></pre>
<p>Luego, firme la solicitud como lo hizo para el servidor en el paso anterior. Esta vez, asegúrese de especificar el tipo de solicitud <code>client</code>:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">./easyrsa sign-req client <span class="highlight">client1</span>
</li></ul></code></pre>
<p>Cuando se le solicite, ingrese <code>yes</code> para confirmar que desea firmar la solicitud de certificado y que esta provino de una fuente confiable:</p>
<pre class="code-pre  third-environment"><code><div class="secondary-code-label " title="Output">Output</div>Type the word 'yes' to continue, or any other input to abort.
Confirm request details: <span class="highlight">yes</span>
</code></pre>
<p>Nuevamente, si cifró su clave de CA, se le solicitará la contraseña en este punto.</p>

<p>Con esto, se creará un archivo de certificado de cliente llamado <code>client1.crt</code>. Transfiera este archivo de vuelta al servidor:</p>
<pre class="code-pre command prefixed third-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">scp pki/issued/client1.crt <span class="highlight">jrodenas</span>@<span class="highlight">your_server_ip</span>:/tmp
</li></ul></code></pre>
<p>Vuelva a su servidor de OpenVPN, copie el certificado del cliente al directorio <code>/home/jrodenas/client-configs/keys/</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cp /tmp/client1.crt /home/jrodenas/client-configs/keys/
</li></ul></code></pre>
<p>Luego copie los archivos <code>ca.crt</code> y <code>ta.key</code> al directorio <code>/home/jrodenas/client-configs/keys/</code> también y establezca los permisos correspondientes para su usuario sudo:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cp /home/jrodenas/easy-rsa/ta.key /home/jrodenas/client-configs/keys/
</li><li class="line" data-prefix="$">sudo cp /etc/openvpn/server/ca.crt /home/jrodenas/client-configs/keys/
</li><li class="line" data-prefix="$">sudo chown <span class="highlight">jrodenas</span>.<span class="highlight">jrodenas</span> /home/jrodenas/client-configs/keys/*
</li></ul></code></pre>
<p>Con esto, se generarán los certificados y las claves de su servidor y cliente, y se almacenarán en los directorios correspondientes de su servidor de OpenVPN. Aún quedan algunas acciones que se deben realizar con estos archivos, pero se realizarán más adelante. Por ahora, puede continuar con la configuración de OpenVPN.</p>

<h2 id="paso-7-configurar-openvpn">Paso 7: Configurar OpenVPN</h2>

<p>Al igual que muchas otras herramientas de código abierto ampliamente usadas, OpenVPN tiene muchas opciones de configuración disponibles para personalizar su servidor para sus necesidades específicas. En esta sección, le daremos instrucciones para instalar una configuración de un servidor de OpenVPN basada en uno de los archivos de configuración de muestra incluidos en la documentación de este software.</p>

<p>Primero, copie el archivo de muestra <code>server.conf</code> como punto de inicio para su propio archivo de configuración:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/server/
</li><li class="line" data-prefix="$">sudo gunzip /etc/openvpn/server/server.conf.gz
</li></ul></code></pre>
<p>Abra el archivo nuevo para editarlo con el editor de texto que prefiera. Usaremos nano en nuestro ejemplo:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo nano /etc/openvpn/server/server.conf
</li></ul></code></pre>
<p>Deberá cambiar algunas líneas en este archivo. Primero, encuentre la sección <code>HMAC</code> de la configuración buscando la directiva <code>tls-auth</code>. Esta línea no debería tener comentarios. Elimine los comentarios agregando un <code>;</code> al comienzo de la línea. Luego, añada una nueva línea después que contenga solo el valor <code>tls-crypt ta.key</code>:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">;</span>tls-auth ta.key 0 # This file is secret
<span class="highlight">tls-crypt ta.key</span>
</code></pre>
<p>Luego, busque las líneas <code>cipher</code> para encontrar la sección de cifrado. El valor predeterminado está configurado para <code>AES-256-CBC</code>; sin embargo, el cipher de <code>AES-256-GCM</code> ofrece un mejor nivel de cifrado, de rendimiento y es muy compatible con los clientes de OpenVPN actualizados. Eliminaremos el valor predeterminado añadiendo un signo <code>;</code> al inicio de esta línea y, luego, añadiremos otra línea que contenga el valor actualizado de <code>AES-256-GCM</code>:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">;</span>cipher AES-256-CBC
<span class="highlight">cipher AES-256-GCM</span>
</code></pre>
<p>Justo después de esta línea, añada una directiva <code>auth</code> para seleccionar el algoritmo de codificación de mensajes HMAC. <code>SHA256</code> es una buena opción:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">auth SHA256</span>
</code></pre>
<p>Luego, encuentre la línea que contenga la directiva <code>dh</code> que define los parámetros Diffie-Hellman. Como hemos configurado todos los certificados para usar Elliptic Curve Cryptography, no necesita un archivo seed Diffie-Hellman. Elimine la línea existente que se parece a <code>dh dh2048.pem</code> o <code>dh dh.pem</code>. El nombre del archivo para la clave Diffie-Hellman puede ser diferente al indicado en el archivo de configuración del servidor de ejemplo. Luego, añada una línea después con el contenido <code>dh none</code>:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">;</span>dh dh2048.pem
<span class="highlight">dh none</span>
</code></pre>
<p>Luego, queremos que OpenVPN se ejecute sin privilegios cuando se inicie, por lo que debemos indicarle que se ejecute con el usuario <strong>nobody</strong> y el grupo <strong>nogroup</strong>. Para ello, encuentre y elimine el comentario de las líneas <code>user nobody</code> y <code>group nogroup</code> eliminando el signo <code>;</code> en el inicio de cada línea:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">user nobody
group nogroup
</code></pre>
<h3 id="opcional-aplicar-cambios-de-dns-para-redireccionar-todo-el-tráfico-a-través-de-la-vpn">(Opcional) Aplicar cambios de DNS para redireccionar todo el tráfico a través de la VPN</h3>

<p>Con los ajustes anteriores, creará la conexión de VPN entre su cliente y el servidor, pero no se forzarán conexiones para usar el túnel. Si desea usar la VPN para dirigir todo su tráfico de clientes a la VPN, probablemente le convenga aplicar algunos ajustes más a las computadoras de clientes.</p>

<p>Para comenzar, encuentre y elimine el comentario de la línea que contenga <code>push "redirect-gateway def1 bypass-dhcp"</code>. Al hacerlo le indicará a su cliente que redirija todo su tráfico a través de su servidor de OpenVPN. Tenga en cuenta que habilitar esta funcionalidad puede causar problemas de conectividad con otros servicios de red, como SSH:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">push "redirect-gateway def1 bypass-dhcp"
</code></pre>
<p>Debajo de esta línea, encontrará la sección <code>dhcp-option</code>. Nuevamente, elimine <code>;</code> del inicio de ambas líneas para quitar los comentarios:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">push "dhcp-option DNS <span class="highlight">208.67.222.222</span>"
push "dhcp-option DNS <span class="highlight">208.67.220.220</span>"
</code></pre>
<p>Estas líneas le indicarán a su cliente que use los <a href="https://www.opendns.com">solucionadores OpenDNS</a> gratuitos en las direcciones IP en la lista. Si prefiere otros solucionadores DNS, puede sustituirlos en lugar de las IP resaltadas.</p>

<p>Esto ayudará a los clientes a configurar de nuevo sus ajustes de DNS para usar el túnel de la VPN como puerta de enlace predeterminada.</p>

<h3 id="opcional-ajustar-el-puerto-y-el-protocolo">(Opcional) Ajustar el puerto y el protocolo</h3>

<p>Por defecto, el servidor de OpenVPN usa el puerto <code>1194</code> y el protocolo UDP para aceptar las conexiones de los clientes. Si necesita usar un puerto diferente debido a restricciones de los entornos de red que sus clientes puedan emplear, puede cambiar la opción <code>port</code>. Si no aloja contenido web en su servidor de OpenVPN, el puerto <code>443</code> es una opción común, ya que se suele permitir en las reglas de firewall.</p>

<p>Para hacer que OpenVPN escuche en el puerto 443, abra el archivo <code>server.conf</code> y encuentre la línea que tiene el siguiente aspecto:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">port 1194
</code></pre>
<p>Edítela para que el puerto sea 443:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"># Optional!
port <span class="highlight">443</span>
</code></pre>
<p>Algunas veces, el protocolo se limita a ese puerto también. Si es así, encuentre la línea <code>proto</code> debajo de la línea <code>port</code> y cambie el protocolo de <code>udp</code> a <code>tcp</code>:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"># Optional!
proto <span class="highlight">tcp</span>
</code></pre>
<p>Si <strong>cambia</strong> el protocolo a TCP, deberá cambiar el valor de la directiva <code>explicit-exit-notify</code> de <code>1</code> a <code>0</code>, ya que solo UDP la usa. Si no lo hace al usar TCP, se producirán errores al iniciar el servicio de OpenVPN.</p>

<p>Busque la línea <code>explicit-exit-notify</code> al final del archivo y cambie el valor a <code>0</code>:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash"># Optional!
explicit-exit-notify <span class="highlight">0</span>
</code></pre>
<p>Si no tiene necesidad de usar un puerto y protocolo distintos, es mejor dejar estos ajustes sin cambiar.</p>

<h3 id="opcional-apuntar-a-credenciales-no-predeterminadas">(Opcional) Apuntar a credenciales no predeterminadas</h3>

<p>Si seleccionó antes un nombre diferente durante el comando <code>./easyrsa gen-req server</code>, modifique las líneas <code>cert</code> y <code>key</code> en el archivo de configuración <code>server.conf</code> para que apunten a los archivos <code>.crt</code> y <code>.key</code> correspondientes. Si usa el nombre predeterminado, <code>server</code>, ya está configurado correctamente:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">cert <span class="highlight">server</span>.crt
key <span class="highlight">server</span>.key
</code></pre>
<p>Cuando termine, guarde y cierre el archivo.</p>

<p>Ya ha terminado de configurar sus ajustes generales de OpenVPN. En el siguiente paso, personalizaremos las opciones de redes del servidor.</p>

<h2 id="paso-6-ajustar-la-configuración-de-redes-del-servidor-de-openvpn">Paso 6: Ajustar la configuración de redes del servidor de OpenVPN</h2>

<p>Hay algunos aspectos de la configuración de redes del servidor que deben modificarse para que OpenVPN pueda dirigir el tráfico de manera correcta a través de la VPN. El primero es el <em>reenvío IP</em>, un método para determinar a dónde se debe dirigir el tráfico de IP. Esto es esencial para la funcionalidad de VPN que proporcionará su servidor.</p>

<p>Para ajustar su configuración de IP predeterminada de su servidor de OpenVPN, abra el archivo <code>/etc/sysctl.conf</code> usando <code>nano</code> o su editor preferido:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo nano /etc/sysctl.conf
</li></ul></code></pre>
<p>Luego, añada la siguiente línea al final del archivo:</p>
<div class="code-label " title="/etc/sysctl.conf">/etc/sysctl.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">net.ipv4.ip_forward = 1
</code></pre>
<p>Guarde y cierre el archivo cuando termine.</p>

<p>Para leer el archivo y cargar los nuevos valores de la sesión actual, escriba lo siguiente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo sysctl -p
</li></ul></code></pre><pre class="code-pre  second-environment"><code><div class="secondary-code-label " title="Output">Output</div>net.ipv4.ip_forward = 1
</code></pre>
<p>Ahora, su servidor de OpenVPN podrá reenviar el tráfico entrante de un dispositivo ethernet a otro. Este ajuste garantiza que el servidor pueda dirigir tráfico desde los clientes que se conectan en la interfaz de VPN virtual fuera de sus otros dispositivos de ethernet físicos.  Esta configuración enviará todo el tráfico web de su cliente a través de la dirección IP de su servidor y la dirección IP pública de su cliente se ocultará de manera eficaz.</p>

<p>En el siguiente paso, deberá configurar algunas reglas de firewall para asegurarse de que el tráfico hacia y desde su servidor de OpenVPN fluye adecuadamente.</p>

<h2 id="paso-9-configuración-del-firewall">Paso 9: Configuración del firewall</h2>

<p>Hasta ahora, instaló OpenVPN en su servidor, lo configuró y generó las claves y los certificados necesarios para que su cliente acceda a la VPN. Sin embargo, aún no ha proporcionado ninguna instrucción a OpenVPN sobre a dónde enviar el tráfico web entrante de los clientes. Puede especificar cómo el servidor debería gestionar el tráfico de clientes estableciendo algunas reglas de firewall y configuraciones de enrutamiento.</p>

<p>Asumiendo que ha seguido los requisitos previos indicados al inicio de este tutorial, ya debería tener instalado <code>ufw</code> y ejecutándose en su servidor. Para permitir OpenVPN a través del firewall, necesitará habilitar el enmascaramiento, un concepto iptables que proporciona traducción de direcciones de red (NAT) para dirigir de manera correcta las conexiones de los clientes.</p>

<p>Antes de abrir el archivo de configuración de firewall para agregar las reglas de enmascaramiento, primero debe encontrar la interfaz de red pública de su máquina. Para hacer esto, escriba lo siguiente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">ip route list default
</li></ul></code></pre>
<p>Su interfaz pública es la secuencia de comandos que se encuentra en el resultado de este comando que sigue a la palabra “dev”. Por ejemplo, este resultado muestra la interfaz llamada <code>eth0</code>, que se resalta a continuación:</p>
<pre class="code-pre  second-environment"><code><div class="secondary-code-label " title="Output">Output</div>default via 159.65.160.1 dev <span class="highlight">eth0</span> proto static
</code></pre>
<p>Una vez que tenga la interfaz asociada con su ruta predeterminada, abra el archivo <code>/etc/ufw/before.rules</code> para agregar  la configuración pertinente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo nano /etc/ufw/before.rules
</li></ul></code></pre>
<p>Las reglas de UFW suelen agregarse usando el comando <code>ufw</code>. Sin embargo, las reglas enumeradas en el archivo <code>before.rules</code> se leen e implementan antes de que se carguen las reglas de UFW. En la parte superior del archivo, agregue las líneas resaltadas a continuación. Con esto, se establecerá la política predeterminada de la cadena <code>POSTROUTING</code> en la tabla <code>nat</code> y se enmascarará el tráfico que provenga de la VPN. Recuerde reemplazar <code><span class="highlight">eth0</span></code> en la línea <code>-A POSTROUTING</code> siguiente por la interfaz que encontró en el comando anterior:</p>
<div class="code-label " title="/etc/ufw/before.rules">/etc/ufw/before.rules</div><pre class="code-pre "><code class="code-highlight language-bash">#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

<span class="highlight"># START OPENVPN RULES</span>
<span class="highlight"># NAT table rules</span>
<span class="highlight">*nat</span>
<span class="highlight">:POSTROUTING ACCEPT [0:0]</span>
<span class="highlight"># Allow traffic from OpenVPN client to </span>eth0<span class="highlight"> (change to the interface you discovered!)</span>
<span class="highlight">-A POSTROUTING -s 10.8.0.0/8 -o </span>enp2s0<span class="highlight"> -j MASQUERADE</span>
<span class="highlight">COMMIT</span>
<span class="highlight"># END OPENVPN RULES</span>

</code></pre>
<p>Guarde y cierre el archivo cuando haya terminado.</p>

<p>Luego, debe indicarle a UFW que permita también los paquetes reenviados de modo predeterminado. Para hacer esto, abra el archivo <code>/etc/default/ufw</code>:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo nano /etc/default/ufw
</li></ul></code></pre>
<p>Dentro de este, encuentre la directiva <code>DEFAULT_FORWARD_POLICY</code> y cambie el valor de <code>DROP</code> a <code>ACCEPT</code>:</p>
<div class="code-label " title="/etc/default/ufw">/etc/default/ufw</div><pre class="code-pre "><code class="code-highlight language-bash">DEFAULT_FORWARD_POLICY="<span class="highlight">ACCEPT</span>"
</code></pre>
<p>Guarde y cierre el archivo cuando termine.</p>

<p>Luego, modifique el firewall para permitir el tráfico hacia OpenVPN. Si no cambió el puerto y el protocolo en el archivo <code>/etc/openvpn/server.conf</code>, deberá abrir el tráfico UDP al puerto <code>1194</code>. Si modificó el puerto o el protocolo, sustituya los valores que seleccionó aquí.</p>

<p>En caso de que se haya olvidado de agregar el puerto SSH al seguir el tutorial de los requisitos previos, agréguelo aquí también:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo ufw allow <span class="highlight">1194</span>/<span class="highlight">udp</span>
</li><li class="line" data-prefix="$">sudo ufw allow OpenSSH
</li></ul></code></pre>
<p>Luego de agregar esas reglas, deshabilite y vuelva a habilitar UFW para reiniciarlo y cargue los cambios de todos los archivos que haya modificado:</p>
<pre class="code-pre command prefixed"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo ufw disable
</li><li class="line" data-prefix="$">sudo ufw enable
</li></ul></code></pre>
<p>Su servidor quedará, así, configurado para manejar de manera correcta el tráfico de OpenVPN. Con las reglas de firewall implementadas, podemos iniciar el servicio de OpenVPN en el servidor.</p>

<h2 id="paso-10-iniciar-openvpn">Paso 10: Iniciar OpenVPN</h2>

<p>OpenVPN se ejecuta como servicio <code>systemd</code>, por lo que podemos usar <code>systemctl</code> para administrarlo. Configuraremos OpenVPN para que se inicie en el arranque para que pueda conectarse a su VPN en cualquier momento siempre que su servidor esté ejecutándose. Para ello, habilite el servicio de OpenVPN añadiéndolo a <code>systemctl</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo systemctl -f enable openvpn-server@server.service
</li></ul></code></pre>
<p>Luego, inicie el servicio de OpenVPN:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo systemctl start openvpn-server@server.service
</li></ul></code></pre>
<p>Vuelva a verificar que el servicio de OpenVPN está activo con el siguiente comando. Debería ver <code>active (running)</code> en el resultado:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo systemctl status openvpn-server@server.service
</li></ul></code></pre><pre class="code-pre  second-environment"><code><div class="secondary-code-label " title="Output">Output</div>
● openvpn-server@server.service - OpenVPN service for server
     Loaded: loaded (/lib/systemd/system/openvpn-server@.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2020-04-29 15:39:59 UTC; 6s ago
       Docs: man:openvpn(8)
             https://community.openvpn.net/openvpn/wiki/Openvpn24ManPage
             https://community.openvpn.net/openvpn/wiki/HOWTO
   Main PID: 16872 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1137)
     Memory: 1.0M
     CGroup: /system.slice/system-openvpn\x2dserver.slice/openvpn-server@server.service
             └─16872 /usr/sbin/openvpn --status /run/openvpn-server/status-server.log --status-version 2 --suppress-timestamps --c&gt;
. . .
. . .
Apr 29 15:39:59 ubuntu-20 openvpn[16872]: Initialization Sequence Completed
</code></pre>
<p>Ya completamos la configuración de la parte del servidor para OpenVPN. A continuación, configurará su máquina de cliente y se conectará con el servidor de OpenVPN.</p>

<h2 id="paso-11-crear-la-infraestructura-de-configuración-de-clientes">Paso 11: Crear la infraestructura de configuración de clientes</h2>

<p>Es posible que se deban crear archivos de configuración para clientes de OpenVPN, ya que todos los clientes deben tener su propia configuración y alinearse con los ajustes mencionados en el archivo de configuración del servicio. En este paso, en lugar de detallarse el proceso para escribir un único archivo de configuración que solo se pueda usar en un cliente, se describe un proceso para crear una infraestructura de configuración de cliente que puede usar para generar archivos de configuración sobre la marcha. Primero creará un archivo de configuración &ldquo;de base&rdquo; y, luego, una secuencia de comandos que le permitirá generar archivos de configuración, certificados y claves de clientes exclusivos según sea necesario.</p>

<p>Comience creando un nuevo directorio en el que almacenará archivos de configuración de clientes dentro del directorio <code>client-configs</code> creado anteriormente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">mkdir -p /home/jrodenas/client-configs/files
</li></ul></code></pre>
<p>Luego, copie un archivo de configuración de cliente de ejemplo al directorio <code>client-configs</code> para usarlo como su configuración de base:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /home/jrodenas/client-configs/base.conf
</li></ul></code></pre>
<p>Abra este archivo nuevo con <code>nano</code> o su editor de texto preferido:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">nano /home/jrodenas/client-configs/base.conf
</li></ul></code></pre>
<p>Dentro de este, ubique la directiva <code>remote</code>. Esto dirige al cliente a la dirección de su servidor de OpenVPN: la dirección IP pública de su servidor de OpenVPN. Si decidió cambiar el puerto en el que el servidor de OpenVPN escucha, también deberá cambiar <code>1194</code> por el puerto seleccionado:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash">. . .
# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
remote <span class="highlight">your_server_ip</span> <span class="highlight">1194</span>
. . .
</code></pre>
<p>Asegúrese de que el protocolo coincida con el valor que usa en la configuración del servidor:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash">proto <span class="highlight">udp</span>
</code></pre>
<p>Luego, elimine los comentarios de las directivas <code>user</code> y <code>group</code> quitando el signo <code>;</code> al inicio de cada línea:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"># Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup
</code></pre>
<p>Encuentre las directivas que establecen <code>ca</code>, <code>cert</code> y <code>key</code>. Elimine los comentarios de estas directivas, ya que pronto agregará los certificados y las claves dentro del archivo:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"># SSL/TLS parms.
# See the server config file for more
# description. It's best to use
# a separate .crt/.key file pair
# for each client. A single ca
# file can be used for all clients.
<span class="highlight">;</span>ca ca.crt
<span class="highlight">;</span>cert client.crt
<span class="highlight">;</span>key client.key
</code></pre>
<p>De modo similar, elimine la directiva <code>tls-auth</code>, ya que añadirá <code>ta.key</code> directamente al archivo de configuración del cliente (y se configura el servidor para que use <code>tls-crypt</code>):</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"># If a tls-auth key is used on the server
# then every client must also have the key.
<span class="highlight">;</span>tls-auth ta.key 1
</code></pre>
<p>Refleje los ajustes de <code>cipher</code> y <code>auth</code> establecidos en el archivo <code>/etc/openvpn/server/server.conf</code>:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">cipher AES-256-GCM</span>
<span class="highlight">auth SHA256</span>
</code></pre>
<p>Luego, agregue la directiva <code>key-direction</code> en algún lugar del archivo. Es <strong>necesario que</strong> fije el valor &ldquo;1&rdquo; para esta, a fin de que la VPN funcione de manera correcta en la máquina cliente:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">key-direction 1</span>
</code></pre>
<p>Por último, añada algunas líneas con <strong>comentarios eliminados</strong> para administrar varios métodos que los clientes VPN basados ​​en Linux utilizarán para la resolución DNS. Añadirá dos conjuntos de líneas con comentarios eliminados similares, pero separados. El primer conjunto es para los clientes que <em>no</em> utilizan <code>systemd-resolved</code> para administrar DNS. Estos clientes dependen de la utilidad <code>resolvconf</code> para actualizar la información DNS para clientes de Linux.</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">; script-security 2</span>
<span class="highlight">; up /etc/openvpn/update-resolv-conf</span>
<span class="highlight">; down /etc/openvpn/update-resolv-conf</span>
</code></pre>
<p>Luego, añada otro conjunto de líneas para clientes que utilicen <code>systemd-resolved</code> para la resolución de DNS:</p>
<pre class="code-pre  second-environment"><code class="code-highlight language-bash"><span class="highlight">; script-security 2</span>
<span class="highlight">; up /etc/openvpn/update-systemd-resolved</span>
<span class="highlight">; down /etc/openvpn/update-systemd-resolved</span>
<span class="highlight">; down-pre</span>
<span class="highlight">; dhcp-option DOMAIN-ROUTE .</span>
</code></pre>
<p>Guarde y cierre el archivo cuando termine.</p>

<p>Más adelante, en el <a href="#step-13-%E2%80%94-installing-the-client-configuration">Paso 13: paso para Instalar la configuración del cliente</a> de este tutorial, aprenderá a determinar cómo funciona la resolución DNS para los clientes Linux y qué sección no debería tener comentarios.</p>

<p>A continuación, cree una secuencia de comandos que compile su configuración de base con el certificado, la clave y los archivos de cifrado pertinentes, y, luego, ubique la configuración generada en el directorio <code>/home/jrodenas/client-configs/files</code>. Abra un nuevo archivo llamado <code>make_config.sh</code> en el directorio <code>/home/jrodenas/client-configs</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">nano /home/jrodenas/client-configs/make_config.sh
</li></ul></code></pre>
<p>Dentro de este, agregue el siguiente contenido:</p>
<pre class="code-pre sh second-environment"><code class="code-highlight language-bash">#!/bin/bash

# First argument: Client identifier

KEY_DIR=/home/jrodenas/client-configs/keys
OUTPUT_DIR=/home/jrodenas/client-configs/files
BASE_CONFIG=/home/jrodenas/client-configs/base.conf

cat ${BASE_CONFIG} \
    &lt;(echo -e '&lt;ca&gt;') \
    ${KEY_DIR}/ca.crt \
    &lt;(echo -e '&lt;/ca&gt;\n&lt;cert&gt;') \
    ${KEY_DIR}/${1}.crt \
    &lt;(echo -e '&lt;/cert&gt;\n&lt;key&gt;') \
    ${KEY_DIR}/${1}.key \
    &lt;(echo -e '&lt;/key&gt;\n&lt;tls-crypt&gt;') \
    ${KEY_DIR}/ta.key \
    &lt;(echo -e '&lt;/tls-crypt&gt;') \
    &gt; ${OUTPUT_DIR}/${1}.ovpn
</code></pre>
<p>Guarde y cierre el archivo cuando termine.</p>

<p>Antes de continuar, asegúrese de marcar este archivo como ejecutable escribiendo lo siguiente:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">chmod 700 /home/jrodenas/client-configs/make_config.sh
</li></ul></code></pre>
<p>Esta secuencia de comandos realizará una copia del archivo <code>base.conf</code> que creó, recopilará todos los archivos de certificados y claves que haya confeccionado para su cliente, extraerá el contenido de estos y los anexará a la copia del archivo de configuración de base, y exportará todo este contenido a un nuevo archivo de configuración de cliente. Esto significa que se evita la necesidad de administrar los archivos de configuración, certificado y clave del cliente por separado, y que toda la información necesaria se almacena en un solo lugar. El beneficio de este método es que, si alguna vez necesita agregar un cliente más adelante, puede simplemente ejecutar esta secuencia de comandos para crear de manera rápida el archivo de nueva configuración y asegurarse de que toda la información importante se almacene en una sola ubicación de acceso sencillo.</p>

<p>Tenga en cuenta que siempre que agregue un nuevo cliente, deberá generar claves y certificados nuevos para poder ejecutar esta secuencia de comandos y generar su archivo de configuración. Podrá practicar con este comando en el siguiente paso.</p>

<h2 id="paso-12-generar-las-configuraciones-de-clientes">Paso 12: Generar las configuraciones de clientes</h2>

<p>Si siguió la guía, creó un certificado y una clave de cliente llamados <code>client1.crt</code> y <code>client1.key</code>, respectivamente, en el paso 6. Puede generar un archivo de configuración para estas credenciales si se dirige al directorio <code>/home/jrodenas/client-configs</code> y ejecuta la secuencia de comandos que realizó al final del paso anterior:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">cd /home/jrodenas/client-configs
</li><li class="line" data-prefix="$">./make_config.sh <span class="highlight">client1</span>
</li></ul></code></pre>
<p>Con esto, se creará un archivo llamado <code>client1.ovpn</code> en su directorio <code>/home/jrodenas/client-configs/files</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">ls /home/jrodenas/client-configs/files
</li></ul></code></pre><pre class="code-pre  second-environment"><code><div class="secondary-code-label " title="Output">Output</div>client1.ovpn
</code></pre>
<p>Debe transferir este archivo al dispositivo que planee usar como cliente. Por ejemplo, puede ser su computadora local o un dispositivo móvil.</p>

<p>Si bien las aplicaciones exactas empleadas para lograr esta transferencia dependerán del sistema operativo de su dispositivo y sus preferencias personales, un método seguro y confiable consiste en usar el protocolo de transferencia de archivos SSH (SFTP ) o la copia segura (SCP) en el backend. Con esto, se transportarán los archivos de autenticación de VPN de su cliente a través de una conexión cifrada.</p>

<p>Aquí tiene un comando SFTP de ejemplo que puede ejecutar desde su computadora local (macOS o Linux). Esto copiará el archivo <code><span class="highlight">client1.ovpn</span></code> que hemos creado en el último paso a su directorio de inicio:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="local$">sftp <span class="highlight">jrodenas</span>@<span class="highlight">openvpn_server_ip</span>:client-configs/files/client1.ovpn /home/jrodenas/
</li></ul></code></pre>
<p>A continuación, se muestran diferentes herramientas y tutoriales para transferir de manera segura los archivos del servidor de OpenVPN a una computadora local:</p>

<ul>
<li><a href="http://winscp.net">WinSCP</a></li>
<li><a href="https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server">Cómo usar SFTP para transferir archivos de manera segura con un servidor remoto</a></li>
<li><a href="https://www.digitalocean.com/community/tutorials/how-to-use-filezilla-to-transfer-and-manage-files-securely-on-your-vps">Cómo usar Filezilla para transferir y administrar archivos de manera segura en su VPS</a></li>
</ul>

<h2 id="paso-13-instalar-la-configuración-de-cliente">Paso 13: Instalar la configuración de cliente</h2>

<p>En esta sección, se aborda la forma de instalar un perfil de VPN de cliente en Windows, macOS, Linux, iOS y Android. Ninguna de estas instrucciones para clientes depende de la otra. Por lo tanto, no dude en dirigirse directamente a la que corresponda para su dispositivo.</p>

<p>La conexión de OpenVPN tendrá el mismo nombre que usó para el archivo <code>.ovpn</code>. En lo que respecta a este tutorial, esto significa que la conexión se llama <code>client1.ovpn</code> y guarda correspondencia con el primer archivo de cliente que generó.</p>

<h3 id="windows">Windows</h3>

<p><strong>Instalación</strong></p>

<p>Descargue la aplicación de cliente de OpenVPN para Windows de la <a href="https://openvpn.net/index.php/open-source/downloads.html">página de descargas de OpenVPN</a>. Seleccione la versión adecuada del instalador para su versión de Windows.</p>

<p><span class='note'><strong>Nota</strong>: OpenVPN necesita privilegios administrativos para instalarse.<br></span></p>

<p>Luego de instalar OpenVPN, copie el archivo <code>.ovpn</code> a esta ubicación:</p>
<pre class="code-pre  local-environment"><code>C:\Program Files\OpenVPN\config
</code></pre>
<p>Cuando inicie OpenVPN, este detectará el perfil de manera automática y lo dejará disponible.</p>

<p>Debe ejecutar OpenVPN como administrador cada vez que lo use, aun en cuentas administrativas. Para realizar esto sin tener que hacer clic con el botón secundario y seleccionar <strong>Ejecutar como administrador</strong> cada vez que use la VPN, debe fijarlo como ajuste predeterminado desde una cuenta administrativa. Esto también significa que los usuarios estándares deberán ingresar la contraseña del administrador para usar OpenVPN. Por otro lado, los usuarios estándares no pueden conectarse de manera adecuada al servidor a menos que la aplicación OpenVPN del cliente tenga derechos de administrador. Por lo tanto, se necesitan privilegios elevados.</p>

<p>Para configurar la aplicación OpenVPN de modo que se ejecute siempre con privilegios de administrador, haga clic con el botón secundario en su ícono de acceso directo y diríjase a <strong>Propiedades</strong>. Al final de la pestaña <strong>Compatibilidad</strong>, haga clic en el botón <strong>Cambiar la configuración para todos los usuarios</strong>. En la nueva ventana, seleccione <strong>Ejecutar este programa como administrador</strong>.</p>

<p><strong>Conexión</strong></p>

<p>Cada vez que inicie OpenVPN GUI, Windows le preguntará si quiere que el programa realice cambios en su computadora. Haga clic en <strong>Sí</strong>. Iniciar la aplicación OpenVPN de cliente solo ubica el applet en la bandeja del sistema para que pueda conectar y desconectar la VPN según sea necesario; no establece la conexión de VPN.</p>

<p>Una vez que se inicie OpenVPN, establezca una conexión ingresando al área de notificación y haga clic con el botón secundario en el ícono de OpenVPN. Con esto, se abrirá el menú contextual. Seleccione <strong>client1</strong> en la parte superior del menú (su perfil <code>client1.ovpn</code>) y, luego, <strong>Connect</strong>.</p>

<p>Una ventana de estado se abrirá y mostrará el resultado de registro mientras se establece la conexión, y se mostrará un mensaje una vez que el cliente esté conectado.</p>

<p>Desconéctese de la VPN de la misma forma: ingrese al applet de la bandeja del sistema, haga clic con el botón secundario en el icono de OpenVPN, seleccione el perfil del cliente y haga clic en <strong>Disconnect</strong>.</p>

<h3 id="macos">macOS</h3>

<p><strong>Instalación</strong></p>

<p><a href="https://tunnelblick.net/">Tunnelblick</a> es un cliente de OpenVPN gratuito y de código abierto para macOS. Puede descargar la última imagen de disco desde la <a href="https://tunnelblick.net/downloads.html">página de descargas de Tunnelblick</a>. Haga clic en el archivo <code>.dmg</code> descargado y siga las instrucciones para instalarlo.</p>

<p>Al finalizar el proceso de instalación, Tunnelblick le preguntará si tiene algún archivo de configuración. Responda <strong>I have configuration files</strong> (tengo archivos de configuración) y deje que Tunnelblick finalice el proceso. Abra una ventana de Finder, busque <code>client1.ovpn</code> y haga doble clic en él. Tunnelblick instalará el perfil de cliente. Se necesitan privilegios de administrador.</p>

<p><strong>Conexión</strong></p>

<p>Inicie Tunnelblick haciendo doble clic en el icono de Tunnelblick de la carpeta <strong>Aplicaciones</strong>. Una vez que se haya iniciado Tunnelblick, su icono aparecerá en la barra de menú de la esquina superior derecha de la pantalla para controlar las conexiones. Haga clic en el icono y, luego, en el elemento de menú <strong>Connect client1</strong> para iniciar la conexión de VPN.</p>

<h3 id="linux">Linux</h3>

<p><strong>Instalación</strong></p>

<p>Si usa Linux, dispone de varias herramientas según su distribución. En su entorno de escritorio o gestor de ventanas, también pueden incluirse utilidades de conexión.</p>

<p>Sin embargo, el método de conexión más universal consiste en simplemente usar el software OpenVPN.</p>

<p>En Ubuntu o Debian, puede instalarlo como en el servidor escribiendo lo siguiente:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">sudo apt update
</li><li class="line" data-prefix="client$">sudo apt install openvpn
</li></ul></code></pre>
<p>En CentOS, puede habilitar los repositorios EPEL y, luego, instalarlo escribiendo lo siguiente:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">sudo dnf install epel-release
</li><li class="line" data-prefix="client$">sudo dnf install openvpn
</li></ul></code></pre>
<h4 id="configurar-clientes-que-utilizan-systemd-resolved">Configurar clientes que utilizan <code>systemd-resolved</code></h4>

<p>Primero, determine si su sistema utiliza <code>systemd-resolved</code> para gestionar la resolución DNS comprobando el archivo <code>/etc/resolv.conf</code>:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client1$">cat /etc/resolv.conf
</li></ul></code></pre><pre class="code-pre  local-environment"><code><div class="secondary-code-label " title="Output">Output</div># This file is managed by man:systemd-resolved(8). Do not edit.
. . .

nameserver <span class="highlight">127.0.0.53</span>
options edns0
</code></pre>
<p>Si su sistema está configurado para utilizar <code>systemd-resolved</code> para la resolución DNS, la dirección IP después de la opción <code>nameserver</code> será <code>127.0.0.53</code>. También debería haber comentarios en el archivo, como el resultado que se muestra que explica cómo <code>systemd-resolved</code> administra el archivo. Si tiene una dirección IP diferente a <code>127.0.0.53</code>, entonces lo más probable es que su sistema no esté utilizando <code>systemd-resolved</code> y, en cambio, puede acudir a la siguiente sección sobre cómo configurar clientes Linux que tienen la secuencia de comandos <code>update-resolv-conf</code>.</p>

<p>Para admitir estos clientes, primero instale el paquete <code>openvpn-systemd-resolved</code>. Proporciona secuencias de comandos que forzarán <code>systemd-resolved</code> a utilizar el servidor VPN para la resolución DNS.</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">sudo apt install openvpn-systemd-resolved
</li></ul></code></pre>
<p>Una vez que el paquete esté instalado, configure el cliente para usarlo y envíe todas las consultas DNS a la interfaz VPN. Abra el archivo VPN del cliente:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">nano <span class="highlight">client1</span>.ovpn
</li></ul></code></pre>
<p>Ahora, elimine el comentario de las siguientes líneas que añadió anteriormente:</p>
<div class="code-label " title="client1.ovpn">client1.ovpn</div><pre class="code-pre  local-environment"><code class="code-highlight language-bash">script-security 2
up /etc/openvpn/update-systemd-resolved
down /etc/openvpn/update-systemd-resolved
down-pre
dhcp-option DOMAIN-ROUTE .
</code></pre>
<h4 id="configurar-clientes-que-utilizan-update-resolv-conf">Configurar clientes que utilizan <code>update-resolv-conf</code></h4>

<p>Si su sistema no está utilizando <code>systemd-resolved</code> para administrar DNS, compruebe si su distribución incluye en su lugar la secuencia de comandos <code>/etc/openvpn/update-resolv-conf</code>:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client1$">ls /etc/openvpn
</li></ul></code></pre><pre class="code-pre  local-environment"><code><div class="secondary-code-label " title="Output">Output</div>update-resolv-conf
</code></pre>
<p>Si su cliente incluye el archivo <code>update-resolv-conf</code>, edite el archivo de configuración del cliente de OpenVPN que transfirió anteriormente:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">nano <span class="highlight">client1</span>.ovpn
</li></ul></code></pre>
<p>Elimine los comentarios de las tres líneas que añadió para ajustar la configuración de DNS:</p>
<div class="code-label " title="client1.ovpn">client1.ovpn</div><pre class="code-pre  local-environment"><code class="code-highlight language-bash">script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
</code></pre>
<p>Si usa CentOS, cambie la directiva <code>group</code> de <code>nogroup</code> a <code>nobody</code> para que coincidan los grupos de distribución disponibles:</p>
<div class="code-label " title="client1.ovpn">client1.ovpn</div><pre class="code-pre  local-environment"><code class="code-highlight language-bash">group <span class="highlight">nobody</span>
</code></pre>
<p>Guarde y cierre el archivo.</p>

<p><strong>Conexión</strong></p>

<p>Ahora, podrá conectarse a la VPN simplemente apuntando el comando <code>openvpn</code> hacia el archivo de configuración de cliente:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">sudo openvpn --config <span class="highlight">client1</span>.ovpn
</li></ul></code></pre>
<p>Esto debería permitirle establecer conexión con la VPN.</p>

<span class='note'><p>
<strong>Nota:</strong> Si su cliente utiliza <code>systemd-resolved</code> para administrar DNS, compruebe que la configuración se aplique correctamente ejecutando el comando <code>systemd-resolve --status</code> de la siguiente manera:</p>
<pre class="code-pre custom_prefix prefixed local-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="client$">systemd-resolve --status tun0
</li></ul></code></pre>
<p>Debería ver un resultado como el siguiente:</p>
<pre class="code-pre  local-environment"><code><div class="secondary-code-label " title="Output">Output</div>Link 22 (<span class="highlight">tun0</span>)
. . .
         DNS Servers: <span class="highlight">208.67.222.222</span>
                      <span class="highlight">208.67.220.220</span>
          DNS Domain: <span class="highlight">~.</span>
</code></pre>
<p>Si ve las direcciones IP de los servidores DNS que configuró en el servidor de OpenVPN, junto con la configuración <code>~.</code> para el <em>Dominio DNS</em> en el resultado, habrá configurado correctamente su cliente para utilizar la resolución DNS del servidor de VPN. También puede verificar si usted está enviando consultas DNS a través de VPN utilizando un sitio como <a href="https://www.dnsleaktest.com/">DNSleaktest.com</a><br></p></span>

<h3 id="ios">iOS</h3>

<p><strong>Instalación</strong></p>

<p>Desde iTunes App Store, busque e instale <a href="https://itunes.apple.com/us/app/id590379981">OpenVPN Connect</a>, la aplicación oficial de cliente de OpenVPN de iOS. Para transferir su configuración de cliente iOS al dispositivo, conéctelo directamente a una computadora.</p>

<p>El proceso para completar la transferencia con iTunes se describe aquí. Abra iTunes en la computadora y haga clic en <strong>iPhone</strong> &gt; <strong>apps</strong>. Deslícese hacia la parte inferior, hasta la sección <strong>Compartir archivos</strong>, y haga clic en la app OpenVPN. La ventana en blanco de la derecha, <strong>Documentos OpenVPN</strong>, sirve para compartir archivos. Arrastre el archivo <code>.ovpn</code> a la ventana de documentos de OpenVPN. <img src="https://assets.digitalocean.com/articles/openvpn_ubunutu/1.png" alt="iTunes con el perfil de VPN listo para cargar en el iPhone"></p>

<p>Ahora inicie la aplicación OpenVPN en el iPhone. Recibirá una notificación de que un nuevo perfil está listo para importarse. Toque el símbolo verde del signo de suma para importarlo.</p>

<p><img src="https://assets.digitalocean.com/articles/openvpn_ubunutu/2.png" alt="La app OpenVPN de iOS muestra un nuevo perfil listo para importarse">

<p>De esta manera, OpenVPN estará listo para usarse con el nuevo perfil. Inicie la conexión deslizando el botón <strong>Connect</strong> (Conectar) a la posición <strong>On</strong> (Activado). Finalice la conexión deslizando el mismo botón a la posición <strong>Off</strong> (Desactivado).</p>

<p><span class='note'><strong>Nota</strong>: El conmutador de VPN de <strong>Settings</strong> (Ajustes) no se puede usar para establecer conexión con la VPN. Si lo intenta, recibirá un aviso que le indicará conectarse únicamente utilizando la aplicación OpenVPN.<br></span></p>

<p><img src="https://assets.digitalocean.com/articles/openvpn_ubunutu/3.png" alt="Aplicación OpenVPN de iOS conectada a la VPN"></p>

<h3 id="android">Android</h3>

<p><strong>Instalación</strong></p>

<p>Abra Google Play Store. Busque e instale <a href="https://play.google.com/store/apps/details?id=net.openvpn.openvpn">Android OpenVPN Connect</a>, la aplicación de cliente de OpenVPN oficial de Android.</p>

<p>Puede transferir el perfil <code>.ovpn</code> conectando el dispositivo Android a su computadora a través de un puerto USB y copiando el archivo. De manera alternativa, si tiene un lector de tarjetas SD, puede quitar la tarjeta SD del dispositivo, copiar el perfil a ella y, luego, insertarla tarjeta de vuelta en el dispositivo Android.</p>

<p>Inicie la aplicación OpenVPN y haga clic en el menú <code>FILE</code> (Archivo) para importar el perfil.</p>

<p><img src="https://assets.digitalocean.com/articles/openvpn_android/01-initial_screen.jpg" alt="Selección del menú de importación de perfiles de la aplicación OpenVPN de Android">.</p>

<p>Luego, diríjase a la ubicación del perfil guardado (en la captura de pantalla se usa <code>/storage/emulated/0/openvpn</code>​​​) y seleccione su archivo <code>.ovpn</code>. Haga clic en el botón <code>IMPORT</code> para terminar de importar este perfil.</p>

<p><img src="https://assets.digitalocean.com/articles/openvpn_android/02-import_file_screen.jpg" alt="Selección de un perfil de VPN para su importación en la aplicación OpenVPN de Android">.</p>

<p><strong>Connecting</strong> (Conectando) Cuando se añada el perfil, verá una pantalla como esta:</p>

<p><img src="https://assets.digitalocean.com/articles/openvpn_android/04-profile_added.jpg" alt="La aplicación de Android de OpenVPN con un nuevo perfil añadido"></p>

<p>Para conectarse, haga clic en el botón de activar para el perfil que desea usar. Verá datos en tiempo real de su conexión y tráfico enrutados a través de su servidor de OpenVPN:</p>
<p><img src="https://assets.digitalocean.com/articles/openvpn_android/05-connected.jpg" alt="la aplicación de Android de OpenVPN conectada a la VPN"></p>

<p>Para desconectarse, solo haga clic en el botón de alternancia en la parte superior izquierda de nuevo. Se le solicitará que confirme que desea desconectarse de su VPN.</p>

<h2 id="paso-14-probar-la-conexión-de-su-vpn-opcional">Paso 14: Probar la conexión de su VPN (opcional)</h2>

<p><span class='note'><strong>Nota</strong>: Este método para probar su conexión de VPN solo funcionará si optó por enrutar todo el tráfico a través de la VPN en el paso 7 cuando editó el archivo <code>server.conf</code> para OpenVPN.<br></span></p>

<p>Una vez que todo esté instalado, con una simple revisión confirmará que todo funciona de forma correcta. Sin tener una conexión VPN habilitada, abra un explorador e ingrese a <a href="https://www.dnsleaktest.com">DNSLeakTest</a>.</p>

<p>El sitio mostrará la dirección de IP asignada por su proveedor de servicio de Internet y la forma en que aparece para el resto del mundo. Para corroborar sus ajustes de DNS a través del mismo sitio web, haga clic en <strong>Extended Test</strong> (Prueba extendida). Esto le indicará los servidores DNS que usa.</p>

<p>Ahora, conecte el cliente de OpenVPN a la VPN de su Droplet y actualice el navegador. Con esto, debería aparecer una dirección de IP totalmente distinta (la de su servidor de VPN). De esta manera, aparecerá ante el mundo. Una vez más, la opción <strong>Extended Test</strong> de <a href="https://www.dnsleaktest.com">DNSLeakTest</a> revisará sus ajustes de DNS y confirmará que ahora use los solucionadores de DNS enviados por su VPN.</p>

<h2 id="paso-15-revocar-certificados-de-clientes">Paso 15: Revocar certificados de clientes</h2>

<p>Es posible que, de tanto en tanto, deba rechazar un certificado de cliente para evitar más accesos al servidor de OpenVPN.</p>

<p>Para ello, siga el ejemplo en el tutorial de requisitos previos <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04#optional-%E2%80%94-revoking-a-certificate">Cómo instalar y configurar una entidad de certificación en Ubuntu 20.04</a> en la sección <em>Revocar un certificado</em>.</p>

<p>Cuando haya revocado un certificado para un cliente usando estas instrucciones, deberá copiar el archivo <code>crl.pem</code> generado a su servidor de OpenVPN en el directorio <code>/etc/openvpn/server</code>:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo cp /tmp/crl.pem /etc/openvpn/server/
</li></ul></code></pre>
<p>Luego, abra el archivo de configuración del servidor de OpenVPN:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo nano /etc/openvpn/server/server.conf
</li></ul></code></pre>
<p>Al final del archivo, agregue la opción <code>crl-verify</code>, que indicará al servidor OpenVPN que revise la lista de rechazo de certificados que creamos cada vez que se realice un intento de conexión:</p>
<div class="code-label " title="/etc/openvpn/server/server.conf">/etc/openvpn/server/server.conf</div><pre class="code-pre  second-environment"><code class="code-highlight language-bash">crl-verify crl.pem
</code></pre>
<p>Guarde y cierre el archivo.</p>

<p>Por último, reinicie OpenVPN para implementar el rechazo de certificados:</p>
<pre class="code-pre command prefixed second-environment"><code class="code-highlight language-bash"><ul class="prefixed"><li class="line" data-prefix="$">sudo systemctl restart openvpn-server@server.service
</li></ul></code></pre>
<p>El cliente ya no debería poder conectarse de manera correcta al servidor usando la credencial anterior.</p>

<p>Para rechazar clientes adicionales, siga este proceso:</p>

<ol>
<li>Rechace el certificado con el comando <code>./easyrsa revoke <span class="highlight">client_name</span></code>.</li>
<li>Genere una nueva CRL.</li>
<li>Transfiera el archivo <code>crl.pem</code> nuevo a su servidor de OpenVPN y cópielo al directorio <code>/etc/openvpn/server/</code> para sobrescribir la lista anterior.</li>
<li>Reinicie el servicio de OpenVPN.</li>
</ol>

<p>Puede usar este proceso para rechazar cualquier certificado emitido anteriormente para su servidor.</p>

<h2 id="conclusión">Conclusión</h2>

<p>Ahora debería tener una red privada completamente funcional ejecutándose en su servidor de OpenVPN. Puede navegar en la web y descargar contenido sin preocuparse porque individuos malintencionados rastreen su actividad.</p>

<p>Hay varios pasos que podría seguir para personalizar su instalación de OpenVPN aún más, como configurar su cliente para que se conecte a la VPN de manera automática o configurar reglas específicas para los clientes y políticas de acceso. Para esto y para otras personalizaciones de OpenVPN, debería consultar <a href="https://openvpn.net/index.php/open-source/documentation.html">la documentación oficial de OpenVPN</a>.</p>

<p>Para configurar más clientes, solo debe seguir los pasos <strong>6</strong> y <strong>11-13</strong> para cada dispositivo adicional. Para rechazar el acceso de los clientes, siga el paso <strong>15</strong>.</p>

<h2 id="conclusión">Solucionar error: Please enter password with the systemd-tty-ask-password-agent tool.</h2>
<ol>
<li>Creamos el archivo: <code>sudo nano /etc/openvpn/auth.txt</code>.</li>
<li>Añadimos la passwd y guardamos el archivo</li>
<li>Damos permisos al archivo</li><code>sudo chmod 600 /etc/openvpn/auth.txt</code>
<li>Abrimos el archivo del server<code>sudo nano /etc/openvpn/server/server.conf</code></li>
<li>Añadimos el askpass al final del archivo <code>askpass /etc/openvpn/auth.txt</code></li>
<li>Reinicie el servicio de OpenVPN.</li><code>sudo service openvpn restart</code>
</ol>
<p>¡Ya no hay mensaje para usar systemd-tty-ask-password-agent y se ha iniciado la sesión automáticamente!.</p>

</div>

  </div>
