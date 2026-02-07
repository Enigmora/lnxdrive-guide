# Packaging e Integracion Desktop

> **Ubicacion:** `08-Distribucion/05-packaging-desktop-integration.md`
> **Relacionado:** [Estructura de Repositorios](01-estructura-repositorios.md), [Comunicacion D-Bus](02-comunicacion-dbus.md), [Internacionalizacion](04-internacionalizacion.md)

---

## Parte XIX: Packaging e Integracion Desktop

> **Objetivo:** Documentar los archivos de integracion desktop, mecanismos de autostart, formatos de paquete y scripts post-install necesarios para distribuir LNXDrive como aplicacion nativa en cada entorno de escritorio Linux.

### 19.1 Filosofia: Integracion Nativa por Formato

Cada entorno de escritorio recibe paquetes nativos con archivos de integracion propios. Los repositorios fuente (`lnxdrive`, `lnxdrive-gnome`, etc.) definen los archivos de integracion y `lnxdrive-packaging` los ensambla en paquetes distribuibles.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FLUJO DE INTEGRACION DESKTOP                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Repositorios fuente              lnxdrive-packaging          Distribucion  │
│  ─────────────────                ──────────────────          ────────────  │
│                                                                             │
│  lnxdrive/                                                                  │
│  ├── data/                                                                  │
│  │   ├── lnxdrive.service    ──┐                                           │
│  │   └── dbus-service.service  ├──▶  rpm/              ──▶  .rpm           │
│  │                              │    deb/              ──▶  .deb           │
│  lnxdrive-gnome/               │    flatpak/           ──▶  .flatpak      │
│  ├── data/                     │    aur/               ──▶  PKGBUILD      │
│  │   ├── metadata.json    ─────┤                                           │
│  │   ├── .desktop         ─────┤                                           │
│  │   └── .gschema.xml     ─────┘                                           │
│                                                                             │
│  Cada repo define SUS archivos; packaging los ensambla                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 19.2 Archivos de Integracion por Componente

#### Daemon (`lnxdrive`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `lnxdrive.service` | `/usr/lib/systemd/user/` | Servicio systemd del daemon |
| `com.enigmora.LNXDrive.service` | `/usr/share/dbus-1/services/` | Activacion D-Bus |
| `com.enigmora.LNXDrive.xml` | `/usr/share/dbus-1/interfaces/` | Introspection XML |
| `org.enigmora.lnxdrive.policy` | `/usr/share/polkit-1/actions/` | Politicas PolicyKit |

#### GNOME Shell Extension (`lnxdrive-gnome`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `metadata.json` | `/usr/share/gnome-shell/extensions/lnxdrive-indicator@enigmora.com/` | Metadata de la extension |
| `extension.js` | (mismo directorio) | Punto de entrada de la extension |
| `dbus.js` | (mismo directorio) | Proxy D-Bus |
| `prefs.js` | (mismo directorio) | Panel de preferencias |
| `stylesheet.css` | (mismo directorio) | Estilos |

#### Nautilus Extension (`lnxdrive-gnome`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `liblnxdrive_nautilus.so` | `/usr/lib64/nautilus/extensions-4/` | Extension nativa Nautilus |

#### Preferencias GNOME (`lnxdrive-gnome`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `org.enigmora.LNXDrive.desktop` | `/usr/share/applications/` | Entrada en menu de aplicaciones |
| `org.enigmora.LNXDrive.metainfo.xml` | `/usr/share/metainfo/` | Metadata AppStream |
| `org.enigmora.LNXDrive.gschema.xml` | `/usr/share/glib-2.0/schemas/` | Esquema GSettings |
| `org.enigmora.LNXDrive.svg` | `/usr/share/icons/hicolor/scalable/apps/` | Icono escalable |
| `org.enigmora.LNXDrive-*.png` | `/usr/share/icons/hicolor/*/apps/` | Iconos rasterizados |

#### KDE Plasma (`lnxdrive-plasma`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `metadata.json` (plasmoid) | `/usr/share/plasma/plasmoids/org.enigmora.lnxdrive/` | Widget de Plasma |
| `main.qml` (plasmoid) | `contents/ui/` dentro del plasmoid | UI del widget |
| `lnxdrive-plasma.desktop` | `/usr/share/applications/` | Entrada en menu |
| `org.enigmora.lnxdrive-plasma.service` | `/usr/share/dbus-1/services/` | KDE service file |

#### GTK3 (`lnxdrive-gtk3`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `lnxdrive-gtk3.desktop` | `/usr/share/applications/` | Entrada en menu |
| `lnxdrive-gtk3-autostart.desktop` | `/etc/xdg/autostart/` | Autostart XDG |
| Iconos SVG/PNG | `/usr/share/icons/hicolor/*/apps/` | Iconos del tray |

#### Cosmic (`lnxdrive-cosmic`)

| Archivo | Ruta de Instalacion | Funcion |
|---------|---------------------|---------|
| `lnxdrive-cosmic.desktop` | `/usr/share/applications/` | Entrada en menu |
| `lnxdrive-cosmic.service` | `/usr/lib/systemd/user/` | Autostart via systemd |
| Iconos SVG/PNG | `/usr/share/icons/hicolor/*/apps/` | Iconos de la app |

### 19.3 Systemd User Services

El daemon se ejecuta como un servicio de usuario systemd. La activacion D-Bus permite arranque bajo demanda.

#### Servicio del Daemon

```ini
# /usr/lib/systemd/user/lnxdrive.service

[Unit]
Description=LNXDrive - Cloud Storage Daemon
Documentation=https://github.com/enigmora/lnxdrive
After=network-online.target
Wants=network-online.target

[Service]
Type=dbus
BusName=com.enigmora.LNXDrive
ExecStart=/usr/bin/lnxdrive-daemon
Restart=on-failure
RestartSec=5

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=%h/OneDrive %h/.config/lnxdrive %h/.local/share/lnxdrive
PrivateTmp=true

[Install]
WantedBy=default.target
```

#### Archivo de Activacion D-Bus

```ini
# /usr/share/dbus-1/services/com.enigmora.LNXDrive.service

[D-BUS Service]
Name=com.enigmora.LNXDrive
Exec=/usr/bin/lnxdrive-daemon
SystemdService=lnxdrive.service
```

La directiva `SystemdService=` delega el control de ciclo de vida a systemd, evitando que D-Bus lance instancias duplicadas.

#### Lifecycle

```bash
# Habilitar arranque automatico
systemctl --user enable lnxdrive.service

# Iniciar manualmente
systemctl --user start lnxdrive.service

# Ver estado
systemctl --user status lnxdrive.service

# Reiniciar (aplica nueva configuracion)
systemctl --user restart lnxdrive.service

# Detener
systemctl --user stop lnxdrive.service

# Ver logs
journalctl --user -u lnxdrive.service -f
```

### 19.4 Mecanismos de Autostart por Desktop

El mecanismo correcto de autostart depende del entorno de escritorio y su version. Esta es una fuente comun de errores al migrar entre versiones.

#### Matriz de Autostart

| Escritorio | Versiones | Mecanismo | Notas |
|------------|-----------|-----------|-------|
| **GNOME** | 45 - 48 | XDG autostart `.desktop` | `X-GNOME-Autostart-Phase` funciona |
| **GNOME** | 49+ | systemd user service | `X-GNOME-Autostart-Phase` ignorado; gnome-session delega a systemd |
| **KDE Plasma** | 6.x | XDG autostart `.desktop` o systemd | Ambos funcionan |
| **XFCE** | 4.x | XDG autostart `.desktop` | Mecanismo estandar |
| **Cinnamon** | 6.x | XDG autostart `.desktop` | Mecanismo estandar |
| **MATE** | 1.x | XDG autostart `.desktop` | Mecanismo estandar |
| **Cosmic** | 1.x | systemd user service | Escritorio basado en systemd |

#### XDG Autostart (GNOME <=48, KDE, XFCE, Cinnamon, MATE)

```ini
# /etc/xdg/autostart/lnxdrive-gtk3-autostart.desktop

[Desktop Entry]
Type=Application
Name=LNXDrive
Comment=Cloud Storage Client
Exec=/usr/bin/lnxdrive-gtk3 --background
Icon=lnxdrive
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Phase=Applications
NoDisplay=true
```

#### Systemd User Service (GNOME 49+, Cosmic)

```ini
# /usr/lib/systemd/user/lnxdrive-gnome-indicator.service

[Unit]
Description=LNXDrive GNOME Shell Indicator
After=gnome-shell-wayland.target
PartOf=gnome-session-manager@gnome.service

[Service]
Type=simple
ExecStart=/usr/bin/lnxdrive-gnome --indicator
Restart=on-failure
RestartSec=3

[Install]
WantedBy=gnome-session-manager@gnome.service
```

> **Nota critica:** En GNOME 49+ el campo `X-GNOME-Autostart-Phase` en archivos `.desktop` es ignorado porque gnome-session delega el control de arranque a systemd. Los paquetes para Fedora 43+ y GNOME 49+ deben usar exclusivamente el servicio systemd.

#### Flujo de Arranque

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FLUJO DE ARRANQUE POR DESKTOP                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GNOME 49+                                                                  │
│  ─────────                                                                  │
│  Login ──▶ gnome-session ──▶ systemd --user                               │
│                                  │                                         │
│                    ┌─────────────┴─────────────┐                           │
│                    │                           │                           │
│                    ▼                           ▼                           │
│          lnxdrive.service          lnxdrive-gnome-indicator.service        │
│          (daemon, via D-Bus        (indicador shell,                       │
│           activation)               WantedBy=gnome-session-manager)        │
│                                                                             │
│  GNOME <=48 / XFCE / Cinnamon / MATE                                      │
│  ─────────────────────────────────                                         │
│  Login ──▶ session-manager ──▶ XDG autostart                              │
│                                    │                                       │
│                                    ▼                                       │
│                         lnxdrive-gtk3 --background                         │
│                              │                                             │
│                              ▼                                             │
│                    Conecta via D-Bus a lnxdrive.service                    │
│                    (D-Bus activation inicia el daemon                      │
│                     si no esta corriendo)                                   │
│                                                                             │
│  KDE Plasma                                                                 │
│  ──────────                                                                │
│  Login ──▶ plasma-session ──▶ XDG autostart                               │
│                                    │                                       │
│                                    ▼                                       │
│                         lnxdrive-plasma                                    │
│                              │                                             │
│                              ▼                                             │
│                    D-Bus activation ──▶ lnxdrive.service                   │
│                                                                             │
│  Cosmic                                                                     │
│  ──────                                                                     │
│  Login ──▶ systemd --user ──▶ lnxdrive-cosmic.service                     │
│                                    │                                       │
│                                    ▼                                       │
│                    D-Bus activation ──▶ lnxdrive.service                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 19.5 Formatos de Paquete

#### Subpaquetes

La distribucion se organiza en subpaquetes independientes para que el usuario solo instale lo necesario para su escritorio:

| Subpaquete | Contenido | Dependencia |
|------------|-----------|-------------|
| `lnxdrive` | Daemon + CLI + FUSE | (base) |
| `lnxdrive-gnome` | Shell extension + Nautilus extension + Preferencias GTK4 | `lnxdrive` |
| `lnxdrive-gtk3` | Tray icon + Preferencias GTK3 | `lnxdrive` |
| `lnxdrive-plasma` | App Qt6 + Plasmoid | `lnxdrive` |
| `lnxdrive-cosmic` | App iced + Cosmic applet | `lnxdrive` |

#### RPM (Fedora, openSUSE)

```spec
# rpm/lnxdrive.spec

Name:           lnxdrive
Version:        1.0.0
Release:        1%{?dist}
Summary:        Cloud storage daemon for Linux
License:        GPL-3.0-or-later
URL:            https://github.com/enigmora/lnxdrive

BuildRequires:  rust-packaging
BuildRequires:  systemd-rpm-macros
BuildRequires:  pkgconfig(fuse3)
BuildRequires:  pkgconfig(dbus-1)

Requires:       fuse3
Requires:       dbus

%description
LNXDrive is a native cloud storage client for Linux desktops.
This package contains the daemon, CLI, and FUSE filesystem.

%install
install -Dm755 target/release/lnxdrive-daemon %{buildroot}%{_bindir}/lnxdrive-daemon
install -Dm755 target/release/lnxdrive-cli %{buildroot}%{_bindir}/lnxdrive
install -Dm644 data/lnxdrive.service %{buildroot}%{_userunitdir}/lnxdrive.service
install -Dm644 data/com.enigmora.LNXDrive.service \
    %{buildroot}%{_datadir}/dbus-1/services/com.enigmora.LNXDrive.service

%post
%systemd_user_post lnxdrive.service

%preun
%systemd_user_preun lnxdrive.service

%postun
%systemd_user_postun_with_restart lnxdrive.service

%files
%license LICENSE
%{_bindir}/lnxdrive-daemon
%{_bindir}/lnxdrive
%{_userunitdir}/lnxdrive.service
%{_datadir}/dbus-1/services/com.enigmora.LNXDrive.service
```

```spec
# rpm/lnxdrive-gnome.spec (extracto relevante)

Name:           lnxdrive-gnome
Requires:       lnxdrive >= %{version}
Requires:       gnome-shell >= 45

%post
glib-compile-schemas %{_datadir}/glib-2.0/schemas/ &>/dev/null || :
gtk-update-icon-cache -f %{_datadir}/icons/hicolor/ &>/dev/null || :
update-desktop-database %{_datadir}/applications/ &>/dev/null || :

%postun
glib-compile-schemas %{_datadir}/glib-2.0/schemas/ &>/dev/null || :
gtk-update-icon-cache -f %{_datadir}/icons/hicolor/ &>/dev/null || :
update-desktop-database %{_datadir}/applications/ &>/dev/null || :
```

#### DEB (Debian, Ubuntu)

```
deb/lnxdrive-core/debian/
├── control            # Dependencias y metadata
├── rules              # Build rules
├── lnxdrive.install   # Archivos a instalar
├── lnxdrive.service   # Symlink para dh_installsystemd
├── postinst           # Script post-instalacion
├── prerm              # Script pre-desinstalacion
└── copyright          # Licencia
```

```
# debian/control

Source: lnxdrive
Section: net
Priority: optional
Maintainer: Enigmora <maintainers@enigmora.com>
Build-Depends: debhelper-compat (= 13),
               cargo,
               libfuse3-dev,
               libdbus-1-dev

Package: lnxdrive
Architecture: any
Depends: ${shlibs:Depends}, ${misc:Depends}, fuse3, dbus
Description: Cloud storage daemon for Linux
 LNXDrive is a native cloud storage client for Linux desktops.
 This package contains the daemon, CLI, and FUSE filesystem.
```

```bash
# debian/postinst

#!/bin/sh
set -e

case "$1" in
    configure)
        # Informar al usuario sobre habilitacion del servicio
        echo "Para habilitar LNXDrive al inicio de sesion:"
        echo "  systemctl --user enable --now lnxdrive.service"
        ;;
esac

#DEBHELPER#
```

#### Flatpak

```yaml
# flatpak/org.enigmora.LNXDrive.yml

app-id: org.enigmora.LNXDrive
runtime: org.gnome.Platform
runtime-version: '47'
sdk: org.gnome.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.rust-stable

command: lnxdrive-gnome

finish-args:
  # Acceso al session bus para D-Bus
  - --socket=session-bus
  # Acceso al directorio de sincronizacion
  - --filesystem=home/OneDrive:create
  # Acceso a red
  - --share=network
  # Wayland + fallback X11
  - --socket=wayland
  - --socket=fallback-x11
  # Para FUSE
  - --device=all
  # Talk al daemon
  - --talk-name=com.enigmora.LNXDrive

modules:
  - name: lnxdrive-daemon
    buildsystem: simple
    build-commands:
      - cargo build --release -p lnxdrive-daemon -p lnxdrive-cli
      - install -Dm755 target/release/lnxdrive-daemon /app/bin/lnxdrive-daemon
      - install -Dm755 target/release/lnxdrive /app/bin/lnxdrive
    sources:
      - type: git
        url: https://github.com/enigmora/lnxdrive.git
        tag: v1.0.0

  - name: lnxdrive-gnome
    buildsystem: meson
    sources:
      - type: git
        url: https://github.com/enigmora/lnxdrive-gnome.git
        tag: v1.0.0
```

> **Nota:** Flatpak empaqueta daemon + UI juntos en un sandbox. El campo `--talk-name` permite comunicacion D-Bus entre el sandbox y procesos externos.

#### AUR (Arch Linux)

```bash
# aur/lnxdrive-core/PKGBUILD

pkgname=lnxdrive
pkgver=1.0.0
pkgrel=1
pkgdesc='Cloud storage daemon for Linux'
arch=('x86_64')
url='https://github.com/enigmora/lnxdrive'
license=('GPL-3.0-or-later')
depends=('fuse3' 'dbus')
makedepends=('rust' 'cargo')
source=("$pkgname-$pkgver.tar.gz::$url/archive/v$pkgver.tar.gz")
sha256sums=('SKIP')

build() {
    cd "$pkgname-$pkgver"
    cargo build --release -p lnxdrive-daemon -p lnxdrive-cli
}

package() {
    cd "$pkgname-$pkgver"
    install -Dm755 target/release/lnxdrive-daemon "$pkgdir/usr/bin/lnxdrive-daemon"
    install -Dm755 target/release/lnxdrive "$pkgdir/usr/bin/lnxdrive"
    install -Dm644 data/lnxdrive.service "$pkgdir/usr/lib/systemd/user/lnxdrive.service"
    install -Dm644 data/com.enigmora.LNXDrive.service \
        "$pkgdir/usr/share/dbus-1/services/com.enigmora.LNXDrive.service"
    install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
```

### 19.6 Scripts Post-Install

Los paquetes que instalan esquemas GSettings, iconos, archivos .desktop o servicios systemd necesitan ejecutar comandos post-instalacion para que los cambios surtan efecto.

| Comando | Cuando es Necesario | Paquetes |
|---------|---------------------|----------|
| `glib-compile-schemas /usr/share/glib-2.0/schemas/` | Al instalar `.gschema.xml` | lnxdrive-gnome |
| `gtk-update-icon-cache -f /usr/share/icons/hicolor/` | Al instalar iconos | Todos los UI |
| `update-desktop-database /usr/share/applications/` | Al instalar `.desktop` | Todos los UI |
| `systemctl --user daemon-reload` | Al instalar/actualizar `.service` | lnxdrive, lnxdrive-cosmic |

> **Nota:** `systemctl --user daemon-reload` no puede ejecutarse en scripts de paquete (corren como root). Los macros de RPM (`%systemd_user_post`) y los helpers de DEB (`dh_installsystemd`) manejan esto correctamente. El comando se documenta aqui como referencia para el usuario.

### 19.7 Matriz de Paquetes por Distribucion

| Distribucion | Formato | lnxdrive | lnxdrive-gnome | lnxdrive-gtk3 | lnxdrive-plasma | lnxdrive-cosmic |
|-------------|---------|----------|-----------------|----------------|------------------|-----------------|
| **Fedora** | RPM | Si | Si | Si | Si | - |
| **openSUSE** | RPM | Si | Si | Si | Si | - |
| **Debian** | DEB | Si | Si | Si | Si | - |
| **Ubuntu** | DEB | Si | Si | Si | Si | - |
| **Arch Linux** | AUR | Si | Si | Si | Si | Si |
| **Flathub** | Flatpak | Si (bundled) | Si | Si | - | - |
| **System76** | DEB | Si | - | - | - | Si |

Notas:
- **Fedora** incluye Cosmic a partir de F42, pero el paquete se agrega cuando el escritorio tenga adopcion suficiente
- **Flathub** empaqueta daemon + UI juntos; Plasma no tiene runtime Flatpak oficial de KDE estable aun
- **System76** prioriza Cosmic como escritorio nativo de Pop!\_OS

---

## Riesgos

### P1: Autostart Incompatible entre Versiones de GNOME

| Atributo | Valor |
|----------|-------|
| **Prioridad** | P1 (Alta) |
| **Componentes** | lnxdrive-gnome, lnxdrive-packaging |

**Descripcion:**
El cambio de XDG autostart a systemd user services en GNOME 49+ puede causar que la aplicacion no arranque automaticamente si el paquete no se adapta a la version de GNOME del usuario.

**Mitigacion:**
Los paquetes RPM/DEB deben detectar la version de GNOME y instalar el mecanismo de autostart correcto. En caso de duda, instalar ambos (el servicio systemd tiene prioridad si gnome-session lo soporta).

### P2: Permisos Flatpak Insuficientes

| Atributo | Valor |
|----------|-------|
| **Prioridad** | P2 (Media) |
| **Componentes** | lnxdrive (Flatpak) |

**Descripcion:**
El sandbox de Flatpak puede bloquear operaciones FUSE o acceso a directorios no declarados en `finish-args`.

**Mitigacion:**
Testear exhaustivamente el manifiesto Flatpak en cada release. Documentar claramente los permisos necesarios en la pagina de Flathub.

---

## Ver tambien

- [01-estructura-repositorios.md](01-estructura-repositorios.md) - Estructura de repositorios y `lnxdrive-packaging`
- [02-comunicacion-dbus.md](02-comunicacion-dbus.md) - Interfaces D-Bus y activacion
- [03-gobernanza-proyecto.md](03-gobernanza-proyecto.md) - Gobernanza y releases coordinados
- [04-internacionalizacion.md](04-internacionalizacion.md) - Archivos `.desktop` y metainfo traducidos
