# Решение Оптимизации функции (UDF) возврата списка заказов

## Задача 1
_Проанализировать скрипт функции получения списка заказов и связанные с ней объекты, перечислить выявленные недочёты и потенциальные проблемы производительности._

---
Речь идет о функции _F_WORKS_LIST_, давайте сначала рассмотрим проблемы связанных с ней объектов.

### Стилистика именования
Сразу бросается в глаза различная стилистика именования столбцов, какие-то названы в стиле PaskalCase, другие в Snake_Case, и так далее. Хотя это замечание и не влияет на производительность, но является явным косметическим недочетом.

### F_EMPLOYEE_GET
- Нет обработки случая, когда пользователь не найден.
- Использует SYSTEM_USER, что может быть проблематично в некоторых конфигурациях.

### F_WORKITEMS_COUNT_BY_ID_WORK
- Использует NOT IN, что крайне неэффективно.
- Вызывается дважды для каждого заказа в основной функции.
- Не хватает индекса (id_work и is_complit).

### Преобразование D_DATE
- Столбец D_DATE в таблице @RESULT определен как VARCHAR(10), но CREATE_Date, откуда берутся значения, это DATETIME из таблицы Works.

### FULL_NAME
- Функция F_EMPLOYEE_FULLNAME добавляет большую нагрузку к запросу, ее использование крайне неэффективно. Не хватает индекса.

### LEFT OUTER JOIN
- Works.StatusId может быть NULL, хотя обработки такого случая здесь нет.

### WORKS.IS_DEL
- Для оптимизации таких запросов, следует добавить индекс по IS_DEL.

---
## Задача 2
_Предложить правки запросов без модификации структуры БД такие, что время выполнения запроса получения 3 000 заказов из 50 000 со средним количеством элементов в заказе равным 3 не будет превышать 1-2 сек. При выполнении задания допускается использовать LLM. В случае использования LLM должны быть приведены используемые промты на русском или английском языке._

---

### Заменим NOT IN на NOT EXIST в F_WORKITEMS_COUNT_BY_ID_WORK и сделаем ее таблично-значимой (iTV) функцией
```sql
DROP FUNCTION IF EXISTS dbo.F_WORKITEMS_COUNT_BY_ID_WORK;
GO

CREATE OR ALTER FUNCTION dbo.F_WORKITEMS_COUNT_BY_ID_WORK (
	@id_work INT,
	@is_complit BIT
)
RETURNS TABLE
AS
RETURN
(
	SELECT COUNT(*) AS Count
	FROM WorkItem wi
	WHERE wi.id_work = @id_work
	  AND wi.is_complit = @is_complit
	  AND NOT EXISTS (
		  SELECT 1 FROM Analiz a WHERE a.id_analiz = wi.id_analiz AND a.is_group = 1
	  )
);
GO
```

### Заменим вызов скалярной функции (исправленной выше), на OUTER APPLY в главной функции F_WORK_LIST
```sql
DROP FUNCTION IF EXISTS dbo.F_WORKS_LIST;
GO

CREATE OR ALTER FUNCTION dbo.F_WORKS_LIST()
RETURNS @RESULT TABLE
(
ID_WORK INT,
CREATE_Date DATETIME,
MaterialNumber DECIMAL(8,2),
IS_Complit BIT,
FIO VARCHAR(255),
D_DATE varchar(10),
WorkItemsNotComplit int,
WorkItemsComplit int,
FULL_NAME VARCHAR(101),
StatusId smallint,
StatusName VARCHAR(255),
Is_Print bit
)
AS

begin
insert into @result
SELECT
  Works.Id_Work,
  Works.CREATE_Date,
  Works.MaterialNumber,
  Works.IS_Complit,
  Works.FIO,
  convert(varchar(10), works.CREATE_Date, 104 ) as D_DATE,
  wi_not_complit.Count AS WorkItemsNotComplit,
  wi_complit.Count AS WorkItemsComplit,
  dbo.F_EMPLOYEE_FULLNAME(Works.Id_Employee) as EmployeeFullName,
  Works.StatusId,
  WorkStatus.StatusName,
  case
      when (Works.Print_Date is not null) or
      (Works.SendToClientDate is not null) or
      (works.SendToDoctorDate is not null) or
      (Works.SendToOrgDate is not null) or
      (Works.SendToFax is not null)
      then 1
      else 0
  end as Is_Print  
FROM
 Works
 left outer join WorkStatus on (Works.StatusId = WorkStatus.StatusID)
 OUTER APPLY dbo.F_WORKITEMS_COUNT_BY_ID_WORK(Works.Id_Work, 0) wi_not_complit
 OUTER APPLY dbo.F_WORKITEMS_COUNT_BY_ID_WORK(Works.Id_Work, 1) wi_complit
where
 WORKS.IS_DEL <> 1
 order by id_work desc -- works.MaterialNumber desc
return
end
GO
```

_Стало и правда лучше. вместо 6 секунд, `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` исполняется 2 секунды!_

---

## Задача 3

_Если для оптимизации требуется создание новых таблиц, столбцов, триггеров или хранимых процедур (функций), то необходимо описать возможные недостатки и отрицательные последствия от таких изменений._

---

Приведу крайне лаконичное решение данного пункта, хотя понимаю, что ожидается более развернутый ответ (Все дело в сроках, но если будет возможность порассуждать на эту тему лично, или вернуться к этому вопросу позже - буду рад!). Итак,  любая оптимизация, которая ускоряет запросы получения списка, например - создание индексов на поля IS_DEL, IS_GROUP и IS_COMPLIT, кэширование вычисляемых значений, или добавление триггеров на обновление агрегированныъх данных - все они добавляют расходы на запросы добавления/обновления данных, так как требуют дополнительных действий синхронизации данных. Другими слоавами - добавляют риски рассинхронизации.




