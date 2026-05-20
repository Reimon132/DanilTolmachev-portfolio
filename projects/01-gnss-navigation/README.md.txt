# ГНСС Навигационная система

Профессиональная навигационная система с приёмом данных GPS/ГЛОНАСС в реальном времени. Разработано для морских судовых устройств с внедрением в серийное производство.

Находится в активной разработке

> Исходный код не публикуется по причине NDA. Представлены только скриншоты интерфейса и архитектурное описание.

---

## Технологический стек

| Категория | Технологии |
|:---|:---|
| **Язык** | Python 3.10+ |
| **GUI Framework** | PyQt5 (QMainWindow, QWidget, QSvgWidget, pyqtgraph) |
| **Архитектура** | Signals/Slots, QThread, многопоточность (`threading`), модульное разделение окон |
| **Графика** | Динамическая SVG (`QSvgWidget` + `bytearray.format()`), `pyqtgraph` для карт/графиков |
| **Протоколы** | NMEA 0183 (RMC, VTG, GGA, GSV, GSA), Serial (`pyserial`) |
| **БД** | SQLite (CRUD-операции, кэширование настроек, сессионные данные) |
| **Платформы** | Windows / Linux (адаптация через `sys.platform`) |

---

## Ключевые особенности (реализовано в прототипе)

- ✅ **15+ окон интерфейса**: главное окно, компас, спутники, плоттер, маршруты, waypoints, настройки
- ✅ **Real-time обновление**: приём и парсинг NMEA-данных в отдельном потоке (threading.Thread с daemon=True)
- ✅ **Динамическая SVG-графика**: плавная анимация компаса (0-360°) через bytearray.format() и QSvgWidget
- ✅ **Кроссплатформенность**: единая кодовая база для Windows и Linux (sys.platform detection)
- ✅ **Сетевое взаимодействие**: Serial/Socket-каналы для связи с внешними устройствами (TransportRS класс)
- ✅ **Модульная архитектура**: разделение ответственности через Signals/Slots (sig_update_plotter)
- ✅ **Blur-эффекты**: QGraphicsBlurEffect для модальных окон и оверлеев
- ✅ **Жесты управления**: swipe-навигация между окнами (mousePressEvent/mouseReleaseEvent)

---

### Компоненты системы

**MainWindow (WindowMain)**
- Ядро приложения, управление всеми окнами
- Blur-эффекты и swipe-навигация
- Централизованная обработка NMEA-сообщений

**NMEA Parser (метод send_msg)**
- Парсинг предложений: RMC (позиция/курс/скорость), VTG (курс/скорость), GGA (спутники/HDOP), GSV (видимые спутники), GSA (PDOP/HDOP/VDOP)
- Преобразование координат: float → форматированная строка (56°20.1846' N)
- Валидация статуса (self.msgAll.status == "A" — активный фикс)

**SVG Rendering Engine**
- Динамическая генерация SVG через bytearray.format()
- Компас: Krygovaia_diagramma_new.format(svg_angle)
- Судно: Ship_New (статичный SVG)
- QSvgWidget.renderer().load() для мгновенного обновления

**Plotter Window (pyqtgraph)**
- Отрисовка карты и позиции судна
- Сигнал sig_update_plotter.emit(curr_lat, curr_lon, curr_course)
- Центрирование карты на текущей позиции

**Satellite Window**
- Визуализация до 12 спутников из GSV-сообщений
- Расчёт азимута и угла возвышения для каждого спутника
- SNR (signal-to-noise ratio) отображение

**Compass Window**
- Динамическая SVG-диаграмма (0-360°)
- Отображение истинного курса (True Course) и скорости
- Синхронизация с главным окном

---

## Обработка NMEA-протокола

### Поддерживаемые предложения

| Тип | Данные | Использование |
|:---|:---|:---|
| **RMC** | Position, Course, Speed, Date/Time | Основное обновление позиции |
| **VTG** | True/Magnetic Course, Speed (knots/kmph) | Дублирование курса/скорости |
| **GGA** | Num satellites, HDOP | Качество сигнала GPS |
| **GSV** | Satellite ID, SNR, Elevation, Azimuth | Визуализация спутников |
| **GSA** | PDOP, HDOP, VDOP, Satellite IDs | Точность позиционирования |

### Пример парсинга RMC
```python
# Извлечение чистых чисел для графики
curr_lat = float(self.msgAll.latitude)
curr_lon = float(self.msgAll.longitude)
curr_course = float(self.msgAll.true_course or 0)
curr_speed = float(self.msgAll.spd_over_grnd or 0)

# Форматирование для отображения
self.Longitude = self.msg_Longitude[:2] + "°" + self.msg_Longitude[2:] + "' " + self.msg_LongitudeNapr