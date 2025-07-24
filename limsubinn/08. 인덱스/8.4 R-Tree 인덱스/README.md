## 8.4 R-Tree 인덱스
MySQL의 공간 인덱스는 R-Tree 인덱스 알고리즘을 이용해 2차원의 데이터를 인덱싱하고 검색하는 목적의 인덱스다. <br>
기본적인 내부 메커니즘은 B-Tree와 흡사하며, R-Tree 인덱스는 2차원의 공간 개념 값이다. <br>

MySQL의 공간 확장에는 세 가지 기능이 포함되어 있다.
- 공간 데이터를 저장할 수 있는 데이터 타입
- 공간 데이터의 검색을 위한 공간 인덱스(R-Tree 알고리즘)
- 공간 데이터의 연산 함수(거리 또는 포함 관계의 처리)

<br>

### 8.4.1 구조 및 특성
MySQL은 공간 정보의 저장 및 검색을 위해 여러 가지 기하학적 도형 정보를 관리할 수 있는 데이터 타입을 제공한다. <br>
<img width="500" alt="image" src="https://github.com/user-attachments/assets/028a7041-82cb-41c4-9d01-06a45a49d842"> <br>
마지막의 GEOMETRY 데이터 타입은 나머지 3개 타입의 슈퍼 타입으로, POINT, LINE, POLYGON 객체를 모두 저장할 수 있다.

<br>

MBR(Minimum Bounding Rectangle): 해당 도형을 감싸는 최소 크기의 사각형 <br>
이 사각형들의 포함 관계를 B-Tree 형태로 구현한 인덱스가 R-Tree 인덱스다. <br>
<img width="500" alt="image" src="https://github.com/user-attachments/assets/65ad46cb-16dd-4ccc-95ea-11585ab501a4"> 

<br>

e.g. <br>
![image](https://github.com/user-attachments/assets/293145c6-443a-40e9-aa9e-1e00607613fa) <br>
이 도형들의 MBR을 3개의 레벨로 나눠서 그려볼 수 있다. <br>
![image](https://github.com/user-attachments/assets/0f7df70e-9ce5-4b32-9158-2a89f56a0085) <br>

- 최상위 레벨: R1, R2 ➡️ R-Tree의 루트 노드에 저장되는 정보
- 차상위 레벨: R3, R4, R5, R6 ➡️ R-Tree의 브랜치 노드
- 최하위 레벨: R7 ~ R14 ➡️ 리프 노드

<img width="700" alt="image" src="https://github.com/user-attachments/assets/f2c31659-6384-4073-9f0a-48b458450b7b"> <br>

<br>

### 8.4.2 R-Tree 인덱스의 용도
일반적으로 WGS84(GPS) 기준의 위도, 경도 좌표 저장에 주로 사용된다. <br>
좌표 시스템에 기반을 둔 정보에 대해서는 모두 적용할 수 있다. <br>

R-Tree는 각 도형(MBR)의 포함 관계를 이용해 만들어진 인덱스다. <br>
따라서 `ST_Contains()` 또는 `ST_Within()` 등과 같은 포함 관계를 비교하는 함수로 검색을 수행하는 경우에만 인덱스를 이용할 수 있다. <br>
예를 들어, '현재 사용자의 위치로부터 반경 5km 이내의 음식점 검색' 등과 같은 검색에 사용할 수 있다. <br>

현재 MySQL의 `ST_Distance()`와 `ST_Distance_Sphere()` 함수는 공간 인덱스를 효율적으로 사용하지 못한다. <br>
따라서 공간 인덱스를 사용할 수 있는 `ST_Contains()`와 `ST_Within()`을 이용해 거리 기반의 검색을 해야 한다. <br>

![image](https://github.com/user-attachments/assets/45b6ba76-9af9-4d79-81e9-0bedcdfefbac) <br>
기준점 'P'로부터 반경 거리 5km 이내의 점(위치)들을 검색하려면 우선 사각 점선 상자에 포함되는 점들을 검색하면 된다. <br>
`ST_Contains()` 또는 `ST_Within()` 연산은 사각형 박스와 같은 다각형으로만 연산할 수 있으므로 <br>
반경 5km를 그리는 원을 포함하는 최소 사각형(MBR)로 포함 관계 비교를 수행한 것이다. <br>

```sql
# "사각 상자"에 포함된 좌표 Px 검색
mysql> SELECT * FROM tb_location WHERE ST_Contains(사각 상자, px);
mysql> SELECT * FROM tb_location WHERE ST_Within(px, 사각 상자);
```
- `ST_Contains(포함 경계를 가진 도형, 포함되는 도형 또는 점 좌표)`
- `ST_Within(포함되는 도형 또는 점 좌표, 포함 경계를 가진 도형)`

  
점 'P6'는 기준점 P로부터 반경 5km 이상 떨어져 있지만 최소 사각형 내에는 포함된다. <br>
이때 P6를 제거해야 한다면 `ST_Contains()` 비교의 결과에 대해 `ST_Distance_Sphere()` 함수를 이용해 다시 한번 필터링한다. <br>

```sql
# 반경 내에 포함된 좌표 Px 검색
mysql> SELECT * FROM tb_location
       WHERE ST_Contains(사각 상자, px) AND ST_Distance_Sphere(p, px) <= 5 * 1000;
```
