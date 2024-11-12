# Тестовое задание BI-аналитик

Тестовое задание выполнено Насибовым Шахином

## Часть 1. 

### **Описание таблиц**

В данной базе данных представлены четыре таблицы:

#### 1. Вакансии (vacancy)

| Поле | Тип данных | Описание |
| --- | --- | --- |
| vacancy_id | int | Уникальный числовой идентификатор вакансии |
| name | string | Название вакансии |
| work_schedule | string | График работы |
| disabled | boolean | Переменная, которая сообщает нам о том, была ли вакансия удалена/заблокирована по каким-то причинам |
| area_id | int | Числовой идентификатор населенного пункта, в котором была размещена вакансия |
| creation_time | string | Дата создания вакансии |
| archived | boolean | Переменная, которая сообщает нам о том, является ли вакансия заархивированной |

#### 2. Резюме (resume)

| Поле | Тип данных | Описание |
| --- | --- | --- |
| resume_id | int | Уникальный числовой идентификатор резюме |
| disabled | boolean | Переменная, которая сообщает нам о том, удалено резюме или нет |
| is_finished | int | Переменная, обозначающая статус завершенности резюме (1 — завершено, 2 — проходит модерацию, 3 — не завершено) |
| area_id | int | Числовой идентификатор населенного пункта, который указан в резюме |
| compensation | bigint | Ожидаемая зарплата |
| currency | string | Валюта зарплаты в формате универсального кода из таблицы currency |
| position | string | Желаемая должность |
| birth_day | timestamp | Дата рождения |
| role_id_list | array | Список числовых идентификаторов профессиональных ролей, указаных в резюме |

#### 3. Населенные пункты (area)

| Поле | Тип данных | Описание |
| --- | --- | --- |
| area_id | int | Уникальный числовой идентификатор населенного пункта |
| area_name | string | Название населенного пункта |
| region_name | string | Название региона, где находится населенный пункт |
| country_name | string | Название страны |

#### 4. Валюты (currency)

| Поле | Тип данных | Описание |
| --- | --- | --- |
| code | string | Код валюты |
| rate | decimal | Коэффициент перевода валюты в рубли |







### **Написание SQL-запросов**

1. Выгрузить число созданных вакансий (в динамике по месяцам), опубликованных в России, в названии которых встречается слово «водитель», предлагающих «Гибкий график» работы, за 2020-2021 годы. Важно, чтобы вакансии на момент сбора данных не были удаленными / заблокированными.

```sql
SELECT 
    EXTRACT(YEAR FROM creation_time) AS год,
    EXTRACT(MONTH FROM creation_time) AS месяц,
    COUNT(*) AS количество_вакансий
FROM 
    vacancy
WHERE 
    disabled = FALSE
    AND archived = FALSE
    AND area_id IN (SELECT area_id FROM area WHERE country_name = 'Россия')
    AND name LIKE '%водитель%'
    AND work_schedule = 'Гибкий график'
    AND EXTRACT(YEAR FROM creation_time) BETWEEN 2020 AND 2021
GROUP BY 
    EXTRACT(YEAR FROM creation_time),
    EXTRACT(MONTH FROM creation_time)
ORDER BY 
    год,
    месяц;
```

2. Выяснить, в каких регионах РФ (85 штук) выше всего доля вакансий, предлагающих удаленную работу. Вакансии должны быть не заархивированными, не заблокированными и не удаленными, и быть созданными в 2021-2022 годах.

```sql
WITH 
    -- Подсчитываем количество вакансий, предлагающих удаленную работу, в каждом регионе
    remote_vacancies AS (
        SELECT 
            a.region_name, 
            COUNT(*) AS количество_вакансий
        FROM 
            vacancy v
        JOIN 
            area a ON v.area_id = a.area_id
        WHERE 
            v.disabled = FALSE
            AND v.archived = FALSE
            AND v.work_schedule = 'Удаленная работа'
            AND EXTRACT(YEAR FROM v.creation_time) BETWEEN 2021 AND 2022
            AND a.country_name = 'Россия'
        GROUP BY 
            a.region_name
    ),
    -- Подсчитываем общее количество вакансий в каждом регионе
    total_vacancies AS (
        SELECT 
            a.region_name, 
            COUNT(*) AS количество_всего
        FROM 
            vacancy v
        JOIN 
            area a ON v.area_id = a.area_id
        WHERE 
            v.disabled = FALSE
            AND v.archived = FALSE
            AND EXTRACT(YEAR FROM v.creation_time) BETWEEN 2021 AND 2022
            AND a.country_name = 'Россия'
        GROUP BY 
            a.region_name
    )
-- Вычисляем долю вакансий, предлагающих удаленную работу, в каждом регионе
SELECT 
    rv.region_name AS имя региона, 
    rv.количество_вакансий / tv.количество_всего AS доля_удаленных_вакансий
FROM 
    remote_vacancies rv
JOIN 
    total_vacancies tv ON rv.region_name = tv.region_name
ORDER BY 
    доля_удаленных_вакансий DESC;
```

3. Подсчитать «вилку» (10,25,50,75 и 90 процентиль) ожидаемых зарплат (в рублях) из московских и питерских резюме, имеющих роль «разработчик» (id роли — 91), по городам и возрастным группам (группы сделать как в примере таблицы ниже, не учитывать резюме без указания даты рождения — такие тоже есть). Возрастные группы считать на дату составления запроса. Резюме должно быть не удалено и иметь статус «завершено». Дополнительно выяснить (при помощи того же запроса) долю резюме по каждой группе, в которых указана ожидаемая зарплата. Пример таблицы, которая должна получиться на выходе:

| Город | Возрастная группа | Доля резюме с указанной зарплатой | 10 процентиль | 25 процентиль | 50 процентиль (медиана) | 75 процентиль | 90 процентиль |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Москва | 17 лет и младше |  |  |  |  |  |  |
| Москва | 18–24 |  |  |  |  |  |  |
| Москва | 25–34 |  |  |  |  |  |  |
| Москва | 35–44 |  |  |  |  |  |  |
| Москва | 45–54 |  |  |  |  |  |  |
| Москва | 55 и старше |  |  |  |  |  |  |
| … | … |  |  |  |  |  |  |

```sql
WITH 
    -- Подсчитываем возрастные группы
    age_groups AS (
        SELECT 
            CASE 
                WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_day)) < 18 THEN '17 лет и младше'
                WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_day)) BETWEEN 18 AND 24 THEN '18–24'
                WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_day)) BETWEEN 25 AND 34 THEN '25–34'
                WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_day)) BETWEEN 35 AND 44 THEN '35–44'
                WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_day)) BETWEEN 45 AND 54 THEN '45–54'
                ELSE '55 и старше'
            END AS возрастная_группа,
            r.area_id,
            r.compensation
        FROM 
            resume r
        WHERE 
            r.disabled = FALSE
            AND r.is_finished = 1
            AND birth_day IS NOT NULL
            AND 91 IN (r.role_id_list)
    ),
    -- Подсчитываем долю резюме с указанной зарплатой
    salary_proportion AS (
        SELECT 
            a.area_name AS город,
            ag.возрастная_группа,
            COUNT(CASE WHEN ag.compensation IS NOT NULL THEN 1 END) * 1.0 / COUNT(*) AS доля_резюме_с_указанной_зарплатой
        FROM 
            age_groups ag
        JOIN 
            area a ON ag.area_id = a.area_id
        GROUP BY 
            a.area_name,
            ag.возрастная_группа
    ),
    -- Подсчитываем процентили зарплат
    salary_percentiles AS (
        SELECT 
            a.area_name AS город,
            ag.возрастная_группа,
            PERCENTILE_CONT(0.1) WITHIN GROUP (ORDER BY ag.compensation) AS percentil_10,
            PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY ag.compensation) AS percentil_25,
            PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ag.compensation) AS percentil_50,
            PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY ag.compensation) AS percentil_75,
            PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY ag.compensation) AS percentil_90
        FROM 
            age_groups ag
        JOIN 
            area a ON ag.area_id = a.area_id
        WHERE 
            ag.compensation IS NOT NULL
        GROUP BY 
            a.area_name,
            ag.возрастная_группа
    )
-- Конечная таблтца
SELECT 
    sp.город,
    sp.возрастная_группа,
    sp.доля_резюме_с_указанной_зарплатой,
    sp2.percentil_10,
    sp2.percentil_25,
    sp2.percentil_50,
    sp2.percentil_75,
    sp2.percentil_90
FROM 
    salary_proportion sp
JOIN 
    salary_percentiles sp2 ON sp.город = sp2.город AND sp.возрастная_группа = sp2.возрастная_группа
WHERE 
    sp.город IN ('Москва', 'Санкт-Петербург')
ORDER BY 
    sp.город,
    sp.возрастная_группа;
```

## Часть 2. 

### **Визуализация данных и выводы**

Дашборд тестового задания от HeadHunter на позицию стажера BI-аналитикb доступна на Tableau Public по следующей ссылке: [Ссылка на дашборд](https://public.tableau.com/views/HeadHunter_17313836513010/sheet10?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)
