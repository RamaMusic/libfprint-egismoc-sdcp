# Egis `1c7a:05a1` en Ubuntu 22.04

Este procedimiento deja funcionando el lector de huellas **Egis / LighTuning `1c7a:05a1`** usando el fork `TenSeventy7/libfprint-egismoc-sdcp` con un parche adicional en SDCP host keys.

## Resumen

El lector:

- aparece en Linux como `1c7a:05a1`
- necesita soporte `egismoc` + SDCP
- con Ubuntu 22.04 no compila directo por la versión de OpenSSL del sistema
- además necesita un parche en `fpi-sdcp-device.c` para generar/exportar correctamente la public key EC del host

---

## 1. Verificar que el lector exista

```bash
lsusb
```

Debería aparecer algo así:

```text
1c7a:05a1 LighTuning Technology Inc. ETU905A80-E
```

---

## 2. Instalar dependencias de compilación

```bash
sudo apt update
sudo apt install -y \
  build-essential perl pkg-config meson ninja-build git cmake \
  libglib2.0-dev libgusb-dev libgirepository1.0-dev \
  libnss3-dev libpam0g-dev libpixman-1-dev libcairo2-dev \
  libusb-1.0-0-dev libudev-dev systemd-dev
```

---

## 3. Clonar el repo base

```bash
cd /tmp
git clone https://github.com/TenSeventy7/libfprint-egismoc-sdcp.git libfprint-sdcp
cd /tmp/libfprint-sdcp
```

---

## 4. Compilar OpenSSL 3.0.19 en una ruta privada

No tocar el OpenSSL del sistema. Compilar una copia aparte en `/opt`.

```bash
cd /tmp
curl -LO https://github.com/openssl/openssl/releases/download/openssl-3.0.19/openssl-3.0.19.tar.gz
tar xf openssl-3.0.19.tar.gz
cd openssl-3.0.19

./Configure --prefix=/opt/openssl-3.0.19 --libdir=lib linux-x86_64
make -j"$(nproc)"
sudo make install_sw
```

---

## 5. Exportar variables para usar ese OpenSSL

```bash
export PATH=/opt/openssl-3.0.19/bin:$PATH
export PKG_CONFIG_PATH=/opt/openssl-3.0.19/lib/pkgconfig:$PKG_CONFIG_PATH
export LD_LIBRARY_PATH=/opt/openssl-3.0.19/lib:$LD_LIBRARY_PATH
export CFLAGS="-I/opt/openssl-3.0.19/include $CFLAGS"
export LDFLAGS="-L/opt/openssl-3.0.19/lib $LDFLAGS"
```

Chequeo rápido:

```bash
openssl version
pkg-config --modversion openssl
```

Debería mostrar `3.0.19`.

---

## 6. Aplicar el parche SDCP

Volver al repo:

```bash
cd /tmp/libfprint-sdcp
```

Aplicar este parche automático:

```bash
python3 - <<'PY'
from pathlib import Path

p = Path("libfprint/fpi-sdcp-device.c")
s = p.read_text()

old_decl = """  guchar public_key[SDCP_PUBLIC_KEY_SIZE];
  gsize public_key_len = 0;
  guchar *random;
"""

new_decl = """  guchar public_key[SDCP_PUBLIC_KEY_SIZE];
  gsize public_key_len = 0;
  unsigned char *public_key_tmp = NULL;
  size_t public_key_tmp_len = 0;
  guchar *random;
"""

old_gen = """  if (private_key_bytes)
    key = sdcp_get_pkey (private_key_bytes, NULL);
  else
    key = EVP_EC_gen (SDCP_OPENSSL_CURVE_NAME);
"""

new_gen = """  if (private_key_bytes)
    key = sdcp_get_pkey (private_key_bytes, NULL);
  else
    {
      key = EVP_EC_gen (SDCP_OPENSSL_CURVE_NAME);
      if (!key)
        {
          g_error (\"Failed generating host key\");
          return FALSE;
        }
      if (!EVP_PKEY_set_utf8_string_param (key,
                                           OSSL_PKEY_PARAM_EC_POINT_CONVERSION_FORMAT,
                                           \"uncompressed\"))
        {
          g_error (\"Failed forcing uncompressed EC point format\");
          return FALSE;
        }
    }
"""

old_pub = """  /* get public_key from the key */
  if (!EVP_PKEY_get_octet_string_param (key, OSSL_PKEY_PARAM_PUB_KEY,
      public_key, SDCP_PUBLIC_KEY_SIZE, &public_key_len))
    {
      g_error (\"Failed getting public key\");
      return FALSE;
    }
  g_assert (public_key_len == SDCP_PUBLIC_KEY_SIZE);
"""

new_pub = """  /* get public_key from the key in guaranteed uncompressed format */
  public_key_tmp_len = EVP_PKEY_get1_encoded_public_key (key, &public_key_tmp);
  if (public_key_tmp_len == 0 || public_key_tmp == NULL)
    {
      g_error (\"Failed getting public key\");
      return FALSE;
    }
  g_assert (public_key_tmp_len == SDCP_PUBLIC_KEY_SIZE);
  memcpy (public_key, public_key_tmp, SDCP_PUBLIC_KEY_SIZE);
  public_key_len = public_key_tmp_len;
  OPENSSL_free (public_key_tmp);
"""

if old_decl not in s:
    raise SystemExit("No encontré el bloque de variables.")
if old_gen not in s:
    raise SystemExit("No encontré el bloque de generación de claves.")
if old_pub not in s:
    raise SystemExit("No encontré el bloque de public key.")

s = s.replace(old_decl, new_decl, 1)
s = s.replace(old_gen, new_gen, 1)
s = s.replace(old_pub, new_pub, 1)

p.write_text(s)
print("Patch aplicado correctamente.")
PY
```

---

## 7. Compilar `libfprint`

```bash
cd /tmp/libfprint-sdcp
rm -rf builddir

meson setup builddir --prefix=/usr/local -Ddoc=false -Dgtk-examples=false
ninja -C builddir
sudo env LD_LIBRARY_PATH=/opt/openssl-3.0.19/lib:$LD_LIBRARY_PATH ninja -C builddir install
sudo ldconfig
sudo systemctl restart fprintd.service
```

---

## 8. Probar el lector

```bash
fprintd-list $USER
```

Debería aparecer 1 dispositivo.

Después enrolar:

```bash
fprintd-enroll -f right-index-finger $USER
```

Y verificar:

```bash
fprintd-verify $USER
```

También conviene enrolar un dedo de backup:

```bash
fprintd-enroll -f left-index-finger $USER
```

---

## 9. Logs de debug, solo si algo falla

Activar debug:

```bash
sudo systemctl edit fprintd.service
```

Pegar:

```ini
[Service]
Environment=G_MESSAGES_DEBUG=all
```

Después:

```bash
sudo systemctl daemon-reload
sudo systemctl restart fprintd.service
sudo journalctl -fu fprintd.service
```

---

## 10. Sacar debug cuando ya funcione

```bash
sudo rm /etc/systemd/system/fprintd.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart fprintd.service
```

---

## 11. Guardar el parche en Git

Desde el repo:

```bash
cd /tmp/libfprint-sdcp
git checkout -b fix/egis-05a1
git add .
git commit -m "fix(egismoc): fix SDCP host key generation for 1c7a:05a1"
```

Para exportarlo como patch:

```bash
git format-patch -1 HEAD
```

Para subirlo a tu fork:

```bash
git remote rename origin upstream
git remote add origin https://github.com/RamaMusic/libfprint-egismoc-sdcp.git
git push -u origin fix/egis-05a1
```

---

## 12. Qué problema arregla este parche

El problema original era un crash en:

```text
fpi_sdcp_set_host_keys: assertion failed: (public_key_len == SDCP_PUBLIC_KEY_SIZE)
```

La solución fue:

- forzar formato EC `uncompressed`
- usar `EVP_PKEY_get1_encoded_public_key()` para obtener la public key del host en el formato correcto para SDCP

---

## 13. Notas

- Esto fue probado en **Ubuntu 22.04**
- Se usó **OpenSSL 3.0.19** privado en `/opt/openssl-3.0.19`
- No se reemplazó el OpenSSL del sistema
- El lector detectado fue **Egis / LighTuning `1c7a:05a1`**
