# Gestión de Almacenamiento Elástico con LVM en Linux
---

# 📘 Guía Técnica: Gestión de Almacenamiento Elástico con LVM en Linux

## 1. Introducción y Arquitectura
**LVM (Logical Volume Manager)** proporciona una capa de abstracción entre el sistema operativo y los discos físicos. Esto permite redimensionar volúmenes en caliente (sin reiniciar), agrupar múltiples discos en un solo espacio de almacenamiento y gestionar snapshots.

### Jerarquía de Abstracción LVM
Para entender el flujo, visualice la arquitectura en tres capas:

1.  **PV (Physical Volume / Volumen Físico):** La base hardware. Puede ser un disco duro completo (`/dev/sdb`), una partición (`/dev/sdb1`) o un dispositivo LUN de una SAN.
2.  **VG (Volume Group / Grupo de Volúmenes):** El "Pool" o bolsa de almacenamiento. Agrupa la capacidad de uno o varios **PVs** en una entidad administrativa unificada.
3.  **LV (Logical Volume / Volumen Lógico):** La partición virtual que consume el sistema operativo. Aquí es donde se crea el sistema de archivos (Filesystem).

---

## 2. Flujo de Implementación (Despliegue Inicial)

### Paso 1: Inicialización de la Capa Física (PV)
Preparamos el dispositivo de bloque para ser gestionado por LVM.
*Dispositivo de ejemplo:* `/dev/sdb`

```bash
# Inicializar el disco como volumen físico
sudo pvcreate /dev/sdb

# Verificación
sudo pvs
# Salida esperada: /dev/sdb   lvm2 a--  <size>  <free>
```

### Paso 2: Creación del Pool de Recursos (VG)
Creamos una bolsa de recursos llamada `vg_datos` utilizando el volumen físico preparado.

```bash
# Crear el Grupo de Volúmenes
sudo vgcreate vg_datos /dev/sdb

# Verificación
sudo vgs
# Observará que el VG tiene el tamaño total del disco físico.
```

### Paso 3: Despliegue del Volumen Lógico (LV)
Asignamos una porción del pool para crear el volumen donde residirán los datos.
*Ejemplo:* Crear un volumen de 2GB llamado `lv_proyecto`.

```bash
# Crear volumen lógico de 2GB
sudo lvcreate -L 2G -n lv_proyecto vg_datos

# Verificación
sudo lvs
```

### Paso 4: Formateo y Montaje
Aplicamos un sistema de archivos (EXT4 en este caso) y lo hacemos accesible al sistema.

```bash
# 1. Crear sistema de archivos (Filesystem)
sudo mkfs.ext4 /dev/vg_datos/lv_proyecto

# 2. Crear punto de montaje
sudo mkdir -p /mnt/proyectos

# 3. Montar el volumen
sudo mount /dev/vg_datos/lv_proyecto /mnt/proyectos

# Validación
df -h /mnt/proyectos
```

---

## 3. Configuración de Resiliencia (Persistencia)
El comando `mount` es temporal y se pierde al reiniciar. Para asegurar que el volumen se monte automáticamente al arrancar el sistema, debemos editar el archivo `/etc/fstab`.

1. Obtenga la ruta absoluta del mapper o el UUID (Recomendado):
2. Edite el archivo de configuración:
   ```bash
   sudo nano /etc/fstab
   ```
3. Añada la siguiente línea al final del archivo:

```plaintext
# <Dispositivo>                    <Punto Montaje>  <FS>   <Opciones>  <Dump> <Pass>
/dev/mapper/vg_datos-lv_proyecto   /mnt/proyectos   ext4   defaults     0      2
```

> **⚠️ Nota:** Verifique siempre la configuración con `sudo mount -a` antes de reiniciar para evitar errores de arranque.

---

## 4. Escenario de Escalabilidad (Expansión en Caliente)
Este procedimiento permite ampliar la capacidad de almacenamiento **sin desmontar el disco y sin tiempo de inactividad (Zero Downtime)**.

**Escenario:** El volumen `lv_proyecto` está lleno y necesitamos añadir 2GB extra.

### Fase A: Extensión Lógica (El Contenedor)
Primero ampliamos el límite del LVM utilizando espacio libre disponible en el `vg_datos`.

```bash
# Añadir 2GB al volumen lógico
sudo lvextend -L +2G /dev/vg_datos/lv_proyecto

# Opción alternativa: Usar todo el espacio libre restante del VG
# sudo lvextend -l +100%FREE /dev/vg_datos/lv_proyecto
```

### Fase B: Extensión del Filesystem (El Contenido)
El volumen lógico ha crecido, pero el sistema de archivos (EXT4) aún no "ve" el nuevo espacio. Debemos redimensionarlo en caliente.

```bash
# Redimensionar sistema de archivos EXT4
sudo resize2fs /dev/vg_datos/lv_proyecto
```
*(Nota: Si usas el sistema de archivos XFS, el comando sería `xfs_growfs /mnt/proyectos`).*

---

## 5. Auditoría y Comandos de Diagnóstico

Tabla de referencia rápida para la administración diaria.

| Comando | Nivel | Descripción Funcional |
| :--- | :--- | :--- |
| `lsblk` | Sistema | Muestra el árbol de dependencias de discos, particiones y volúmenes lógicos. |
| `pvs` | Físico | Lista los discos físicos inicializados y su asociación a VGs. |
| `vgs` | Grupo | Muestra el tamaño total del pool y el **espacio libre (VFree)** disponible para expandir. |
| `lvs` | Lógico | Muestra los volúmenes creados, su tamaño actual y el VG al que pertenecen. |
| `df -h` | Usuario | Muestra el espacio en disco disponible y usado desde la perspectiva del sistema operativo. |

---

### 💡 Buenas Prácticas
*   **Nomenclatura:** Use nombres descriptivos para los VGs y LVs (ej: `vg_mysql`, `lv_logs`, `lv_data`) para facilitar la administración.
*   **Monitoreo:** Vigile siempre la columna `VFree` en el comando `vgs`. Si el VG se llena, no podrá extender ningún volumen lógico hasta que añada un nuevo disco físico al grupo (`vgextend`).
*   **Backups:** Aunque LVM es seguro, redimensionar particiones conlleva riesgos inherentes. Siempre tenga copias de seguridad antes de operaciones críticas.

---

### 💡 Evidencia Prácticas
Figura 1: Configuración de hardware inicial en VirtualBox.

Figura 2: Reconocimiento del disco sdb de 10GB.

Figura 3: Creación de PV, VG y LV.

Figura 4: Formateo EXT4 y montaje en /mnt/proyectos.

Figura 5: Configuración de persistencia en fstab.

Figura 6: Estado inicial de 2.0GB antes de la expansión.

Figura 7: Resultado final: Volumen expandido a 3.9GB.
