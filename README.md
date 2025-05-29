# gulp-ssh

====

## Install

Install with [npm](https://npmjs.org/package/gulp-ssh)

```sh
npm install --save-dev gulp-ssh
```

## Example

```js
'use strict'

var fs = require('fs');
var gulp = require('gulp')
var GulpSSH = require('gulp-ssh')

var config = {
  host: '192.168.0.21',
  port: 22,
  username: 'node',
  privateKey: fs.readFileSync('/Users/zensh/.ssh/id_rsa')
}

var gulpSSH = new GulpSSH({
  ignoreErrors: false,
  sshConfig: config
})

gulp.task('exec', function () {
  return gulpSSH
    .exec(['uptime', 'ls -a', 'pwd'], {filePath: 'commands.log'})
    .pipe(gulp.dest('logs'))
})

gulp.task('dest', function () {
  return gulp
    .src(['./**/*.js', '!**/node_modules/**'])
    .pipe(gulpSSH.dest('/home/iojs/test/gulp-ssh/'))
})

gulp.task('sftp-read', function () {
  return gulpSSH.sftp('read', '/home/iojs/test/gulp-ssh/index.js', {filePath: 'index.js'})
    .pipe(gulp.dest('logs'))
})

gulp.task('sftp-write', function () {
  return gulp.src('index.js')
    .pipe(gulpSSH.sftp('write', '/home/iojs/test/gulp-ssh/test.js'))
})

gulp.task('shell', function () {
  return gulpSSH
    .shell(['cd /home/iojs/test/thunks', 'git pull', 'npm install', 'npm update', 'npm test'], {filePath: 'shell.log'})
    .pipe(gulp.dest('logs'))
})
```

## API

```js
var GulpSSH = require('gulp-ssh')
```

### GulpSSH(options)

```js
var gulpSSH = new GulpSSH(options)
```

#### options.sshConfig

*Required*

Type: `Object`

* **host** - `String` - Nombre o dirección IP del servidor. **Predeterminado:** 'localhost'

* **port** - `Number` - Número de puerto del servidor. **Predeterminado:** 22

* **username** - `String` - Nombre de usuario para la autenticación. **Predeterminado:** (none)

* **password** - `String` - Contraseña para la autenticación de usuarios basada en contraseña. **Predeterminado:** (none)

* **privateKey** - `String` or `Buffer` - Buffer o cadena que contiene una clave privada para la autenticación de usuarios basada en clave (formato OpenSSH). **Predeterminado:** (none)

* **privateKeyFile** - `String` - Ruta a un archivo que contiene una clave privada para la autenticación de usuarios basada en clave (formato OpenSSH). Extensión gulp-ssh.  **Predeterminado:** (none)

* **useAgent** - `Boolean` - Detecta automáticamente el agente SSH en ejecución (mediante la variable de entorno SSH_AUTH_SOCK) y lo usa para realizar la autenticación. Extensión gulp-ssh. **Predeterminado:** (falso)

* ...and so forth.

Para obtener una lista completa de las opciones de conexión, consulte la referencia de método[connect()](https://github.com/mscdex/ssh2#client-methods)

#### options.ignoreErrors

Type: `Boolean`

Ignorar errores al ejecutar comandos. **Predeterminado:** (falso)

*****

### gulpSSH.shell(commands, options)

Devuelve `stream`, hay un evento "ssh2Data" en el stream que emite el fragmento del stream ssh2.

**IMPORTANTE:** Si uno de los comandos requiere la interacción del usuario, esta función se bloqueará.
Observe el evento ssh2Data para depurar la interacción con el servidor.

#### comandos

*Required*
Tipo: `String` o `Array`

#### options.filePath

*Option*
Type: `String`

file path to write on local. **Default:** ('gulp-ssh.shell.log')

#### options.autoExit

*Option*
Type: `Boolean`

auto exit shell. **Default:** (true)

### gulpSSH.exec(commands, options)

Devuelve `stream`, hay un evento "ssh2Data" en el stream que emite el fragmento del stream ssh2.

**IMPORTANTE:** Si uno de los comandos requiere la interacción del usuario, esta función se bloqueará.
Observe el evento ssh2Data para depurar la interacción con el servidor.

#### commands

*Required*
Type: `String` or `Array`

#### options.filePath

*Option*
Type: `String`

Ruta del archivo para escribir en local. **Default:** ('gulp-ssh.exec.log')


### gulpSSH.sftp(command, filePath, options)

return `stream`

#### command

*Required*
Type: `String`
Value: 'read' or 'write'

#### filePath

*Required*
Type: `String`

Ruta del archivo para escribir en server. **Default:** (none)

#### options

*Option*
Type: `Object`

### gulpSSH.dest(destDir, options)

devuelve `stream`, copia los archivos al control remoto a través de sftp, actúa de manera similar a Gulp dest, creará directorios si no existen.

## Tests

Esta biblioteca es un cliente de transferencia SSH/SFTP.
Por lo tanto, necesitamos conectarnos a un servidor SSH para probarla.

La estrategia para probar esta biblioteca consiste en conectarse a la máquina actual como el usuario actual a través de SSH.
Esto permite que las pruebas verifiquen ambos extremos de la transferencia SSH.

Para ejecutar las pruebas, se necesita un servidor SSH local en ejecución y una clave SSH que las pruebas puedan usar para autenticarse.
(Las instrucciones de esta sección son específicas para Linux).

Primero, generemos una clave SSH sin contraseña para las pruebas.

```sh
mkdir -p test/etc/ssh
ssh-keygen -t rsa -b 4096 -N "" -f test/etc/ssh/id_rsa -q
chmod 600 test/etc/ssh/id_rsa*
```

A continuación, agregue esta clave a las claves SSH autorizadas para el usuario actual:

```sh
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat test/etc/ssh/id_rsa.pub > ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Si ya tienes un servidor SSH ejecutándose en tu equipo, puedes usarlo para ejecutar las pruebas.
Hagamos una prueba para asegurarnos de que podemos conectarnos.

```sh
ssh -i etc/test/ssh $USER@localhost uptime
```

Deberías ver el tiempo de actividad de la máquina actual impreso en la consola.
Si funciona, ¡estás listo para ejecutar las pruebas!

```sh
yarn test
```

Si no tiene un servidor SSH en ejecución, puede ejecutar uno en un puerto de espacio de usuario (2222) para las pruebas.
Comience creando una configuración de servidor y una clave de host:

```sh
mkdir -p test/etc/sshd
cat << EOF > test/etc/sshd/sshd_config
Port 2222
ListenAddress 127.0.0.1
HostKey $(pwd)/test/etc/sshd/host_rsa
PidFile $(pwd)/test/etc/sshd/pid
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
Subsystem sftp /usr/lib/openssh/sftp-server
UsePAM no
EOF
ssh-keygen -t rsa -b 4096 -N "" -f test/etc/sshd/host_rsa -q
```

A continuación, inicie el servidor:

```sh
/usr/sbin/sshd -f test/etc/sshd/sshd_config
```

Si falta el comando sshd, deberá instalar el paquete openssh-server para su distribución.

Intentemos conectarnos y asegurarnos de que funciona correctamente:

```sh
ssh -i etc/test/ssh -p 2222 $USER@localhost uptime
```

Deberías ver el tiempo de actividad de la máquina actual impreso en la consola.
Si funciona, ¡estás listo para ejecutar las pruebas!

```sh
CI=true yarn test
```

Pasamos CI=true para que las pruebas utilicen el puerto 2222 para conectarse en lugar del puerto predeterminado.

Cuando haya terminado de ejecutar las pruebas, puede usar este comando para detener el servidor SSH:

```sh
kill $(cat test/etc/sshd/pid)
```

En el directorio test/scripts puedes encontrar el proceso de configuración y desmontaje que se utiliza en CI para ejecutar estas pruebas, que es similar al que se describe en esta sección.

## License

MIT © [Teambition](https://www.teambition.com)

[npm-url]: https://npmjs.org/package/gulp-ssh
[npm-image]: http://img.shields.io/npm/v/gulp-ssh.svg

[downloads-url]: https://npmjs.org/package/gulp-ssh
[ci-url]: https://travis-ci.org/teambition/gulp-ssh
[ci-image]: https://img.shields.io/travis/teambition/gulp-ssh/master.svg
