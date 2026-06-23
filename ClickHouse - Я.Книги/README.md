# Анализ продуктовых метрик и контента книжного сервиса в ClickHouse

Проект посвящен глубокому анализу пользовательского поведения, проверке гипотез продуктового менеджмента и поиску аномалий в данных с использованием SQL и ClickHouse.

## 📌 Описание проекта
В рамках работы решается комплекс аналитических задач для сервиса цифровых книг (аналог Букмейт). Основной фокус направлен на:
*   Сегментацию мобильной аудитории (iOS vs Android).
*   Анализ потребления контента (аудиокниги vs текстовые версии).
*   Проверку эффективности системы обновлений приложений.
*   Data Quality — поиск аномалий в записи сессий и аудит разметки каталога.

## 🛠 Стек технологий
*   **СУБД:** ClickHouse (использование агрегатных функций-комбинаторов `sumIf`, `avgIf`, `uniqExactIf`).
*   **SQL Modern Features:** Оконные функции, работа с массивами (`groupUniqArray`, `hasAll`, `length`), работа со строками.
*   **Инструменты:** Jupyter Notebook (анализ и визуализация результатов).

## 🚀 Ключевые кейсы и результаты

### 1. Проверка продуктовых гипотез
*   **Гипотеза:** Пользователи iOS читают в 2 раза больше, чем слушают.
*   **Результат:** Гипотеза *опровергнута*. Хотя на iOS читателей больше на 22%, разрыв не достигает двукратного значения. Сегментация проводилась на основе PUID и определения "основной платформы" пользователя по времени сессий.

### 2. Анализ обновлений приложений (Mobile Analytics)
*   Рассчитана метрика **Update Rate** (средняя частота обновлений на пользователя). 
*   Выявлено, что пользователи Android обновляются чаще (3.19), чем на iOS (2.43).
*   Обнаружен "провал" в обновлении до актуальной версии на iOS (всего 1.92% пользователей на последней версии), что требует внимания команды разработки.

### 3. Поиск аномалий (Data Quality)
*   С помощью **коэффициента вариации** (отношение `stddevPop` к `avg`) выявлена критическая аномалия в записи длительности сессий в Латвии на платформе Android (CV = 7.77).
*   Проведен аудит тегов: проверено соответствие количества категорий внутренним гайдлайнам (не более 3-4 тегов на книгу).

### 4. Контент-аналитика
*   Составлен топ авторов и книг по суммарному времени потребления.
*   Выявлено интересное поведение: для бизнес-литературы (например, "Илон Маск") время прослушивания аудиоверсии в 2.5 раза превышает время чтения текста.

## 📊 Пример сложного запроса
В проекте реализованы многоуровневые CTE для сегментации пользователей по их предпочтениям:
```sql
with sessions as (
    select
        puid,
        usage_platform_ru platform,
        msk_business_dt_str dt,
        app_version
    from source_db.audition
    where usage_platform_ru in ('Букмейт iOS', 'Букмейт Android')
), ranked as (
    select
        puid,
        platform,
        dt,
        app_version,
        row_number() over (partition by puid, platform order by dt) rank,
        lagInFrame(app_version) over (partition by puid, platform order by dt) prev_version
    from sessions
), updates as (
    select
        puid,
        platform,
        countIf(app_version != prev_version) update_count
    from ranked
    where prev_version is not null
    group by puid, platform
), user_counts as (
    select
        platform,
        count(distinct puid) total_users
    from sessions
    group by platform
)
select
    u.platform,
    round(sum(u.update_count) / uc.total_users, 2) update_rate
from updates u
join user_counts uc on u.platform = uc.platform
group by u.platform, uc.total_users
order by u.platform;
```