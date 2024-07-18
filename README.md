# 📒 RDB 동적으로 그리드 컬럼을 바꿀 수 있는 테이블 구조

[ag-grid](https://www.ag-grid.com/example-finance/) 와 같은 그리드 라이브러리를 사용하여 사용자가 컬럼을 추가하거나 삭제 기능이 필요 할 때 데이터베이스 테이블을 어떻게 구성할 수 있을까?

UI에서 보여지는 grid 컬럼과 RDB(관계형데이터베이스) 컬럼을 일치 시켜 개발하면 참 편하고 좋을 텐데...

하지만 RDB에서는 테이블 컬럼을 동적으로 자유롭게 변경 시킬 수 없다.&#x20;



따라서 컬럼 CRUD를 할 수 있는 유기적인 테이블 구조를 만들어야 한다.



자 시작해보자!



<mark style="color:yellow;">**"컬럼은 여러개의 데이터를 가진다"**</mark>



해당 기능의 요구사항을 봤을때 들었던 생각이다.&#x20;

아래 ERD 다이어그램을 보자

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

`CUSTOM_TABLE`: grid 컬럼의 정의 정보를 담는 테이블 (컬럼이름, 컬럼타입 등)

`CUSTOM_TABLE_VALUE`: 컬럼에 대한 데이터를 담는 테이블



`CUSTOM_TABLE 1 : CUSTOM_TABLE_VALUE N` 으로 1:N 관계를 가지게 구성하였다.

여기서 중요한건 `CUSTOM_TABLE_VALUE` 에 `row_order` 으로 행의 순서를 보장해준다.



각 테이블에 더미 데이터를 넣어보면



`CUSTOM_TABLE`

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

`CUSTOM_TABLE_VALUE`

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

각 컬럼에 대한 데이터가 수직으로 들어간 모습니다.



이제 이 2개의 테이블을 조인하여 `CUSTOM_TABLE`의 <mark style="color:red;">`column_name`</mark>으로 피봇팅 할 수 있는 쿼리를 작성해주면 된다.&#x20;

<figure><img src=".gitbook/assets/스크린샷 2024-07-18 오후 2.12.25.png" alt=""><figcaption></figcaption></figure>

이런 결과를 원할 것 이다.

쿼리를 보자

```sql
select * 
from (
	select 
	    ctv.row_order,
        -- < 동적쿼리 부분 CUSTOM_TABLE 컬럼 정보을 가져와서 동적으로 작성 가능~
	    	max(case when ct.column_name = '어이없엉' then ctv.value else null end) as 어이없엉,
		max(case when ct.column_name = '컬럼이름' then ctv.value else null end) as 컬럼이름,
		max(case when ct.column_name = '흥부' then ctv.value else null end) as 흥부,
		max(case when ct.column_name = '코드' then ctv.value else null end) as 코드,
		max(case when ct.column_name = '배고파' then ctv.value else null end) as 배고파,
		max(case when ct.column_name = '하이' then ctv.value else null end) as 하이,
		max(case when ct.column_name = '비고란' then ctv.value else null end) as 비고란,
		max(case when ct.column_name = '비비고' then ctv.value else null end) as 비고고
        -- >
	from "CUSTOM_TABLE" ct 
	left join "CUSTOM_TABLE_VALUE" ctv on ct.custom_table_id = ctv.custom_table_id 
	group by ctv.row_order 
	order by ctv.row_order 
) t
-- 검색하려면
-- where t.흥부 = '대전' 
-- 페이징 처리 
-- offset 10 
-- limit 100

```



`CUSTOM_TABLE`,  `CUSTOM_TABLE_VALUE` 두 테이블을 `left join` 으로 조인하고 `row_order` 로 `group` 하고 `case when` 절을 활용하여 해당 컬럼 이름과 맞지 않으면 null로 값을 채우고 max를 취하게 되면 된다.



`group by` 조건은 `row_order` 로 하지 않아도 된다. `ctv.custom_table_id` 로 하여도 같은 결과 일 것이다.

내가 작성한 예제는 개발 당시 행을 입력하는 순서만 보장 해주면 되어 `group by` 조건을 넣었다.



<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

&#x20;원하던 결과를 볼 수 있다.



쿼리 수행 결과는 아래와 같다



`CUSTOM_TABLE` 9 row, `CUSTOM_TABLE_VALUE` 9000 row 수행시 결과

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

나름 준수한 성능이다.

JPA를 사용한다면... querydsl을 사용하는 것이 좋겠다.
