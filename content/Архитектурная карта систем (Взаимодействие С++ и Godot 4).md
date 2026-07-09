Разделение обязанностей между C++ (GDExtension) и игровым движком Godot 4 для достижения максимального FPS.

---

```mermaid
graph TB
    subgraph Godot ["Слой Godot 4 (GDScript / Серверы)"]
        Player[Игрок и Камера]
        WorkerPool[WorkerThreadPool]
        RenderServer[RenderingServer]
        PhysicsServer[PhysicsServer3D]
    end

    subgraph Bridge ["GDExtension"]
        VoxelWorldNode["[[Класс-мост VoxelWorld (GDExtension)]]"]
    end

    subgraph Core ["C++ Core"]
        MemManager["[[Структура Менеджера Памяти (Slab или Bucket Allocator)]]"]
        Mesher["[[Оптимизация графики (Greedy Meshing)]]"]
        Loader["[[Жизненный цикл чанка (I-O & Стриминг)]]"]
    end

    Player --> VoxelWorldNode
    VoxelWorldNode --> WorkerPool
    WorkerPool --> Mesher
    Mesher --> RenderServer & PhysicsServer
    VoxelWorldNode --> MemManager & Loader
```

---
## Распределение задач
* **C++ (GDExtension)**: [[Структура Менеджера Памяти (Slab или Bucket Allocator)|Управление пулами памяти]], битовые сдвиги, [[Оптимизация графики (Greedy Meshing)|Генерация полигональных сеток (Greedy Meshing)]], расчет шумов.
* **Godot 4**: Вывод графики (`RenderingServer`), просчет коллизий (`PhysicsServer3D / Jolt`), менеджмент потоков (`WorkerThreadPool`), UI и логика игрока.
