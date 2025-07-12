## 8.5 전문 검색 인덱스

문서의 내용 전체를 인덱스화해서 특정 키워드가 포함된 문서를 검색하는 Full Text 검색에는 일반적인 용도의 B-Tree 인덱스를 사용할 수 없다. <br>
문서 전체에 대한 분석과 검색을 위한 인덱싱 알고리즘을 전문 검색(Full Text search) 인덱스라고 한다. <br>

<br>

### 8.5.1 인덱스 알고리즘
전문 검색에서는 문서 본문의 내용에서 사용자가 검색하게 될 키워드를 분석해 내고, 이러한 키워드로 인덱스를 구축한다. <br>
전문 검색 인덱스는 문서의 키워드를 인덱싱하는 기법에 따라 크게 단어의 어근 분석과 n-gram 분석 알고리즘으로 구분할 수 있다. <br>

#### 1️⃣ 어근 분석 알고리즘
1. 불용어(Stop Word) 처리
   - 검색에서 별 가치가 없는 단어를 모두 필터링해서 제거하는 작업
   - 불용어의 개수는 많지 않기 때문에 알고리즘을 구현한 코드에 모두 상수로 정의해서 사용하는 경우가 많다.
   - 유연성을 위해 불용어 자체를 데이터베이스화해서 사용자가 추가하거나 삭제할 수 있게 구현하는 경우도 있다.
2. 어근 분석(Stemming)
   - 검색어로 선정된 단어의 뿌리인 원형을 찾는 작업
   - 각 국가의 언어가 서로 문법이 다르고 다른 방식으로 발전해왔기 때문에 형태소 분석이나 어근 분석도 언어별로 방식이 모두 다르다.
   - 한글이나 일본어의 경우 영어와 같이 단어의 변형 자체는 거의 없기 때문에 어근 분석보다는 문장의 형태소를 분석해서 명사와 조사를 구분하는 기능이 더 중요하다.
   - MySQL 서버에서는 오픈소스 형태소 분석 라이브러리인 [MeCab](https://taku910.github.io/mecab/)(일본어...)을 플러그인 형태로 사용할 수 있게 지원한다.
     하지만 MeCab 프로그램만 설치한다고 되는 것은 아니고, 단어 사전이 필요하며, 문장을 해체해서 각 단어의 품사를 식별할 수 있는 문장의 구조 인식이 필요하다.
     문장의 구조 인식을 위해서는 실제 언어의 샘플을 이용해 언어를 학습하는 과정이 필요하다.

#### 2️⃣ n-gram 알고리즘
형태소 분석은 매우 전문적인 전문 검색 알고리즘이어서 많은 노력과 시간을 필요로 한다. <br>
이런 단점을 보완하기 위한 방법으로 도입된 것이 n-gram 알고리즘이다. <br>
단순히 키워드를 검색해내기 위한 인덱싱 알고리즘이라고 할 수 있다. <br>

n-gram이란 본문을 무조건 몇 글자씩 잘라서 인덱싱하는 방법이다. <br>
형태소 분석보다 단순하고 국가별 언어에 대한 이해와 준비 작업이 필요 없는 반면, 만들어진 인덱스의 크기는 상당히 큰 편이다. <br>
n은 인덱싱할 키워드의 최소 글자 수를 의미하는데, 일반적으로 2-gram 방식이 많이 사용된다. <br>

e.g. <br>
2-gram 알고리즘으로 `To be or not to be. That is the question` 문장의 토큰을 분리하는 방법을 살펴보자. <br>
<table><thead>
  <tr>
    <th>단어</th>
    <th colspan="7">bi-gram 토큰</th>
  </tr></thead>
<tbody>
  <tr>
    <td>To</td>
    <td>To</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>be</td>
    <td>be</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>or</td>
    <td>or</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>not</td>
    <td>no</td>
    <td>ot</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>to</td>
    <td>to</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>be</td>
    <td>be</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>That</td>
    <td>Th</td>
    <td>ha</td>
    <td>at</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>is</td>
    <td>is</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>the</td>
    <td>th</td>
    <td>he</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>question</td>
    <td>qu</td>
    <td>ue</td>
    <td>es</td>
    <td>st</td>
    <td>ti</td>
    <td>io</td>
    <td>on</td>
  </tr>
</tbody></table>

불용어 처리 -> 동일하거나 포함하는 경우 걸러서 버린다.
<details>
  <summary>`information_schema.innodb_ft_default_stopword` 테이블을 통해 확인 가능</summary>
  <img src="https://github.com/user-attachments/assets/bc50df02-7e1a-455f-aea0-9c6e93b9bd96">
</details>

<table><thead>
  <tr>
    <th>입력</th>
    <th>불용어 일치</th>
    <th>불용어 포함</th>
    <th>출력 (최종 인덱스 등록)</th>
  </tr></thead>
<tbody>
  <tr>
    <td>at</td>
    <td></td>
    <td>O</td>
    <td></td>
  </tr>
  <tr>
    <td>be</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>be</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>es</td>
    <td></td>
    <td></td>
    <td>es</td>
  </tr>
  <tr>
    <td>ha</td>
    <td></td>
    <td>O</td>
    <td></td>
  </tr>
  <tr>
    <td>he</td>
    <td></td>
    <td></td>
    <td>he</td>
  </tr>
  <tr>
    <td>io</td>
    <td></td>
    <td>O</td>
    <td></td>
  </tr>
  <tr>
    <td>is</td>
    <td></td>
    <td>O</td>
    <td></td>
  </tr>
  <tr>
    <td>no</td>
    <td></td>
    <td></td>
    <td>no</td>
  </tr>
  <tr>
    <td>on</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>or</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>ot</td>
    <td></td>
    <td></td>
    <td>ot</td>
  </tr>
  <tr>
    <td>qu</td>
    <td></td>
    <td></td>
    <td>qu</td>
  </tr>
  <tr>
    <td>st</td>
    <td></td>
    <td></td>
    <td>st</td>
  </tr>
  <tr>
    <td>Th</td>
    <td></td>
    <td></td>
    <td>Th</td>
  </tr>
  <tr>
    <td>th</td>
    <td></td>
    <td></td>
    <td>th</td>
  </tr>
  <tr>
    <td>ti</td>
    <td></td>
    <td>O</td>
    <td></td>
  </tr>
  <tr>
    <td>To</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>to</td>
    <td>O</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>ue</td>
    <td></td>
    <td></td>
    <td>ue</td>
  </tr>
</tbody></table>

MySQL 서버는 이렇게 구분된 토큰을 단순한 B-Tree 인덱스에 저장한다.

#### 3️⃣ 불용어 변경 및 삭제
앞서 살펴본 예시를 보면 "ti", "at", "ha" 와 같은 토큰들이 모두 걸러져서 버려졌다. <br>
이처럼 불용어 처리는 사용자를 더 혼란스럽게 만드는 기능일 수도 있다. <br>
따라서 불용어 자체를 완전히 무시하거나 사용자가 직접 불용어를 등록하는 방법을 권장한다. <br>

**✅ 불용어 처리 무시** <br>
1. MySQL 서버의 모든 전문 검색 인덱스에 대해 불용어를 완전히 제거하는 것
   - 설정 파일(`my.cnf`)의 `ft_stopword_file` 시스템 변수에 빈 문자열(`''`) 설정 -> MySQL 서버 재시작
2. InnoDB 스토리지 엔진을 사용하는 테이블의 전문 검색 인덱스에 대해서만 불용어 처리 무시
   - `innodb_ft_enable_stopword` 시스템 변수를 `OFF`로 설정

**✅ 사용자 정의 불용어 사용**
1. 불용어 목록을 파일로 저장하여 등록
   - 설정 파일(`my.cnf`)의 `ft_stopword_file` 시스템 변수에 파일명 등록
2. 불용어의 목록을 테이블로 저장 (InnoDB 스토리지 엔진을 사용하는 테이블만 가능)
   - 불용어 테이블을 생성하여 `innodb_ft_server_stopword_table` 시스템 변수에 불용어 테이블 설정
   - 여러 전문 검색 인덱스가 서로 다른 불용어를 사용해야 하는 경우 이용하면 된다.
   - ```sql
     # 불용어 테이블 생성
     mysql> CREATE TABLE my_stopword(value VARCHAR(30)) ENGINE = INNODB;
     mysql> INSERT INTO my_stopword(value) VALUES ('MySQL');
     
     # 불용어 테이블 설정
     mysql> SET GLBOAL innodb_ft_server_stopword_table = 'mydb/my_stopword';
     mysql> ALTER TABLE tb_bi_gram
             ADD FULLTEXT INDEX fx_title_body(title, body) WITH PARSER ngram;
     ```
   - ❗️주의❗️
     - 전역 설정이기 때문에 순차적으로 적용하고 인덱스를 만들어야 한다.
     - 변경 이후에 전문 검색 인덱스가 생성되어야만 변경된 불용어가 적용된다.

#### *️⃣ 추가 - 불용어 처리를 도대체 왜 하는가?
- "the", "is", "and", "a", "in", "of" 같은 자주 등장하지만 의미 없는 단어
- 이런 단어들을 인덱싱하면 인덱스 크기만 커지고 검색 성능은 떨어진다.
- e.g. <br>
  사용자가 "a book in the library"를 검색했을 때 핵심은 "book", "library" <br>
  -> "a", "in", "the"와 같은 단어는 제외하는 것이 더 정확한 검색 결과를 제공
<br>

### 8.5.2 전문 검색 인덱스의 가용성
전문 검색 인덱스를 사용하려면 반드시 다음 두 가지 조건을 갖춰야 한다.
- 쿼리 문장이 전문 검색을 위한 문법(MATCH ... AGAINST ...)을 사용
- 테이블이 전문 검색 대상 컬럼에 대해서 전문 인덱스 보유

<br>

e.g. <br>
테이블의 컬럼에 전문 검색 인덱스 생성
```sql
mysql> CREATE TABLE tb_test (
          doc_id INT,
          doc_body TEXT,
          PRIMARY KEY (doc_id),
          FULLTEXT KEY fx_docbody (doc_body) WITH PARSER ngram
        ) ENGINE = InnoDB;
```

다음과 같은 쿼리로도 원하는 검색 결과를 얻을 수 있지만, 효율적으로 쿼리가 실행된 것이 아닌 풀 테이블 스캔으로 처리한다.
```sql
mysql> SELECT * FROM tb_test WHERE doc_body LIKE '%애플%';
```

다음은 전문 검색 인덱스를 사용한 쿼리이다.
```sql
mysql> SELECT * FROM tb_test
       WHERE MATCH(doc_body) AGAINST('애플' IN BOOLEAN MODE);
```

- `IN BOOLEAN MODE`는 논리 연산자를 사용해서 더 정밀한 검색 가능 <br>

  | 기호  | 의미              | 예시                    |
  | :--- | :--------------- | :--------------------- |
  | `+` | 반드시 포함 (AND)    | `+애플 +아이폰` → 둘 다 포함   |
  | `-` | 반드시 제외 (NOT)    | `애플 -삼성` → 삼성 제외      |
  | `"` | 정확히 일치 (phrase) | `"아이폰 15"` → 문구 전체 일치 |
  | `*` | 와일드카드(뒤 확장)     | `아이*` → 아이폰, 아이패드 등   |
  | `~` | 관련성 낮은 것 포함 가능  | `~애플` → 낮은 점수 포함      |

<br>

#### ✏️ 참고
| 모드                           | 특징                          |
| :---------------------------- | :--------------------------- |
| `NATURAL LANGUAGE MODE` (기본) | 점수 기반, 불용어 무시됨, 연산자 사용 못 함  |
| `BOOLEAN MODE`               | 점수 무시, 연산자 사용 가능, 불용어 포함 가능 |
| `WITH QUERY EXPANSION`       | 초기 검색 결과 기반으로 확장 검색 수행      |


     
