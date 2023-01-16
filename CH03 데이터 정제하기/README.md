# CH03 데이터 정제하기
## 3-1. 불필요한 데이터 삭제하기
### 3-1-1. 열 삭제하기
1. loc 메서드 
  - 슬라이싱
  ```python
  ns_book = ns_df.loc[:. '번호':'등록일자'] # 번호 ~ 등록일자 열까지 전체 행을 선택
  ```
  - 불리언 배열
  ```python
  selected_columns = ns_df.columns != 'Unnamed: 13' # 삭제 대상 컬럼: Unnamed: 13
  ns_book = ns_df.loc[:, selected_columns] # selected_columns가 True인 모든 행을 선택
  ```
2. drop() 메서드
```python
ns_book = ns_df.drop('Unnamed: 13', axis=1)
ns_book = ns_df.drop(['Unnamed: 13', '부가기호'], axis=1)
ns_df.drop('Unnamed: 13', axis=1, inplace=True) # inplace=True 데이터프레임 바로 수정
```
3. dropna() 메서드
```python
ns_book = ns_df.dropna(axis=1) # dropna(): Nan이 하나 이상 포함된 행이나 열을 삭제
ns_book = ns_df.dropna(axis=1, how='all') # how='all' 모든 값이 NaN인 열을 삭제
```

### 3-1-2. 행 삭제하기
1. drop() 메서드
```python
ns_book = ns_book.drop([0,1]) # 인덱스 0부터 1까지 삭제
```
2. [] 연산자
  - 슬라이싱
  ```python
  ns_book2 = ns_book[2:]
  ns_book2 = ns_book[0:2]
  ```
  - 불리언 배열
  ```python
  selected_rows = ns_df['출판사'] == '한빛미디어'
  ns_book2 = ns_book[selected_rows]
  ns_book2 = ns_book.loc[selected_rows]
  ns_book2 = ns_book[ns_book['대출건수'] > 1000]
  ```

### 3-1-3. 중복된 행 찾기
```python
sum(ns_book.duplicated()) # 중복된 행 확인
sum(ns_book.duplicated(subset=['도서명','저자','ISBN'])) # subset을 기준으로 중복된 행을 찾음

dup_rows = ns_book.duplicated(subset=['도서명','저자','ISBN'], keep=False) # keep=False 모든 중복된 행을 True로 표시
ns_book3 = ns_book[dup_rows]
```

### 3-1-4. 그룹별로 모으기
```python
count_df = ns_book[['도서명','저자','ISBN','권','대출건수']]
group_df = count_df.groupby(by=['도서명','저자','ISBN','권'], dropna=False) # NaN이 있는 행을 삭제하지 않음
loan_count = count_df.groupby(by=['도서명','저자','ISBN','권'], dropna=False).sum() 
```

### 3-1-5. 원본 데이터 업데이트하기
1. 준비
```python
dup_rows = ns_book.duplicated(subset=['도서명','저자','ISBN','권'])
unique_rows = ~dup_rows # 중복된 행(True)를 False로 반전
ns_book3 = ns_book[unique_rows].copy() # 고유한 행만 선택
```
2. 원본 데이터프레임 인덱스 설정하기
```python
ns_book3.set_index(['도서명','저자','ISBN','권'], inplace=True)
ns_book3.head()
```
3. 업데이트하기: update() 메서드
```python
loan_count = count_df.groupby(by=['도서명','저자','ISBN','권'], dropna=False).sum()
ns_book3.update(loan_count) # 데이터프레임 합쳐짐
ns_book3.head()

ns_book4 = ns_book3.reset_index()

sum(ns_book['대출건수']>100) # 2311
sum(ns_book4['대출건수']>100) # 2550

ns_book4 = ns_book4[ns_book.columns] # 열 순서 맞추기
ns_book4.to_csv('ns_book4.csv', index=False)
```

### 3-1-6. 일괄 처리 함수 만들기
'''python
def data_cleaning(filename):
    """
    남산 도서관 장서 CSV 데이터 전처리 함수
    
    :param filename: CSV 파일이름
    """
    # 파일을 데이터프레임으로 읽습니다.
    ns_df = pd.read_csv(filename, low_memory=False)
    # NaN인 열을 삭제합니다.
    ns_book = ns_df.dropna(axis=1, how='all')

    # 대출건수를 합치기 위해 필요한 행만 추출하여 count_df 데이터프레임을 만듭니다.
    count_df = ns_book[['도서명','저자','ISBN','권','대출건수']]
    # 도서명, 저자, ISBN, 권을 기준으로 대출건수를 groupby합니다.
    loan_count = count_df.groupby(by=['도서명','저자','ISBN','권'], dropna=False).sum()
    # 원본 데이터프레임에서 중복된 행을 제외하고 고유한 행만 추출하여 복사합니다.
    dup_rows = ns_book.duplicated(subset=['도서명','저자','ISBN','권'])
    unique_rows = ~dup_rows
    ns_book3 = ns_book[unique_rows].copy()
    # 도서명, 저자, ISBN, 권을 인덱스로 설정합니다.
    ns_book3.set_index(['도서명','저자','ISBN','권'], inplace=True)
    # load_count에 저장된 누적 대출건수를 업데이트합니다.
    ns_book3.update(loan_count)
    
    # 인덱스를 재설정합니다.
    ns_book4 = ns_book3.reset_index()
    # 원본 데이터프레임의 열 순서로 변경합니다.
    ns_book4 = ns_book4[ns_book.columns]
    
    return ns_book4

new_ns_book4 = data_cleaning('ns_202104.csv')
'''

## 3-2. 잘못된 데이터 수정하기
### 3-2-1. 데이터프레임 정보 요약 확인하기
```python
ns_book4.info()
ns_book4.info(memory_usage='deep') # 메모리 사용량 확인
```
### 3-2-2. 누락된 값 처리하기
1. isna() 메서드 - 누락된 값 개수 확인하기
```python
ns_book4.isna().sum()
```
2. None과 np.nan - 누락된 값으로 표시하기
```python
ns_book4.loc[0, '도서권수'] = None
ns_book4['도서권수'].isna().sum()

ns_book4.loc[0, '도서권수'] = 1
ns_book4 = ns_book4.astype({'도서권수':'int32', '대출건수': 'int32'}) # astype() 메서드: 데이터 타입 지정

ns_book4.loc[0, '부가기호'] = None

import numpy as np
ns_book4.loc[0, '부가기호'] = np.nan # NaN 값으로 변경
```
3. loc, fillna() 메서드 - 누락된 값 바꾸기(1)
```python
set_isbn_na_rows = ns_book4['세트 ISBN'].isna() # 불리언 배열
ns_book4.loc[set_isbn_na_rows, '세트 ISBN'] = '' # 누락된 값을 ''로 교체

ns_book4['세트 ISBN'].isna().sum()

ns_book4['부가기호'].fillna('없음').isna().sum()
ns_book4.fillna({'부가기호':'없음'}).isna().sum() # 부가기호가 '없음'인 열만 선택
```
4. replace() 메서드 - 누락된 값 바꾸기(2)
  - 바꾸려는 값이 하나일 때
  ```python
  ns_book4.replace(np.nan, '없음').isna().sum()  # replace(원래 값, 새로운 값)
  ```
  - 바꾸려는 값이 여러 개일 때
  ```python
  ns_book4.replace([np.nan, '2021'], ['없음', '21']).head(2) # replace([원래 값1, 원래 값2],[새로운 값1, 새로운 값2)
  ns_book4.replace({np.nan: '없음', '2021': '21'}).head(2) 
  ```
  - 열 마다 다른 값으로 바꿀 때
  ```python
  ns_book4.replace({'부가기호': np.nan}, '없음').head(2) # replace({열 이름: 원래 값}, 새로운 값}
  ns_book4.replace({'부가기호': {np.nan: '없음'}, '발행년도': {'2021': '21'}}).head(2)
  ```
### 3-2-3. 정규 표현식
1. \d - 숫자 찾기
```python
ns_book4.replace({'발행년도': {r'\d\d(\d\d)': r'\1'}}, regex=True)[100:102] # () 그룹, \d\d\d\d 숫자 4개
ns_book4.replace({'발행년도': {r'\d{2}(\d{2})': r'\1'}}, regex=True)[100:102] # {} 반복 횟수
```
2. 마침표(.) - 문자 찾기
```python
ns_book4.replace({'저자': {r'(.*)\s\(지은이\)(.*)\s\(옮긴이\)': r'\1\2'}, 
                  '발행년도': {r'\d{2}(\d{2})': r'\1'}}, regex=True)[100:102]
```
### 3-2-4. 잘못된 값 바꾸기
```python
invalid_number = ns_book4['발행년도'].str.contains('\D', na=True)
print(invalid_number.sum())

ns_book5 = ns_book4.replace({'발행년도':r'.*(\d{4}).*'}, r'\1', regex=True)
ns_book5[invalid_number].head()

unkown_year = ns_book5['발행년도'].str.contains('\D', na=True)
print(unkown_year.sum())
ns_book5[unkown_year].head()

ns_book5.loc[unkown_year, '발행년도'] = '-1'
ns_book5 = ns_book5.astype({'발행년도': 'int32'})

ns_book5['발행년도'].gt(4000).sum()

dangun_yy_rows = ns_book5['발행년도'].gt(4000)
ns_book5.loc[dangun_yy_rows, '발행년도'] = ns_book5.loc[dangun_yy_rows, '발행년도'] - 2333

dangun_year = ns_book5['발행년도'].gt(4000)
print(dangun_year.sum())
ns_book5[dangun_year].head(2)

ns_book5.loc[dangun_year, '발행년도'] = -1

old_books = ns_book5['발행년도'].gt(0) & ns_book5['발행년도'].lt(1900)
ns_book5[old_books]

ns_book5.loc[old_books, '발행년도'] = -1
ns_book5['발행년도'].eq(-1).sum()
```
### 3-2-5. 누락된 정보 채우기
```python
na_rows = ns_book5['도서명'].isna() | ns_book5['저자'].isna() \
          | ns_book5['출판사'].isna() | ns_book5['발행년도'].eq(-1)
print(na_rows.sum())
# 크롤링 생략
```
### 3-2-6. 데이터를 이해하고 올바르게 정제하기
```python
def data_fixing(ns_book4):
    """
    잘못된 값을 수정하거나 NaN 값을 채우는 함수
    
    :param ns_book4: data_cleaning() 함수에서 전처리된 데이터프레임
    """
    # 도서권수와 대출건수를 int32로 바꿉니다.
    ns_book4 = ns_book4.astype({'도서권수':'int32', '대출건수': 'int32'})
    # NaN인 세트 ISBN을 빈문자열로 바꿉니다.
    set_isbn_na_rows = ns_book4['세트 ISBN'].isna()
    ns_book4.loc[set_isbn_na_rows, '세트 ISBN'] = ''
    
    # 발행년도 열에서 연도 네 자리를 추출하여 대체합니다. 나머지 발행년도는 -1로 바꿉니다.
    ns_book5 = ns_book4.replace({'발행년도':'.*(\d{4}).*'}, r'\1', regex=True)
    unkown_year = ns_book5['발행년도'].str.contains('\D', na=True)
    ns_book5.loc[unkown_year, '발행년도'] = '-1'
    
    # 발행년도를 int32로 바꿉니다.
    ns_book5 = ns_book5.astype({'발행년도': 'int32'})
    # 4000년 이상인 경우 2333년을 뺍니다.
    dangun_yy_rows = ns_book5['발행년도'].gt(4000)
    ns_book5.loc[dangun_yy_rows, '발행년도'] = ns_book5.loc[dangun_yy_rows, '발행년도'] - 2333
    # 여전히 4000년 이상인 경우 -1로 바꿉니다.
    dangun_year = ns_book5['발행년도'].gt(4000)
    ns_book5.loc[dangun_year, '발행년도'] = -1
    # 0~1900년 사이의 발행년도는 -1로 바꿉니다.
    old_books = ns_book5['발행년도'].gt(0) & ns_book5['발행년도'].lt(1900)
    ns_book5.loc[old_books, '발행년도'] = -1
    
    # 도서명, 저자, 출판사가 NaN이거나 발행년도가 -1인 행을 찾습니다.
    na_rows = ns_book5['도서명'].isna() | ns_book5['저자'].isna() \
              | ns_book5['출판사'].isna() | ns_book5['발행년도'].eq(-1)
    # 교보문고 도서 상세 페이지에서 누락된 정보를 채웁니다.
    updated_sample = ns_book5[na_rows].apply(get_book_info, 
        axis=1, result_type ='expand')
    
    # 도서명, 저자, 출판사가 NaN이거나 발행년도가 -1인 행을 삭제합니다.
    ns_book6 = ns_book5.dropna(subset=['도서명','저자','출판사'])
    ns_book6 = ns_book6[ns_book6['발행년도'] != -1]
    
    return ns_book6
```
