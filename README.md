# Copy Fail Lab — CVE-2026-31431 (v2)

Devcontainer reproducible para experimentar con la vulnerabilidad **Copy Fail**
(CVE-2026-31431) en un kernel Linux 6.12 controlado dentro de QEMU.

Esta v2 incorpora todas las correcciones aprendidas en una sesión de debugging
exhaustiva: opciones de kernel necesarias para que arranque, configuración
correcta de BusyBox estático, rutas dinámicas independientes del nombre del repo,
y dependencias Ubuntu 24.04 corregidas.

---

## Inicio rápido para el estudiante

1. Abre un Codespace desde este repo.
   ```bash
   #CONFIGURACION DE EJEMPLO!!!!!!!!!!!
   apt update
   apt install gh
   
   gh api user --jq '"\(.name) → \(.email // .login)"'
   
   git config --global user.name "Jonathan E. Tito O."
   git config --global user.email "jonathantito@users.noreply.github.com"
   git config --global --add safe.directory /workspaces/copy-fail-challenge-1
   make setup
   ```
3. Configura tu identidad git:
   ```bash
   git config --global user.name "Tu Nombre"
   git config --global user.email "tu@correo.com"
   ```
4. Ejecuta:
   ```bash
   make setup    # descarga kernel + arma rootfs (~5 min)
   make qemu     # arranca la VM vulnerable
   ```

Para salir de QEMU: `Ctrl+A` luego `X`.

---

## Configuración inicial del docente (una sola vez)

### 1. Subir este repo a GitHub

```bash
cd copyfail-v2
git init && git add -A && git commit -m "initial"
git branch -M main
gh repo create TU-ORG/copy-fail-lab --public --source=. --push
```

### 2. Marcarlo como Template

GitHub → tu repo → Settings → marcar `Template repository`.

### 3. Editar `.devcontainer/devcontainer.json`

Cambia el valor `KERNEL_REPO`:
```json
"KERNEL_REPO": "TU-ORG/copy-fail-lab"
```

Commit y push.

### 4. Disparar el workflow del kernel

GitHub → Actions → `Build Vulnerable Kernel` → Run workflow.
Tarda ~25 min en los servidores de GitHub (no en tu Codespace).
Al terminar crea un Release con el `bzImage_vuln` listo para descarga.

### 5. Verificar

Tu repo → Releases → debe aparecer `kernel-v6.12-vuln` con tres archivos
adjuntos. Los estudiantes ahora pueden hacer `make setup` y descarga en 2 min.

---

## Estructura del repo

```
.
├── .devcontainer/
│   ├── Dockerfile             ← Ubuntu 24.04 + deps verificadas
│   └── devcontainer.json      ← sin rutas hardcodeadas
├── .github/workflows/
│   └── build-kernel.yml       ← compila kernel y crea Release
├── scripts/
│   ├── 00_welcome.sh
│   ├── 01_fetch_kernel.sh     ← descarga del Release
│   ├── 02_build_kernel.sh     ← fallback: compila desde fuente
│   ├── 03_build_rootfs.sh     ← BusyBox estático + initramfs
│   └── 04_run_qemu.sh
├── Makefile
└── README.md
```

---

## Comandos disponibles

| Comando | Acción |
|---|---|
| `make setup` | Descarga kernel + arma rootfs (~5 min) |
| `make qemu` | Arranca la VM vulnerable |
| `make info` | Muestra el estado del ambiente |
| `make rootfs` | Reconstruye solo el initramfs |
| `make fetch-kernel` | Solo descarga el bzImage del Release |
| `make build-kernel` | Compila kernel desde fuente (~25 min) |
| `make clean` | Borra builds (mantiene fuentes) |
| `make clean-all` | Borra todo |

---

## Recursos del CVE

- Write-up técnico: https://xint.io/blog/copy-fail-linux-distributions
- Sitio del CVE: https://copy.fail
- PoC oficial: https://github.com/theori-io/copy-fail-CVE-2026-31431

---

## Lecciones aprendidas (referencia para futuras versiones)

Esta v2 incorpora los siguientes fixes respecto a la v1:

- `hexdump` → `bsdextrautils` en Ubuntu 24.04
- `bzip2` agregado al Dockerfile (lo necesita BusyBox)
- Eliminado el `mounts` con ruta hardcodeada en `devcontainer.json`
- Todos los scripts detectan workspace con `SCRIPT_DIR` dinámico
- Kernel: agregadas opciones críticas `BINFMT_ELF`, `BINFMT_SCRIPT`, `RD_GZIP`
- Kernel: agregada dep `CRYPTO_AEAD` antes de `CRYPTO_AUTHENCESN`
- BusyBox: reemplazado `scripts/config` (no existe) por `sed`
- BusyBox: eliminado `olddefconfig` (no existe en BusyBox)
- BusyBox: deshabilitado `CONFIG_TC` (rompe compilación con kernels nuevos)
- BusyBox: forzado `CONFIG_STATIC=y` y verificado con `file`
- Workflow Actions: greps de verificación con `|| echo`, tolerantes






11/05/2026
 1  apt update
    2  apt install gh
    3  gh api user --jq '"\(.name) → \(.email // .login)"'
    4  git config --global user.name "Luis A. Novillo C."
    5  git config --global user.email "luis3novillo@gmail.com"
    6  git config --global --add safe.directory /workspaces/copy-fail-challenge-1
    7  make setup
    8  make qemu
    9  apt update
   10  apt install -y file
   11  make rootfs
   12  make qemu
   13  make setup
   14  make qemu
   15  make clean
   16  make setup
   17  make qemu
   18  make clean
   19  make setup
   20  make qemu
   21  find . -name "*.qcow2"
   22  find . -name "*rootfs*"
   23  sed -n '1,220p' scripts/03_build_rootfs.sh
   24  grep -n "chmod\|su\|student\|python" scripts/03_build_rootfs.sh
   25  find build -name su -exec ls -l {} \;
   26  find . -type f -name su -exec ls -l {} \;
   27  find . -type d -name "*initramfs*"
   28  find . -name "initramfs.cpio.gz" -o -name "*.cpio.gz"
   29  mkdir /tmp/initramfs_test
   30  cd /tmp/initramfs_test
   31  gzip -dc /workspaces/copy-fail-challenge/kernel/build/initramfs.cpio.gz | cpio -idmv
   32  rm -rf /tmp/initramfs_test
   33  mkdir /tmp/initramfs_test
   34  cd /tmp/initramfs_test
   35  gzip -dc /workspaces/copy-fail-challenge-B/kernel/build/initramfs.cpio.gz | cpio -idmv
   36  ls -l bin/su
   37  cd /tmp
   38  wget https://copy.fail/exp -O copy_fail_exp.py
   39  head -40 copy_fail_exp.py
   40  make qemu
   41  cd..
   42  cd ..
   43  cd /workspaces/copy-fail-challenge-B
   44  make qemu
   45  cd /tmp/initramfs_test
   46  mkdir -p usr/bin
   47  ln -s /bin/su usr/bin/su
   48  find . | cpio -o -H newc 2>/dev/null | gzip > /workspaces/copy-fail-challenge-B/kernel/build/initramfs.cpio.gz
   49  cd /workspaces/copy-fail-challenge-B
   50  make qemu
   51  cd /tmp/initramfs_test
   52  chmod 755 .
   53  chmod 755 home
   54  chmod 755 home/student
   55  chmod 755 root
   56  chmod 755 usr
   57  chmod 755 usr/bin
   58  chmod 755 bin
   59  chown -R root:root .
   60  chown 1001:1001 home/student
   61  ls -l usr/bin/su
   62  ls -ld . home home/student usr usr/bin
   63  find . | cpio -o -H newc 2>/dev/null | gzip > /workspaces/copy-fail-challenge-B/kernel/build/initramfs.cpio.gz
   64  cd /workspaces/copy-fail-challenge-B
   65  make qemu
   66  grep -R "python3\|copy_fail\|exp\|wget" -n .
   67  ls -R scripts
   68  ls
   69  which python3
   70  python3 --version
   71  ldd $(which python3)
   72  cd /tmp/initramfs_test
   73  mkdir -p usr/bin usr/lib lib/x86_64-linux-gnu lib64
   74  cp /usr/bin/python3 usr/bin/python3
   75  cp -a /usr/lib/python3.12 usr/lib/
   76  cp /lib/x86_64-linux-gnu/libm.so.6 lib/x86_64-linux-gnu/
   77  cp /lib/x86_64-linux-gnu/libz.so.1 lib/x86_64-linux-gnu/
   78  cp /lib/x86_64-linux-gnu/libexpat.so.1 lib/x86_64-linux-gnu/
   79  cp /lib/x86_64-linux-gnu/libc.so.6 lib/x86_64-linux-gnu/
   80  cp /lib64/ld-linux-x86-64.so.2 lib64/
   81  cp /tmp/copy_fail_exp.py home/student/copy_fail_exp.py
   82  chown 1001:1001 home/student/copy_fail_exp.py
   83  find . | cpio -o -H newc 2>/dev/null | gzip > /workspaces/copy-fail-challenge-B/kernel/build/initramfs.cpio.gz
   84  cd /workspaces/copy-fail-challenge-B
   85  make qemu
   86  cd /tmp/initramfs_test
   87  chmod 755 usr/bin/python3
   88  chmod 755 usr
   89  chmod 755 usr/bin
   90  chmod 755 usr/lib
   91  chmod -R a+rX usr/lib/python3.12
   92  chmod 755 lib lib/x86_64-linux-gnu lib64
   93  chmod 755 lib64/ld-linux-x86-64.so.2
   94  chmod 755 lib/x86_64-linux-gnu/*.so*
   95  find . | cpio -o -H newc 2>/dev/null | gzip > /workspaces/copy-fail-challenge-B/kernel/build/initramfs.cpio.gz
   96  cd /workspaces/copy-fail-challenge-B
   97  make qemu
   98  find kernel -name "algif_aead*"
   99  find kernel -name "authencesn*"
  100  find kernel -name "*.ko" | grep -E "alg|auth"
  101  grep -E "CONFIG_CRYPTO_USER_API_AEAD|CONFIG_CRYPTO_AUTHENC|CONFIG_CRYPTO_AUTHENCESN|CONFIG_CRYPTO_CBC|CONFIG_CRYPTO_SHA256|CONFIG_CRYPTO_AES" kernel/linux/.c
  102  grep -E "CONFIG_CRYPTO_USER_API_AEAD|CONFIG_CRYPTO_AUTHENC|CONFIG_CRYPTO_AUTHENCESN|CONFIG_CRYPTO_CBC|CONFIG_CRYPTO_SHA256|CONFIG_CRYPTO_AES" kernel/linux/.cg
  103  grep CONFIG_CRYPTO_AUTHENCESN kernel/linux/.config
  104  cd /workspaces/copy-fail-challenge-B/kernel/linux
  105  scripts/config --enable CRYPTO_AUTHENCESN
  106  make olddefconfig
  107  grep CONFIG_CRYPTO_AUTHENCESN .config
  108  grep -R "authencesn\|AUTH" -n crypto/Kconfig crypto/Makefile
  109  grep -R "authenc" -n crypto/Kconfig crypto/Makefile
  110  grep CONFIG_CRYPTO_AUTHENC .config
  111  cd /workspaces/copy-fail-challenge-B
  112  make clean
  113  make setup
  114  make qemu
  115  find /workspaces/copy-fail-challenge-B/kernel/linux -name "authencesn.o"
  116  nm /workspaces/copy-fail-challenge-B/kernel/linux/crypto/authencesn.o | head
  117  grep -R "authencesn" -n /workspaces/copy-fail-challenge-B/kernel/linux/crypto
  118  make qemu
  119  grep -E "CONFIG_CRYPTO_AEAD|CONFIG_CRYPTO_USER_API|CONFIG_NET" /workspaces/copy-fail-challenge-B/kernel/linux/.config
  120  grep -E "CONFIG_CRYPTO_AEAD2|CONFIG_CRYPTO_MANAGER|CONFIG_CRYPTO_NULL" /workspaces/copy-fail-challenge-B/kernel/linux/.config
  121  nano /workspaces/copy-fail-challenge-B/scripts/03_build_rootfs.sh
  122  vi /workspaces/copy-fail-challenge-B/scripts/03_build_rootfs.sh
  123  cd /workspaces/copy-fail-challenge-B
  124  make clean
  125  make setup
  126  make qemu
  127  grep -n "DEBUG\|modprobe" /workspaces/copy-fail-challenge-B/scripts/03_build_rootfs.sh
  128  {   echo "=== HITO 1: KERNEL VULNERABLE CONFIRMADO ===";   echo "Fecha: $(date)";   echo "Hostname: $(hostname)";   echo "Kernel: $(uname -r)";   echo "Identt
  129  mkdir -p evidence
  130  vi evidence/hito1_vuln_confirmed.txt
  131  git add evidence/hito1_vuln_confirmed.txt
  132  git commit -m "hito-1: kernel vulnerable confirmado"
  133  history