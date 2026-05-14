# Informe técnico de solución propia
## CVE-2026-31431 "Copy Fail"
### Evaluación práctica de UNIX - Compilación, ejecución en QEMU, explotación controlada y parche del kernel Linux 6.12

**Luis Novillo – 13/05/2026**

---

## 1. Contexto

La evaluación consistía en reproducir en un ambiente controlado la vulnerabilidad CVE-2026-31431, conocida como "Copy Fail". El objetivo era demostrar el ciclo completo de trabajo con un kernel vulnerable: compilarlo, ejecutarlo en una máquina virtual, comprobar la vulnerabilidad, aplicar un parche permanente y verificar que el mismo script de prueba ya no consiguiera privilegios de root.

El procedimiento original indicaba usar comandos automáticos del repositorio entregado por el profesor, como `make setup` y `make qemu`. Sin embargo, ese entorno presentó errores en Codespaces. Por esa razón, se construyó una solución propia y manual, manteniendo la lógica de la evaluación: kernel vulnerable, prueba controlada, parche y verificación.

---

## 2. Solución propuesta

La solución propuesta consistió en reconstruir manualmente el entorno vulnerable descargando y compilando manualmente el kernel Linux 6.12 desde el código fuente utilizando GitHub Codespaces. Posteriormente, se habilitó el soporte criptográfico necesario (AF_ALG) requerido por el exploit, se creó un root filesystem funcional y se arrancó el kernel compilado mediante una máquina virtual (QEMU). Una vez validado el entorno vulnerable, se ejecutó el exploit Python dentro de la máquina virtual, logrando escalar privilegios desde el usuario `student` hasta `root`. Finalmente, se identificó y corrigió la lógica vulnerable en el archivo `crypto/algif_aead.c`, se recompiló el kernel y se verificó que el exploit ya no podía obtener privilegios elevados, demostrando así que el parche aplicado solucionaba correctamente la vulnerabilidad CVE-2026-31431 "Copy Fail".

---

## 3. Preparación del entorno en Codespaces

Primero se instalaron las herramientas necesarias para compilar el kernel, crear el root filesystem y arrancar QEMU.

```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev bc dwarves cpio \
gzip qemu-system-x86 busybox-static debootstrap kmod
```

**Propósito de los paquetes principales:**

- `build-essential`, `bison`, `flex`, `bc`, `libssl-dev`, `libelf-dev` y `dwarves`: dependencias para compilar Linux.
- `qemu-system-x86`: permite arrancar el kernel compilado dentro de una VM.
- `debootstrap`: permite construir un rootfs mínimo de Ubuntu/Debian.
- `cpio` y `gzip`: sirven para empaquetar el rootfs como initramfs.
- `kmod`: provee herramientas como `depmod` y `modprobe`.

---

## 4. Descarga y compilación manual del kernel Linux 6.12

Como el repositorio original fallaba, se descargó manualmente el código fuente oficial de Linux 6.12 y se compiló desde cero.

```bash
mkdir -p ~/kernel-lab
cd ~/kernel-lab
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.tar.xz
tar -xf linux-6.12.tar.xz
cd linux-6.12
make defconfig
make -j2
```

**Explicación de los comandos:**

- `mkdir -p ~/kernel-lab` crea el directorio de trabajo.
- `wget` descarga el código fuente del kernel Linux 6.12.
- `tar -xf` extrae el archivo comprimido.
- `make defconfig` genera una configuración base para compilar el kernel.
- `make -j2` compila el kernel usando dos procesos en paralelo, opción más estable para Codespaces.

Después de la compilación se verificó que los archivos principales existieran:

```bash
ls -lh arch/x86/boot/bzImage
ls -lh vmlinux
make kernelrelease
```

- `bzImage` confirmó que el kernel arrancable fue generado correctamente.
- `vmlinux` confirmó que también existe la imagen del kernel sin comprimir.
- `make kernelrelease` permitió confirmar la versión compilada.

---

## 5. Activación de AF_ALG y soporte criptográfico necesario

Al probar el script inicialmente apareció el error `"Address family not supported by protocol"`. Ese error significaba que el kernel arrancado no tenía disponible AF_ALG, la familia de sockets usada para acceder al subsistema criptográfico desde espacio de usuario.

Primero se intentó usar módulos, pero en el entorno mínimo de QEMU los módulos existían y aun así no se cargaban correctamente. La solución más estable fue compilar esas opciones directamente dentro del kernel, usando `--enable` en lugar de `--module`.

```bash
cd ~/kernel-lab/linux-6.12
scripts/config --enable CONFIG_CRYPTO_USER_API && scripts/config --enable CONFIG_CRYPTO_USER_API_AEAD \
&& scripts/config --enable CONFIG_CRYPTO_USER_API_SKCIPHER && scripts/config --enable \
CONFIG_CRYPTO_AUTHENC && scripts/config --enable CONFIG_CRYPTO_AUTHENCESN && scripts/config --enable \
CONFIG_CRYPTO_CBC && scripts/config --enable CONFIG_CRYPTO_SHA256 && scripts/config --enable \
CONFIG_CRYPTO_AES
make olddefconfig
make -j2
```

Después de recompilar, se verificó dentro de QEMU que AF_ALG estuviera disponible ejecutando una prueba mínima de Python:

```python
python3 - <<'PY'
import socket
print(socket.AF_ALG)
s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
print("AF_ALG OK")
PY
```

La salida `"AF_ALG OK"` confirmó que el kernel ya aceptaba sockets AF_ALG y que el script de prueba podía ejecutarse en el entorno correcto.

---

## 6. Creación del rootfs para la VM

El kernel por sí solo no basta para arrancar un sistema usable. También se necesitó un root filesystem con herramientas básicas, Python y un usuario normal `student`. Para eso se preparó un rootfs mínimo.

```bash
mkdir ~/rootfs
sudo debootstrap --arch=amd64 jammy ~/rootfs http://archive.ubuntu.com/ubuntu/
```

Luego se ingresó al rootfs con chroot para instalar herramientas y crear el usuario `student`:

```bash
sudo chroot ~/rootfs
apt update
apt install -y python3 sudo passwd kmod
useradd -m student
passwd student
exit
```

El usuario `student` fue importante porque la evaluación pedía demostrar que un usuario sin privilegios podía pasar a root. Si la VM inicia directamente como root, no se puede demostrar la escalación de privilegios.

Finalmente, se empaquetó el rootfs como initramfs:

```bash
cd ~/rootfs
find . | cpio -H newc -ov | gzip > ~/rootfs.cpio.gz
```

---

## 7. Diferencia entre archivos en Codespaces y archivos dentro de QEMU

Una parte importante fue entender que Codespaces y la VM de QEMU no comparten automáticamente el mismo sistema de archivos. Un archivo que existe en `/workspaces` dentro de Codespaces no aparece automáticamente dentro de `/home/student` en la VM.

Por eso, si el script Python estaba en Codespaces, había que copiarlo al rootfs antes de arrancar QEMU o crearlo manualmente dentro de la VM.

```bash
cp /workspaces/copy-fail-challenge-B/copy_fail_exp.py ~/rootfs/home/student/copy_fail_exp.py
chown 1000:1000 ~/rootfs/home/student/copy_fail_exp.py
cd ~/rootfs
find . | cpio -H newc -ov | gzip > ~/rootfs.cpio.gz
```

También se podía crear el archivo directamente dentro de QEMU con:

```bash
cat > copy_fail_exp.py
# pegar el contenido del script
# terminar con Ctrl + D
```

---

## 8. Arranque del kernel vulnerable con QEMU

Después de compilar el kernel y preparar el rootfs, se arrancó la VM con QEMU usando el `bzImage` compilado.

```bash
qemu-system-x86_64 -kernel ~/kernel-lab/linux-6.12/arch/x86/boot/bzImage -initrd ~/rootfs.cpio.gz -m \
2048 -nographic -append "console=ttyS0 rdinit=/sbin/init"
```

**Este comando hace lo siguiente:**

- `-kernel` indica qué kernel arrancar; en este caso, el `bzImage` compilado manualmente.
- `-initrd` indica el rootfs empaquetado.
- `-m 2048` asigna 2048 MB de RAM a la VM.
- `-nographic` permite usar la VM desde la terminal.
- `console=ttyS0` redirige la consola de la VM a la terminal.
- `rdinit=/sbin/init` indica el proceso inicial dentro del rootfs.

---

## 9. Comprobación del entorno vulnerable

Dentro de la VM se inició sesión como `student` y se comprobó la identidad y la versión del kernel.

```bash
whoami
id
uname -r
```

El resultado esperado antes de ejecutar el exploit era que `whoami` mostrara `student` y que `id` mostrara un usuario normal, no root. Además, `uname -r` confirmaba que la VM estaba ejecutando el kernel 6.12 compilado manualmente.

```bash
# Evidencia recomendada antes del exploit
echo "=== HITO 1: KERNEL VULNERABLE ARRANCADO ===" > /tmp/hito1.txt
date >> /tmp/hito1.txt
uname -r >> /tmp/hito1.txt
whoami >> /tmp/hito1.txt
id >> /tmp/hito1.txt
cat /tmp/hito1.txt
```

---

## 10. Demostración de la vulnerabilidad con el script Python

Luego se ejecutó el script Python del PoC dentro de la VM. El script no se ejecutó en Codespaces directamente, porque necesita interactuar con el kernel vulnerable en ejecución dentro de QEMU.

```bash
ls -lh copy_fail_exp.py
python3 copy_fail_exp.py
whoami
id
```

Antes de ejecutar el script, la identidad era `student`. Después de ejecutar el script en el kernel vulnerable, la identidad cambió a `root`. Esto demostró que la vulnerabilidad estaba presente y que el entorno construido manualmente era funcional.

---

## 11. Aplicación del parche permanente

Después de demostrar la vulnerabilidad, se salió de QEMU con `Ctrl + A` y luego `X`. El parche se aplicó en el código fuente del kernel, específicamente en el archivo:

```
crypto/algif_aead.c
```

La función relacionada era `_aead_recvmsg()`. El problema estaba en el uso incorrecto de scatterlists dentro de la operación AEAD. Conceptualmente, el exploit funcionaba porque el kernel seguía una ruta en la que los buffers de entrada y salida podían mezclarse de forma peligrosa.

Primero se hizo una copia de seguridad del archivo original:

```bash
cd ~/kernel-lab/linux-6.12
cp crypto/algif_aead.c crypto/algif_aead.c.bak
```

Luego se ubicó la línea relacionada con `aead_request_set_crypt`:

```bash
grep -n "aead_request_set_crypt" crypto/algif_aead.c
```

Se editó el archivo con `vi` porque `nano` no estaba disponible:

```bash
vi crypto/algif_aead.c
```

El cambio aplicado fue conceptual y puntual: cambiar el origen usado por la operación AEAD. La forma vulnerable usaba `rsgl_src` como origen. La corrección usó `tsgl->sg` como origen correcto.

```c
// Línea corregida
aead_request_set_crypt(&areq->cra_u.aead_req, tsgl->sg,
    areq->first_rsgl.sgl.sgl, used, ctx->iv);
```

La razón técnica del cambio es que `tsgl->sg` representa correctamente la lista de entrada/transmisión, mientras que `areq->first_rsgl.sgl.sgl` representa el destino de recepción. Así se evita que la operación use la ruta vulnerable que permitía la corrupción de memoria o page cache.

---

## 12. Generación del archivo de parche y recompilación

Después de modificar el archivo, se generó un diff para dejar evidencia exacta del cambio aplicado.

```bash
mkdir -p ~/kernel-lab/patches
diff -u crypto/algif_aead.c.bak crypto/algif_aead.c > ~/kernel-lab/patches/fix_algif_aead.patch
cat ~/kernel-lab/patches/fix_algif_aead.patch
```

Luego se recompiló el kernel parcheado:

```bash
make -j2
```

Durante la recompilación aparecieron errores porque primero se intentó usar `tsgl` directamente, pero el compilador indicó que ese tipo no era `struct scatterlist *`. Luego se intentó `tsgl->sgl.sgl`, pero el compilador indicó que el miembro correcto era `sg`. Finalmente la línea correcta fue `tsgl->sg` y la compilación terminó correctamente.

Después se verificó nuevamente que el kernel arrancable existiera:

```bash
ls -lh arch/x86/boot/bzImage
```

---

## 13. Verificación del parche en QEMU

Se arrancó nuevamente QEMU con el `bzImage` recompilado. Como el comando de arranque siempre apuntaba al mismo archivo `arch/x86/boot/bzImage`, al recompilar el kernel ya se estaba usando la versión parcheada.

```bash
qemu-system-x86_64 -kernel ~/kernel-lab/linux-6.12/arch/x86/boot/bzImage -initrd ~/rootfs.cpio.gz -m \
2048 -nographic -append "console=ttyS0 rdinit=/sbin/init"
```

Dentro de la VM, se volvió a comprobar que el usuario inicial era `student`:

```bash
whoami
id
uname -r
```

Luego se ejecutó nuevamente el script de Python. Esta vez, después de ejecutarlo, `whoami` e `id` seguían mostrando `student`. Es decir, el exploit ya no logró obtener root.

```bash
python3 copy_fail_exp.py
whoami
id
```

---

## 14. Explicación corta de por qué el parche funcionó

El script Python funcionaba porque el kernel vulnerable permitía que una operación criptográfica AEAD usara una ruta incorrecta de buffers/scatterlists. Esa ruta hacía posible que el exploit provocara una escritura peligrosa en memoria asociada al page cache y terminara afectando el comportamiento de un binario con permisos setuid, logrando pasar de `student` a `root`.

Al cambiar la llamada a `aead_request_set_crypt` para usar `tsgl->sg` como origen, se separó correctamente la entrada de la salida. El exploit dependía de la confusión anterior; al eliminar esa condición, el script podía ejecutarse, pero ya no conseguía la escalación de privilegios.

---

## 16. Conclusión final

La solución propia reemplazó el flujo automático roto del repositorio original. Se compiló Linux 6.12 manualmente, se activó el soporte necesario de AF_ALG, se creó un rootfs con usuario `student`, se arrancó el kernel en QEMU y se demostró la escalación de privilegios dentro de un laboratorio controlado.

Luego se corrigió el archivo `crypto/algif_aead.c`, se recompiló el kernel y se verificó que el mismo script ya no lograra cambiar la identidad de `student` a `root`. Esto demuestra que el parche aplicado neutralizó la condición vulnerable usada por el exploit.

---

## Bibliografía

- OpenAI. (2026). ChatGPT (modelo GPT-5.5) https://chat.openai.com/
- Theori. (2026). copy_fail_exp.py GitHub. https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py
- The Linux Kernel Organization. (2026). Linux kernel 6.12 source code [Código fuente]. kernel.org. https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.tar.xz