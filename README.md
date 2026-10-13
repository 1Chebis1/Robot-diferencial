# Robot-diferencial

# Robot móvil diferencial con ROS 2 y Gazebo

Simulación de un robot móvil diferencial en **ROS 2 Jazzy** y **Gazebo Harmonic**, con LiDAR 2D, cámara RGB, odometría, árbol TF y un nodo propio de cinemática diferencial. El robot recorre un pasillo con una curva, un cilindro como obstáculo y un tramo final angosto.

<!-- Agrega aquí una captura o un GIF de la simulación -->
<!-- ![Simulación](docs/simulacion.png) -->

## Características

- Robot descrito una sola vez en **Xacro**, usado por Gazebo y por `robot_state_publisher`.
- Tracción diferencial con el plugin `DiffDrive` (odometría y TF `odom` → `base_footprint`).
- **LiDAR 2D** de 360 muestras, rango de 0.10 a 8.0 m, a 10 Hz (`/scan`).
- **Cámara RGB** de 640 x 480 a 20 Hz con frame óptico (`/camera/image_raw`, `/camera/camera_info`).
- **Nodo de cinemática diferencial** (`cinematica_diferencial`) que calcula las velocidades angulares teóricas de cada rueda a partir de `/cmd_vel`.
- Mundo de Gazebo con **pasillo**: tramo recto, curva de 90°, embudo y tramo angosto, más un obstáculo cilíndrico.
- Un único launch que levanta Gazebo, el robot, el bridge, RViz y el nodo propio.

## Requisitos

| Componente | Versión |
|---|---|
| Ubuntu | 24.04 |
| ROS 2 | Jazzy Jalisco |
| Gazebo | Harmonic |

Paquetes de ROS 2 usados (se instalan con `rosdep`): `ros_gz_sim`, `ros_gz_bridge`, `robot_state_publisher`, `joint_state_publisher_gui`, `rviz2`, `xacro`, `teleop_twist_keyboard`, `rqt_image_view`.

## Instalación

```bash
mkdir -p ~/robot_diferencial_ws/src
cd ~/robot_diferencial_ws/src
git clone https://github.com/<usuario>/<repositorio>.git robot_diferencial

cd ~/robot_diferencial_ws
source /opt/ros/jazzy/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

## Uso

### Simulación completa

```bash
ros2 launch robot_diferencial simulation.launch.py
```

Esto abre Gazebo con el mundo, crea el robot, conecta los topics con `ros_gz_bridge`, abre RViz con la configuración guardada e inicia el nodo de cinemática.

### Teleoperación

En otra terminal:

```bash
source /opt/ros/jazzy/setup.bash
source ~/robot_diferencial_ws/install/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

| Tecla | Acción |
|---|---|
| `i` / `,` | Avanzar / retroceder |
| `j` / `l` | Girar a la izquierda / derecha |
| `k` | Detener |

### Cámara

```bash
ros2 run rqt_image_view rqt_image_view
```

Seleccionar el topic `/camera/image_raw`.

### Solo visualizar el modelo

```bash
ros2 launch robot_diferencial display.launch.py
```

Abre RViz y `joint_state_publisher_gui` para revisar el modelo sin Gazebo.

## Estructura del paquete

```
robot_diferencial/
├── config/
├── launch/
│   ├── display.launch.py        # Visualización del modelo en RViz
│   └── simulation.launch.py     # Simulación completa
├── meshes/
├── resource/
├── robot_diferencial/
│   ├── __init__.py
│   └── cinematica_diferencial.py
├── rviz/
│   └── robot.rviz
├── urdf/
│   └── robot.urdf.xacro         # Modelo del robot, sensores y plugins
├── worlds/
│   └── mundo.sdf                # Pasillo con curva, embudo y obstáculo
├── package.xml
├── setup.cfg
└── setup.py
```

## Descripción del robot

| Parámetro | Valor |
|---|---|
| Chasis (largo x ancho x alto) | 0.45 x 0.32 x 0.12 m |
| Radio de rueda | 0.08 m |
| Separación entre ruedas | 0.36 m |
| Masa del chasis | 5.0 kg |
| Apoyo | Esfera (caster) de radio 0.035 m |

Árbol de enlaces:

```
base_footprint
└── base_link
    ├── left_wheel_link
    ├── right_wheel_link
    ├── caster_link
    ├── lidar_link
    └── camera_link
        └── camera_optical_link
```

El LiDAR está a z = 0.18 m para que el plano de escaneo no detecte la cámara frontal ni el propio chasis.

## Topics

| Topic | Tipo | Dirección |
|---|---|---|
| `/cmd_vel` | `geometry_msgs/msg/Twist` | ROS 2 → Gazebo |
| `/odom` | `nav_msgs/msg/Odometry` | Gazebo → ROS 2 |
| `/tf` | `tf2_msgs/msg/TFMessage` | Gazebo → ROS 2 |
| `/joint_states` | `sensor_msgs/msg/JointState` | Gazebo → ROS 2 |
| `/scan` | `sensor_msgs/msg/LaserScan` | Gazebo → ROS 2 |
| `/camera/image_raw` | `sensor_msgs/msg/Image` | Gazebo → ROS 2 |
| `/camera/camera_info` | `sensor_msgs/msg/CameraInfo` | Gazebo → ROS 2 |
| `/clock` | `rosgraph_msgs/msg/Clock` | Gazebo → ROS 2 |
| `/velocidad_rueda_izquierda` | `std_msgs/msg/Float64` | Nodo de cinemática |
| `/velocidad_rueda_derecha` | `std_msgs/msg/Float64` | Nodo de cinemática |
| `/velocidades_ruedas` | `std_msgs/msg/Float64MultiArray` | Nodo de cinemática (`[izquierda, derecha]`) |

## Cinemática diferencial

El plugin `DiffDrive` de Gazebo es el que mueve el robot. El nodo `cinematica_diferencial` recibe el mismo `/cmd_vel` y calcula las velocidades angulares teóricas de las ruedas, para comparar el modelo matemático con la simulación.

Con velocidad lineal `v`, velocidad angular `ω`, radio de rueda `r` y separación `L`:

```
ωR = (v + (L/2)·ω) / r
ωL = (v - (L/2)·ω) / r
```

Parámetros del nodo (se cambian desde el launch):

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| `radio_rueda` | 0.08 | Radio de rueda en metros |
| `separacion_ruedas` | 0.36 | Distancia entre ruedas en metros |

> Si cambias `wheel_separation` o `wheel_radius` en el Xacro, actualiza también estos parámetros en `simulation.launch.py`.

## Mundo de simulación

El archivo `worlds/mundo.sdf` define un pasillo de 1.2 m de ancho libre, con paredes de 0.6 m de alto:

1. **Tramo recto inicial:** de x = -1 a x = 6 m, con un cilindro (radio 0.12 m) en (3.0, 0.15) que obliga a esquivarlo por la derecha.
2. **Curva:** giro de 90° a la izquierda, con radio de eje de 2.0 m.
3. **Segundo tramo recto:** de y = 2 a y = 6 m.
4. **Embudo:** el ancho libre baja de 1.2 m a 0.8 m entre y = 6 y y = 8 m.
5. **Tramo angosto:** de y = 8 a y = 10 m, cerrado con una pared al fondo.

El robot nace en el origen, mirando hacia +X. Para modificar el mundo, edita `worlds/mundo.sdf` y vuelve a lanzar la simulación; con `--symlink-install` no hace falta recompilar.



