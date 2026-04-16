# Actividad 3.3 — Mapa 2D con Fotogrametría (Simulación Vista de Dron)

Simulación de la vista de un dron sobrevolando una zona plana. Se tomaron fotografías cenitales del **Mapa del Campus del Tecnológico de Monterrey** desde distintos ángulos y se reconstruyeron como un único **mapa ortomosaico 2D** usando técnicas de fotogrametría con Python y OpenCV.

---

## Resultado Final

![Mapa 2D Ortomosaico](map_tec_result.jpg)

> Mapa final ensamblado a partir de **11 imágenes superpuestas** — tamaño de salida: **3472 × 3530 px**

---

## Pipeline

```
Fotos (vista cenital simulando dron)
  └─▶ [1] Calibración de Cámara  — método de Zhang (tablero de ajedrez)
        └─▶ [2] Carga y Redimensión  — límite de 2000 px (gestión de memoria)
              └─▶ [3] Undistorsión  — corrección de distorsión de lente con K + dist
                    └─▶ [4] Detección de Características  — keypoints SIFT + descriptores 128-D
                          └─▶ [5] Matching de Características  — FLANN KNN + test de ratio de Lowe
                                └─▶ [6] Estimación de Homografía  — RANSAC (≥4 inliers)
                                      └─▶ [7] Warp + Stitching  — warp en perspectiva + mezcla
                                            └─▶ Mapa 2D Ortomosaico
```

---

## Visualizaciones

### Keypoints SIFT

| Mapa acumulado | Imagen entrante |
|:---:|:---:|
| ![Keypoints 1](keypoints_img1.jpg) | ![Keypoints 2](keypoints_img2.jpg) |

### Coincidencias de Características (primer par)

![Feature Matches](feature_matches.jpg)

> FLANN + test de ratio de Lowe (umbral 0.75). Las líneas conectan los keypoints coincidentes entre el mapa acumulado y la siguiente imagen.

---

### Progresión del Ensamblado

| Paso 1 | Paso 5 | Paso 10 (final) |
|:---:|:---:|:---:|
| ![Paso 1](map_step_1.jpg) | ![Paso 5](map_step_5.jpg) | ![Paso 10](map_step_10.jpg) |

---

## Resultados de ejecución (dataset map_tec)

| Paso | Imágenes ensambladas | Tamaño del canvas | SIFT kps (nueva) | Buenos matches | Inliers RANSAC |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | 1 → 2 | 1467 × 2215 | 16 915 | 3 336 | 2 701 |
| 2 | 2 → 3 | 2148 × 2215 | 13 951 | 2 463 | 1 808 |
| 3 | 3 → 4 | 2817 × 2215 | 12 997 | 1 135 | 445 |
| 4 | 4 → 5 | 2992 × 2793 | 10 607 | 1 570 | 742 |
| 5 | 5 → 6 | 2992 × 3530 | 9 871 | 1 716 | 850 |
| 6 | 6 → 7 | 2992 × 3530 | 6 583 | 2 783 | 2 240 |
| 7 | 7 → 8 | 2992 × 3530 | 9 633 | 2 914 | 2 365 |
| 8 | 8 → 9 | 3036 × 3530 | 14 740 | 2 947 | 2 300 |
| 9 | 9 → 10 | 3161 × 3530 | 12 488 | 3 115 | 2 553 |
| 10 | 10 → 11 | **3472 × 3530** | 17 233 | 2 064 | 1 368 |

---

## Archivos

| Archivo | Descripción |
|---|---|
| `repo/calibrate_camera.py` | Calibración de cámara con tablero de ajedrez (método de Zhang) |
| `repo/drone_map_pipeline.py` | Pipeline completo: undistort → SIFT → FLANN → RANSAC → stitch |
| `camera_params.npz` | Matriz intrínseca K + coeficientes de distorsión |
| `map_tec_result.jpg` | Mapa 2D ortomosaico final (campus TEC) |
| `feature_matches.jpg` | Visualización de matches FLANN para el primer par de imágenes |
| `keypoints_img1.jpg` | Keypoints SIFT sobre el mapa acumulado |
| `keypoints_img2.jpg` | Keypoints SIFT sobre la imagen entrante |
| `map_step_1.jpg … map_step_10.jpg` | Canvas intermedio después de cada paso de stitching |
| `undistorted_1.jpg … undistorted_11.jpg` | Imágenes corregidas de distorsión |
| `calib_images/` | Fotos del tablero de ajedrez para calibración |
| `map_tec/` | Imágenes originales del dataset |

---

## Uso

### 1. Instalar dependencias

```bash
pip install opencv-python numpy pillow pillow-heif
```

### 2. Calibrar la cámara

Coloca fotos del tablero en la carpeta `calib_images/` y ejecuta:

```bash
python3 repo/calibrate_camera.py
```

Produce `camera_params.npz` con la matriz intrínseca K y los coeficientes de distorsión. Apunta a un error de reproyección menor a 1 px.

### 3. Generar el mapa 2D

Con una carpeta de imágenes cenitales:

```bash
python3 repo/drone_map_pipeline.py --folder map_tec --params camera_params.npz
```

Con una lista explícita de imágenes:

```bash
python3 repo/drone_map_pipeline.py --imgs img1.jpg img2.jpg img3.jpg
```

Todas las opciones:

```
--folder      Carpeta con imágenes (HEIC / JPG / PNG)
--imgs        Lista explícita de rutas de imágenes
--params      camera_params.npz de calibrate_camera.py   (default: camera_params.npz)
--output      Nombre del archivo de salida               (default: map_2d.jpg)
--max-images  Limitar número de imágenes a ensamblar
--max-dim     Redimensionar de modo que la dimensión mayor ≤ N px  (default: 2000)
```

---

## Cómo funciona

### Calibración de cámara — método de Zhang
Se fotografía un tablero de ajedrez impreso desde múltiples ángulos. OpenCV detecta las esquinas interiores y, a partir de las posiciones 3D conocidas de esas esquinas versus sus posiciones 2D en imagen, resuelve la **matriz intrínseca K** (distancias focales `fx`, `fy` y centro óptico `cx`, `cy`) y los **coeficientes de distorsión** `[k1, k2, p1, p2, k3]`.

### Undistorsión
Cada foto se corrige usando K y los coeficientes de distorsión con `cv2.undistort`. Si la resolución de trabajo difiere de la de calibración, K se escala proporcionalmente.

### Detección de características — SIFT
SIFT (Scale-Invariant Feature Transform) encuentra keypoints distintivos en cada imagen y computa un descriptor de 128 dimensiones por keypoint. Los descriptores son invariantes a escala y rotación, lo que los hace robustos para emparejar fotos tomadas desde alturas o ángulos ligeramente distintos.

### Matching de características — FLANN + test de ratio de Lowe
FLANN (Fast Library for Approximate Nearest Neighbors) encuentra las dos coincidencias de descriptor más cercanas para cada keypoint. El **test de ratio de Lowe** conserva solo los matches inequívocos: un match se acepta solo si su distancia es menor a 0.75× la distancia al segundo mejor match.

### Estimación de homografía — RANSAC
Una homografía H (matriz proyectiva 3×3) mapea cada píxel de la nueva imagen al sistema de coordenadas del mapa acumulado. RANSAC ajusta H robustamente muestreando iterativamente subconjuntos de 4 puntos, computando un H candidato, contando inliers (error de reproyección < 5 px) y guardando el mejor resultado.

### Warp y stitching
La nueva imagen se deforma al sistema de coordenadas del mapa acumulado con `cv2.warpPerspective`. El canvas se expande automáticamente para contener ambas imágenes. En la región de solapamiento, las dos imágenes se mezclan al 50/50.

---

## Reflexión

Durante esta actividad encontramos varias dificultades técnicas. La primera fue la compatibilidad con archivos HEIC generados por iPhone, que OpenCV no puede leer nativamente y requirió integrar la librería `pillow-heif`. La segunda fue que las imágenes cenitales tienen muy poca variación de perspectiva entre fotogramas, lo que hace el trabajo de RANSAC más difícil cuando el solapamiento es insuficiente — una sola imagen fuera de orden rompe la cadena de stitching. El tercer desafío fue el tamaño del canvas: a medida que se ensamblan más imágenes el canvas crece y la memoria se convierte en un cuello de botella, haciendo necesario el parámetro `--max-dim`. Finalmente, la calibración de cámara con un número reducido de imágenes válidas del tablero produce parámetros intrínsecos menos fiables, lo que genera artefactos visibles en las costuras del mapa final.
