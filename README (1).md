# Самостійна робота 1. ROS 2: workspace, ООП-nodes та ESP32 через micro-ROS

**Студент:** Artem Bilotil
**Група:** ← ВСТАВТЕ СВОЮ ГРУПУ
**Дата виконання:** ← ВСТАВТЕ ДАТУ

**Workspace:** `ros2_stud_bilotil_ws`
**Пакет C++:** `stud_bilotil_cpp_pkg`
**Пакет Python:** `stud_bilotil_py_pkg`
**Основний topic (текстові повідомлення):** `/stud_bilotil/message`

Цей звіт покриває Частину A і Частину B роботи (Етапи 1–7). Апаратна частина (ESP32,
micro-ROS, Temperature Monitor) буде додана окремо після виконання Етапів 8–12.

---

## Схема системи (поточний стан, Етапи 1–7)

```text
                 ros2_stud_bilotil_ws
                          │
         ┌────────────────┴────────────────┐
         │                                  │
  stud_bilotil_cpp_pkg              stud_bilotil_py_pkg
         │                                  │
 stud_bilotil_publisher      stud_bilotil_subscriber_1/2/3
         │                                  │
         └────────► /stud_bilotil/message ◄─┘
```

---

## 1. Правило іменування

Усі об'єкти в роботі названі за прізвищем студента (`bilotil`):

| Об'єкт | Ім'я |
|---|---|
| Workspace | `ros2_stud_bilotil_ws` |
| C++ package | `stud_bilotil_cpp_pkg` |
| Python package | `stud_bilotil_py_pkg` |
| Node — publisher | `stud_bilotil_publisher` |
| Node — subscriber | `stud_bilotil_subscriber` |
| Node — монітор температури | `stud_bilotil_temperature_monitor` |
| Node на мікроконтролері | `stud_bilotil_esp32` |
| Topic — текстові повідомлення | `/stud_bilotil/message` |
| Topic — температура, °C | `/stud_bilotil/temperature/celsius` |
| Topic — температура, °F | `/stud_bilotil/temperature/fahrenheit` |

---

## 2. Стиль коду

Обидва nodes реалізовано класами:

- `StudentPublisher` успадковує `rclcpp::Node` (C++), усі ROS-сутності (`publisher_`,
  `timer_`, `counter_`) — приватні поля класу, `main()` містить лише
  `rclcpp::init()` → `std::make_shared<StudentPublisher>()` → `rclcpp::spin()` →
  `rclcpp::shutdown()`.
- `StudentSubscriber` успадковує `rclpy.node.Node` (Python), конструктор викликає
  `super().__init__(...)`, `listener_callback` — метод класу.

---

## 3. Етап 1. Власний workspace

### Структура каталогів

| Каталог | Опис |
|---|---|
| `src` | Містить початковий вихідний код (source code) усіх створених або завантажених пакетів ROS 2. |
| `build` | Використовується системою збірки `colcon` для зберігання проміжних файлів компіляції та конфігурації. |
| `install` | Містить готові зібрані виконувані файли, бібліотеки, скрипти та `setup.bash` для їх підключення до середовища. |
| `log` | Зберігає текстові файли журналів (логів) із детальним звітом про процес виконання команд `colcon build`. |

### Вивід перевірки

```text
$ printenv | grep ROS
ROS_VERSION=2
ROS_PYTHON_VERSION=3
ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
ROS_DISTRO=jazzy

$ echo $AMENT_PREFIX_PATH
/home/ros/ros2_stud_bilotil_ws/install/my_test_pkg:/opt/ros/jazzy
```

<!-- ВСТАВТЕ СЮДИ СВІЙ ВЛАСНИЙ ВИВІД printenv | grep ROS та echo $AMENT_PREFIX_PATH,
     знятий у НОВОМУ терміналі (без ручного джерела source), щоб показати, що
     сорсинг у ~/.bashrc працює автоматично -->

### Чому `install/setup.bash` підключається після `/opt/ros/jazzy/setup.bash`, а не замість нього

Файл `install/setup.bash` локального workspace підключається після системного
`/opt/ros/jazzy/setup.bash`, оскільки ROS 2 працює за принципом каскадного накладання
середовищ (underlay / overlay). Системний ROS 2 надає базовий фреймворк, стандартні
бібліотеки та типи повідомлень, а локальний workspace будується поверх нього і
додає власні пакети. Послідовне підключення дописує шляхи локального workspace на
початок змінних оточення (наприклад, `AMENT_PREFIX_PATH`), при цьому зберігаючи
доступ до всіх системних ресурсів ROS 2.

---

## 4. Етап 2. Два packages

Створено два пакети у `~/ros2_stud_bilotil_ws/src/`:

| Package | Build type | Залежності |
|---|---|---|
| `stud_bilotil_cpp_pkg` | `ament_cmake` | `rclcpp`, `std_msgs` |
| `stud_bilotil_py_pkg` | `ament_python` | `rclpy`, `std_msgs` |

### Структура C++ package

```text
stud_bilotil_cpp_pkg/
├── include/
│   └── stud_bilotil_cpp_pkg/
│       └── student_publisher.hpp
├── src/
│   ├── student_publisher.cpp
│   └── main.cpp
├── CMakeLists.txt
└── package.xml
```

### Структура Python package

```text
stud_bilotil_py_pkg/
├── stud_bilotil_py_pkg/
│   ├── __init__.py
│   ├── student_subscriber.py
│   └── temperature_monitor.py
├── resource/
├── setup.py
├── setup.cfg
└── package.xml
```

### Перевірка (наявність обох пакетів)

```text
$ source ~/ros2_stud_bilotil_ws/install/setup.bash
$ ros2 pkg list | grep bilotil
stud_bilotil_cpp_pkg
stud_bilotil_py_pkg
```

<!-- Якщо у вас є власний, більш свіжий вивід ros2 pkg list | grep bilotil —
     вставте його сюди замість наведеного вище -->

---

## 5. Етап 3. C++ Publisher

**Node:** `stud_bilotil_publisher`

| Параметр | Значення |
|---|---|
| Topic | `/stud_bilotil/message` |
| Тип повідомлення | `std_msgs/msg/String` |
| Частота | 2 Hz (таймер 500 мс) |
| Формат повідомлення | `Bilotil | Message #N` |

### Код `student_publisher.hpp`

```cpp
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ
     ~/ros2_stud_bilotil_ws/src/stud_bilotil_cpp_pkg/include/stud_bilotil_cpp_pkg/student_publisher.hpp -->
```

### Код `student_publisher.cpp`

```cpp
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ
     ~/ros2_stud_bilotil_ws/src/stud_bilotil_cpp_pkg/src/student_publisher.cpp -->
```

### Код `main.cpp`

```cpp
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ
     ~/ros2_stud_bilotil_ws/src/stud_bilotil_cpp_pkg/src/main.cpp -->
```

### Код `CMakeLists.txt`

```cmake
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ CMakeLists.txt пакета stud_bilotil_cpp_pkg -->
```

### Перевірка запуску

```text
$ cd ~/ros2_stud_bilotil_ws
$ colcon build --packages-select stud_bilotil_cpp_pkg
$ source install/setup.bash
$ ros2 run stud_bilotil_cpp_pkg stud_bilotil_publisher
[INFO] [timestamp] [stud_bilotil_publisher]: Publishing: 'Bilotil | Message #0'
[INFO] [timestamp] [stud_bilotil_publisher]: Publishing: 'Bilotil | Message #1'
[INFO] [timestamp] [stud_bilotil_publisher]: Publishing: 'Bilotil | Message #2'
```

<!-- ВСТАВТЕ СЮДИ СКРІНШОТ ТЕРМІНАЛУ З ЛОГАМИ PUBLISHER-А
     (наприклад: ![Publisher log](docs/publisher_log.png)) -->

---

## 6. Етап 4. Python Subscriber

**Node:** `stud_bilotil_subscriber`, підписаний на `/stud_bilotil/message`.

### Код `student_subscriber.py`

```python
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ
     ~/ros2_stud_bilotil_ws/src/stud_bilotil_py_pkg/stud_bilotil_py_pkg/student_subscriber.py -->
```

### Код `setup.py`

```python
<!-- ВСТАВТЕ СЮДИ ПОВНИЙ ВМІСТ ФАЙЛУ setup.py пакета stud_bilotil_py_pkg -->
```

### Перевірка (Publisher + Subscriber одночасно, два різні термінали)

**Термінал 1 (Publisher):**
```text
$ source ~/ros2_stud_bilotil_ws/install/setup.bash
$ ros2 run stud_bilotil_cpp_pkg stud_bilotil_publisher
```

**Термінал 2 (Subscriber):**
```text
$ source ~/ros2_stud_bilotil_ws/install/setup.bash
$ ros2 run stud_bilotil_py_pkg stud_bilotil_subscriber
[INFO] [timestamp] [stud_bilotil_subscriber]: I heard: "Bilotil | Message #0"
[INFO] [timestamp] [stud_bilotil_subscriber]: I heard: "Bilotil | Message #1"
[INFO] [timestamp] [stud_bilotil_subscriber]: I heard: "Bilotil | Message #2"
```

<!-- ВСТАВТЕ СЮДИ СКРІНШОТ ДВОХ ТЕРМІНАЛІВ ПОРУЧ (publisher зліва, subscriber справа),
     де видно, що номери повідомлень збігаються -->

---

## 7. Етап 5. Дослідження системи через ROS 2 CLI

Виконано (у Терміналі 3, коли publisher і subscriber вже працювали) сім
діагностичних команд ROS 2 CLI.

**1. Список активних nodes** — `ros2 node list`
```text
/stud_bilotil_publisher
/stud_bilotil_subscriber
```

**2. Список активних topics** — `ros2 topic list -t`
```text
/parameter_events [rcl_interfaces/msg/ParameterEvent]
/rosout [rcl_interfaces/msg/Log]
/stud_bilotil/message [std_msgs/msg/String]
```

**3. Publishers/subscribers конкретного node** — `ros2 node info /stud_bilotil_publisher`
```text
/stud_bilotil_publisher
  Subscribers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
  Publishers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /rosout: rcl_interfaces/msg/Log
    /stud_bilotil/message: std_msgs/msg/String
  Service Servers:
    /stud_bilotil_publisher/describe_parameters: rcl_interfaces/srv/DescribeParameters
    /stud_bilotil_publisher/get_parameter_types: rcl_interfaces/srv/GetParameterTypes
    /stud_bilotil_publisher/get_parameters: rcl_interfaces/srv/GetParameters
    /stud_bilotil_publisher/get_type_description: type_description_interfaces/srv/GetTypeDescription
    /stud_bilotil_publisher/list_parameters: rcl_interfaces/srv/ListParameters
    /stud_bilotil_publisher/set_parameters: rcl_interfaces/srv/SetParameters
    /stud_bilotil_publisher/set_parameters_atomically: rcl_interfaces/srv/SetParametersAtomically
  Service Clients:

  Action Servers:

  Action Clients:
```

**4. Тип і підключення topic** — `ros2 topic info /stud_bilotil/message`
```text
Type: std_msgs/msg/String
Publisher count: 1
Subscription count: 1
```

**5. Структура типу повідомлення** — `ros2 interface show std_msgs/msg/String`
```text
# This was originally provided as an example message.
# It is deprecated as of Foxy
# It is recommended to create your own semantically meaningful message.
# However if you would like to continue using this please use the equivalent in example_msgs.

string data
```

**6. Читання повідомлень з topic** — `ros2 topic echo /stud_bilotil/message`
```text
data: 'Bilotil | Message #763'
---
data: 'Bilotil | Message #764'
---
data: 'Bilotil | Message #765'
---
data: 'Bilotil | Message #766'
---
data: 'Bilotil | Message #767'
---
```
*(зупинено вручну через Ctrl+C)*

**7. Фактична частота публікації** — `ros2 topic hz /stud_bilotil/message`
```text
average rate: 1.992
	min: 0.499s max: 0.509s std dev: 0.00418s window: 4
average rate: 2.001
	min: 0.492s max: 0.509s std dev: 0.00560s window: 9
average rate: 2.000
	min: 0.492s max: 0.509s std dev: 0.00492s window: 12
average rate: 2.000
	min: 0.479s max: 0.522s std dev: 0.00828s window: 32
```
*(зупинено вручну через Ctrl+C)*

**Висновок:** фактична середня частота публікації (~2.000 Hz) точно відповідає
заданій у коді C++ частоті таймера (період 500 мс = 2 Hz). Значення `min`/`max`
коливаються в межах кількох мілісекунд, що є нормальним джерелом джитера в
непристосованому до real-time середовищі ОС.

<!-- Якщо у вас частота суттєво відрізняється від 2 Hz — за вимогою методички
     потрібно вказати, у чому була причина і як ви її усунули -->

---

## Місця, які ще потрібно заповнити власними даними

| № | Розділ | Що вставити |
|---|---|---|
| 1 | Етап 1 | Власний вивід `printenv | grep ROS` та `echo $AMENT_PREFIX_PATH` у новому терміналі |
| 2 | Етап 3 | Повний код `student_publisher.hpp`, `student_publisher.cpp`, `main.cpp`, `CMakeLists.txt` |
| 3 | Етап 3 | Скріншот терміналу з логами publisher-а |
| 4 | Етап 4 | Повний код `student_subscriber.py`, `setup.py` |
| 5 | Етап 4 | Скріншот двох терміналів (publisher + subscriber) поруч |
| 6 | Титул | Група та дата виконання |

> Етапи 6 (три Subscribers) та 7 (rqt_graph) методички у вашому листуванні вже
> позначені як пункти 6 і 7 CLI-переліку — якщо потрібен окремий звіт і по них
> (три subscribers через remapping, rqt_graph зі скріншотами, фізичний
> експеримент із графом), скажіть — доповню цей файл цими розділами теж.
