Алгоритм мутации данных чанка при изменении вокселя игроком или физикой.

---

При вызове `voxel_world.set_block(x, y, z, block_id)` со стороны GDScript происходит следующая цепочка:

1. **Проверка**: 
	Если чанк однородный и новый `block_id` совпадает со старым → мгновенный выход.
2. **Апгрейд (если нужен)**: 
	Если чанк однородный или количество бит не вмещает новый тип → запрашивается слот в более тяжелом пуле памяти. Старые биты перепаковываются в новый слот. Старый слот возвращается в свой `FreeList`.
3. **Запись**: 
	Выполняется сдвиг `(mixed.packed_voxels[array_idx] >> bit_offset) & base_mask` для обновления бит (или сброс маской старых бит и наложение новых через `|`).
4. **События**: 
	Flag `is_dirty = true`, flag `needs_mesh_rebuild = true`.
5. **Потоки**: 
	Задача на пересчет геометрии уходит в `WorkerThreadPool`. После готовности меша в главном потоке обновляются `RenderingServer` и `PhysicsServer3D`.



```mermaid
flowchart TD
    Start([Вызов set_voxel от игрока]) --> CheckHomogeneous{Чанк однородный?}
    
    %% Ветка однородного чанка
    CheckHomogeneous -- Да --> MatchID{Новый ID == Старый ID?}
    MatchID -- Да --> Exit([Ничего не делаем / Выход])
    MatchID -- Нет --> UpgradeFromHomogeneous[Разворачиваем чанк в Mixed <br> Выделяем слот в пуле 1-бит или 2-бит]
    UpgradeFromHomogeneous --> WriteBits
    
    %% Ветка смешанного чанка
    CheckHomogeneous -- Нет --> CheckPalette{Новый тип есть в палитре?}
    CheckPalette -- Да --> WriteBits
    CheckPalette -- Нет --> CheckPaletteFull{Палитра заполнена <br> для текущих бит?}
    
    CheckPaletteFull -- Нет --> AddToPalette[Добавляем тип в палитру] --> WriteBits
    CheckPaletteFull -- Да --> UpgradeBits[Апгрейд разрядности: <br> 1. Запрос слота в пуле тяжелее <br> 2. Перепаковка старых бит в новые <br> 3. Возврат старого слота в FreeList]
    UpgradeBits --> AddToPalette

    %% Финальные операции
    WriteBits[Запись новых бит сдвигами] --> SetFlags[Выставляем флаги: <br> is_dirty = true <br> needs_mesh_rebuild = true]
    SetFlags --> PushToWorker[Отправка задачи мешинга <br> в WorkerThreadPool Godot]
    PushToWorker --> AsyncMesh[В ФОНЕ: Расчет Greedy Meshing]
    AsyncMesh --> UpdateServers[В ГЛАВНОМ ПОТОКЕ: <br> Обновление RenderingServer <br> Обновление PhysicsServer3D]
    UpdateServers --> End([Готово])

```

---
## Реализация сдвигов
Используется логика сдвига 64-битного слова вправо на `bit_offset` с последующим наложением маленькой константной маски (например, `0x03` для 2 бит), что эффективнее динамических масок.

---
*Связанные заметки:* [[Принципы управления памятью (Chunk Memory)]], [[Структура Менеджера Памяти (Slab или Bucket Allocator)]], [[Класс-мост VoxelWorld (GDExtension)]].
